---
title:  "Playing with Go: Using SQLite as storage and Redis as protocol"
date:   2022-06-07 08:00:00 +0100
categories: go sqlite
---

# Playing with Go: Using SQLite as storage and Redis as protocol

## Introduction

I wanted to play with [Go](https://go.dev/) since I think it has a pretty impressive ratio speed of development / app performance. I decided to try one of the classics, build a Redis clone. Redis is open source and pretty well documented, so I gave it a try.

Using the Redis protocol for communication allows you to use any of the Redis clients to validate your implementation.

## Implementation

First, a warning, this is a toy project, is not feature complete at all and it is not worth using it for any other thing than learning.

I copied the boring details of how to interact with Redis protocol from [miniredis](https://github.com/alicebob/miniredis).

The main loop of the program.

```go
const (
	HOST = "0.0.0.0"
	PORT = "6379"
	TYPE = "tcp"
)

func main() {
	listener, err := net.Listen("tcp", fmt.Sprintf("%s:%s", HOST, PORT))
	if err != nil {
		log.Panicln(err)
	}
	log.Printf("Listening on %s:%s\n", HOST, PORT)

	defer listener.Close()
	err = database.Init(database.SQLITE, "database.db")
	if err != nil {
		log.Panicln(err)
	}

	for {
		conn, err := listener.Accept()
		connection := protocol.Connection{Conn: conn}

		if err != nil {
			log.Panicln(err)
		}

		go handleRequest(connection)
	}
}
```

The program starts by listening on a socket, initializing the database and then it enters in loop where every new connection gets assigned to a new goroutine.

When the server receives a new client, it waits and try to parse the incoming command. If the commands is valid, a method from `commands` module gets called.

```go
func handleRequest(c protocol.Connection) {
	log.Printf("Connection from: %s\n", c.Conn.RemoteAddr().String())
	connectionAlive := true

	for connectionAlive {
		command, err := c.ReadCommand()
		if err != nil {
			connectionAlive = false
			break
		}
		log.Printf("Received: %s\n", command)
		dispatch(command, c)
	}

	c.Conn.Close()
}

func dispatch(command []string, c protocol.Connection) {
	switch strings.ToUpper(command[0]) {
	case "PING":
		commands.Ping(c)
	case "SET":
		commands.Set(c, command[1], command[2])
	case "GET":
		commands.Get(c, command[1])
	case "DEL":
		commands.Del(c, command[1:])
	case "KEYS":
		commands.Keys(c, command[1])
	case "EXPIRE":
		commands.Expire(c, command[1], command[2])
	case "TTL":
		commands.Ttl(c, command[1])
	case "COMMAND":
		commands.Command(c)
	default:
		log.Println("[Unknown command]")
	}
}
```

I tried to keep the `commands` module free of any network or database detail, this is the implementation of `GET`.

```go
func Get(c protocol.ConnectionInterface, key string) {
	value, exists := database.DB.Get(key)
	if exists {
		c.SendResponse(value)
	} else {
		c.SendResponse("(nil)")
	}
}
```

And `SET`.

```go
func Set(c protocol.ConnectionInterface, key, value string) {
	database.DB.Set(key, value)
	c.SendResponse("OK")
}
```

I have built two database implmentations, one for an in-memory database and another one for SQLite. Both modules share a similar interface.

```go
type DatabaseInterface interface {
	Init(connection string) error
	cleanExpiredKeys()
	Set(key, value string)
	Get(key string) (value string, exists bool)
	Del(keys []string)
	Keys(pattern string) []string
	Expire(key string, seconds int) bool
	Ttl(key string) int64
}
```

### In-memory

The data type holding the data for the in-memory backend is.

```go
type InMemory struct {
	values     map[string]string
	expiration map[string]int64
	mutex      sync.RWMutex
}
```

`values` for the information we want to store, expiration to support `TTL` and a mutex to ensure that we don’t run into concurrency issues.

Most of the operations are pretty straightforward.

```go
func (db *InMemory) Set(key, value string) {
	db.mutex.Lock()
	defer db.mutex.Unlock()

	db.values[key] = value
	delete(db.expiration, key)
}

func (db *InMemory) Get(key string) (value string, exists bool) {
	db.mutex.RLock()
	defer db.mutex.RUnlock()

	value, exists = db.values[key]
	expiration, expirationExists := db.expiration[key]

	if expirationExists && expiration < time.Now().Unix() {
		return "", false
	}
	return value, exists
}

func (db *InMemory) Del(keys []string) {
	db.mutex.Lock()
	defer db.mutex.Unlock()

	for _, key := range keys {
		delete(db.values, key)
	}
}
```

I like how `defer` makes easy to pair the `Lock` and `Unlock` of the mutexes but I feel that it is easy to introduce subtle bugs if one is not careful.

If one key is never accessed after its expiration it will be kept in memory forever, to avoid that in the `Init` method call I start a goroutine to clean all the keys that are expired already. This makes the server more memory efficient.

```go
func (db *InMemory) Init(connection string) error {
	db.values = make(map[string]string)
	db.expiration = make(map[string]int64)
	go db.cleanExpiredKeys()

	return nil
}

func (db *InMemory) cleanExpiredKeys() {
	for {
		time.Sleep(10 * time.Second)
		db.mutex.Lock()

		for key, expiration := range db.expiration {
			if expiration < time.Now().Unix() {
				delete(db.values, key)
				delete(db.expiration, key)
				log.Println("Expired key:", key)
			}
		}

		db.mutex.Unlock()
	}
}
```

The goroutine runs every 10 seconds and it locks the entire database for writes. This could be more efficient keeping a list of the keys with expiration at the cost of higher memory usage. Or it could lock up to a maximum time to avoid large pauses.

Note that I do not use `defer` to unlock the mutex here, this is important since the method never finishes.

### SQLite

For the SQLite implementation I am using [go-sqlite3](https://github.com/mattn/go-sqlite3).

The database structure look like this.

```sql

CREATE TABLE IF NOT EXISTS data (
	id INTEGER NOT NULL PRIMARY KEY,
	key VARCHAR(256) NOT NULL,
	value TEXT,
	expiration INTEGER
);

CREATE UNIQUE INDEX IF NOT EXISTS idx_data_key ON data (key);
CREATE UNIQUE INDEX IF NOT EXISTS idx_data_expiration ON data (expiration);
```

The implementation of the operations is a bit more complex than for the in-memory database.

```go
func (sqlite *SQLite) Set(key, value string) {
	sqlite.db.Exec("DELETE FROM data WHERE key=?;", key)
	sqlite.db.Exec("INSERT INTO data VALUES(NULL,?,?,NULL);", key, value)
}

func (sqlite *SQLite) Get(key string) (string, bool) {
	var value string

	now := time.Now().Unix()
	row := sqlite.db.QueryRow("SELECT value FROM data WHERE key = ? AND (expiration >= ? OR expiration IS NULL)", key, now)

	if err := row.Scan(&value); err != nil {
		return "", false
	}

	return value, true
}
```

Cleaning the expired keys get reduced to one SQL statement.

```go
func (sqlite *SQLite) cleanExpiredKeys() {
	for {
		time.Sleep(10 * time.Second)

		now := time.Now().Unix()
		sqlite.db.Exec("DELETE FROM data WHERE expiration < ? OR expiration IS NOT NULL;", now)
	}
}
```

### Testing

The testing library I have used is [testify](github.com/stretchr/testify), it is one of the most popular libraries for Go. The tests are pretty legible and they run very fast.

```go
func TestSQSetGetDel(t *testing.T) {
	setupSQ()
	defer teardownSQ()
	var key, value = "key", "value"

	assert := assert.New(t)

	DB.Set(key, value)
	DB.Del([]string{key})
	result, exists := DB.Get(key)

	assert.Equal(false, exists)
	assert.Equal("", result)
}
```

## Final thoughts

Concurrency is always hard but Go constructs ( goroutines, channels, mutexes ) make it much easier.

Testing is not as flexible as in ruby, but it is good enough for most project.

The code of the blog post can be downloaded [here](https://github.com/javiyu/redis_lite).

## Links

[Redis protocol spec](https://redis.io/docs/reference/protocol-spec/)

[Repo](https://github.com/javiyu/redis_lite)

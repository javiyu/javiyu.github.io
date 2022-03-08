---
layout: post
title:  "How to create a RuboCop custom cop"
date:   2022-03-04 08:00:00 +0100
categories: ruby rubocop
---

## Introduction

Ruby is a highly dynamic language. As long as the syntax of the code you write is correct, the program will start running until it finishes or reaches a statement that causes a runtime error. The developer have a lot of freedom to choose how the program interact with the different objects. Since there is not static type check ruby community usually relay a lot on [unit testing](https://en.wikipedia.org/wiki/Unit_testing).

When writing ruby you can still benefit from some help. [RuboCop](https://rubocop.org/) is a static code analyzer with a lot of extensions such as [RuboCop Rails.](https://docs.rubocop.org/rubocop-rails/index.html)

It catches some basic syntax errors, formatting issues and provide insight into best practices. RuboCop has integrations with the most used code editors for ruby development.

It is possible for everyone to write a new cop (extension) and that is what this post is about, we will create it step by step.

## Description of the problem

Imagine you are dealing with some code like this one.

```ruby
def calculate_new_value(value)
  value * 2
end

array = [1, 2, 3, 4, 5]
result = array.each do |i|
  calculate_new_value(i)
end
```

Any experienced ruby developer will catch the error but it is not trivial if you have never wrote any ruby code. The author of that snippet intended to get the in `result` an array with the new values produced after calling `calculate_new_value`, but instead got the original array back.

`each` returns the collection you are iterating. To collect the new values produced by your calculation you would need to instead call `map`.

Code like this appear in any large ruby project, and production code is usually more complex. It is useful to have a RuboCop rule to detect when we are assigning the return value of an `each` block.

In the next section we are going to create a RuboCop extension for that.

## How to create a cop

### Basic structure

The easiest path to start writing a new cop is using [rubocop-extension-generator](https://github.com/rubocop/rubocop-extension-generator).

```ruby
rubocop-extension-generator rubocop-each_return
```

When running this command it produces the basic structure of a gem, with everything prepared for you to start writing your cop.
First thing we need to do is to write some unit tests.

```ruby
RSpec.describe(RuboCop::Cop::EachReturnValue, :config) do
  subject(:cop) { described_class.new(config) }

  it "adds an offense when using the return value of an each block" do
    expect_offense(<<~RUBY)
      value = a.each {|x| x}
      ^^^^^^^^^^^^^^^^^^^^^^ Do not use the return value of .each
    RUBY
  end

  it "adds an offense when using the return value of an each block when calling an array" do
    expect_offense(<<~RUBY)
      value = [1,2,3].each {|x| x}
      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ Do not use the return value of .each
    RUBY
  end
end
```

### Writing the cop

We only need the first test case. I like to add a variation to make sure the code does not assume something very specific of the snippet I’m using in my test.

Now you may be wondering, how we do this? should we use regex? any kind of ruby magic you have not been told yet?
We are going to use a representation of the code called AST.

An AST is a tree representation of the code, you can see the representation of your code with a tool called `ruby-parse`.

```ruby
➜ ruby-parse -e 'value = a.each {|x| x}'
(lvasgn :value
  (block
    (send
      (send nil :a) :each)
    (args
      (procarg0
        (arg :x)))
    (lvar :x)))
```

As you can imagine this representation helps but it would still be a lot of work to identify the cases we are interested on. RuboCop makes it easier by introducing the [Node Pattern](https://github.com/rubocop/rubocop-ast/blob/master/docs/modules/ROOT/pages/node_pattern.adoc).

This pattern is a DSL to help you in the task of identifying sections of the code.
So, how would the pattern look in this case?

```ruby
module RuboCop
  module Cop
    class EachReturnValue < RuboCop::Cop::Base
      def_node_matcher(:array_each?, '(lvasgn _ (block (send _ :each) ...))')

      def on_lvasgn(node)
        if array_each?(node)
          add_offense(node, message: 'Do not use the return value of .each')
        end
      end
    end
  end
end
```

A lot to digest here, the callbacks offered by RuboCop follow the naming `on_#{node_type}`, we are interested on the left var assignments so we need to implement `on_lvasgn`.

The matcher looks difficult `(lvasgn _ (block (send _ :each) ...))`. `_` matches any node and `...` any number of nodes, kind of like `?` and `*` in regular expressions. My process to get this matcher is to replace one bit at a time, making it a more generic each step.

```ruby
(lvasgn :value (block (send (send nil :a) :each) (args (procarg0 (arg :x))) (lvar :x))) # original
(lvasgn _ (block (send (send nil :a) :each) (args (procarg0 (arg :x))) (lvar :x))) # :value is too specific, replaced by _
(lvasgn _ (block (send _ :each) (args (procarg0 (arg :x))) (lvar :x))) # (send nil :a) is also too specific, replaced by _
(lvasgn _ (block (send _ :each) ...)) # we dont need to know anything about the block, it can be just hidden under ...
```

Finally the method we have available to mark a problem in a node is `add_offense`.

### Autocorrect

If you have ever used RuboCop you may know that it can correct a good part of the linting errors automatically. Run `rubocop -a` and all the safe changes get written to disk. In our case `value = a.each {|x| x}` should get fixed as `a.each {|x| x}`, again, we start updating the test cases. We introduce `expect_correction`.

```ruby
RSpec.describe(RuboCop::Cop::EachReturnValue, :config) do
  subject(:cop) { described_class.new(config) }

  it "adds an offense when using the return value of an each block" do
    expect_offense(<<~RUBY)
      value = a.each {|x| x}
      ^^^^^^^^^^^^^^^^^^^^^^ Do not use the return value of .each
    RUBY

    expect_correction(<<~RUBY)
      a.each {|x| x}
    RUBY
  end

  it "adds an offense and corrects node when using the return value of an each block when calling an array" do
    expect_offense(<<~RUBY)
      value = [1,2,3].each {|x| x}
      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^ Do not use the return value of .each
    RUBY

    expect_correction(<<~RUBY)
      [1,2,3].each {|x| x}
    RUBY
  end
end
```

We need to find a way to remove the `variable =` portion of the statement, RuboCop provides a `corrector` object to help us with that.

```ruby
# frozen_string_literal: true

require 'rubocop'

module RuboCop
  module Cop
    class EachReturnValue < RuboCop::Cop::Base
      extend AutoCorrector
      include RangeHelp

      def_node_matcher(:array_each?, '(lvasgn _ (block (send _ :each) ...))')

      def on_lvasgn(node)
        if array_each?(node)
          add_offense(node, message: 'Do not use the return value of .each') do |corrector|
            corrector.remove(node.loc.name)
            corrector.remove(range_with_surrounding_space(range: node.loc.operator))
          end
        end
      end
    end
  end
end
```

Since we are dealing with the AST instead of raw strings the task is much easier. We remove the name of the statement and the operator. It works well but it left some unwanted whitespaces in the output. We can make use of the helper `range_with_surrounding_space` to identify the surrounding white spaces.

## Final thoughts

The final version of the code is [here](https://github.com/javiyu/rubocop-each_return), I have published the [gem](https://rubygems.org/gems/rubocop-each_return) in case you want to try it.

You may be thinking, “ok, this is great, but only if I want to contribute to RuboCop” and that is probably not a priority to you. Most of big ruby codebases I have seen over the years follow some unwritten conventions. Things like “operation classes must have a perform method”, “only call method should be public”, “we don’t want to use default_scope in our project”.

You can write those rules as RuboCop cops and integrate them in your CI system. Any developer would have access to that knowledge when writing the code and if it get missed the CI would fail. Instead of repeating those over and over again when reviewing pull requests.

## Links

[Rubocop Development Docs](https://docs.rubocop.org/rubocop/1.25/development.html)

[Node Pattern](https://github.com/rubocop/rubocop-ast/blob/master/docs/modules/ROOT/pages/node_pattern.adoc)

[Rubocop Extension Generator](https://github.com/rubocop/rubocop-extension-generator)

[Rubocop Rails](https://github.com/rubocop/rubocop-rails) has many examples of how to write a cop

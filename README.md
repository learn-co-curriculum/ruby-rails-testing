# Testing in Rails

One of the most fundamental aspects of programmer productivity is **the feedback
loop**. "Scripting" languages like Ruby and Python are great for this, because
you can run your code immediately after writing it, as opposed to lower-level
languages like C, which must be compiled before being run.

![Two programmers swordfight from their office chairs, using "Compiling!" as an
excuse](http://imgs.xkcd.com/comics/compiling.png "Compiling!")

## The Feedback Loop

1. Decide what to do next
1. Think of an approach
1. Write some code
1. Compile/run the code
1. Observe the output
  - If it looks good, move on to Step 1 for the next task.
  - If there are problems, go back to Step 1 for this task.

Having a short feedback loop--from brain to fingertips to running
process--lowers the resistance for trying new things, and protects you from
distractions that sneak in while you're waiting between steps.

Unfortunately, there's more than one way to distract a programmer!

## The Rails Feedback Loop

1. Decide what to do next
1. Read every related Rails Guide at least twice
1. Spend an hour poring over StackOverflow answers from two major versions ago
1. Copy-paste some sketchy-looking code
1. Run the code, which immediately screws up your database
1. Run `bundle exec rake db:reset`
1. Get a migraine because you forgot to put your painstakingly-designed seed
   data in `db/seeds.rb` and now it's all gone
1. Go for a walk
1. Write some code
1. Refresh browser
1. Scratch your head over why it says there's a missing template
1. Spend another hour tweaking your new code until you realize you made a typo
   in the filename and nothing you've been doing for the past three hours had a
   chance of working in the first place
1. Take a break to read *The Hitchhiker's Guide to Rails*:

> *Rails is big. Really big. You just won't believe how vastly, hugely,
> mind-bogglingly big it is. I mean, you may think it's a long way down the road
> to the chemist, but that's just peanuts to Rails.*
>
> -- Douglas Adams

(Okay, so maybe Mr. Adams was talking about space, but he totally *would have*
said it this way if he was writing about web development instead of interstellar
travel.)

Rails ships with many features to save precious seconds in developer feedback
loops, but there's no two ways about it: in anything but the most trivial app,
it can be pretty complex to make sure your code is actually working correctly.

In this lesson, we'll learn to shorten our feedback loop with different flavors
of Rails tests, combining some standard approaches suggested in the Guides
themselves with some more advanced practices that require additional
dependencies (namely Capybara).


# Objectives

After this lesson, you should be able to...

- List the different types of tests applicable to a Sinatra/Rails app.
- Compare and contrast the different types of tests.
- Describe the usage of Capybara in feature testing.


# Three Test Types

We'll be covering three types of tests:

- **Models** (RSpec)
- **Controllers** (RSpec)
- **Features** (RSpec/Capybara)

Features are the fanciest, so we'll leave them for last. They are preferred over
regular Rails "View" tests.


# RSpec

By default, Rails uses `Test::Unit` for testing, which keeps its tests in the
mysteriously-named `test/` folder.

If you're planning from the start to use RSpec instead, you can tell Rails to
skip `Test::Unit` by passing the `-T` flag to `rails new`, like so:

```
rails new cool_app -T
```

Then, you will add the gem to your Gemfile:

```ruby
gem 'rspec-rails'
```

And use the builtin generator to add a `spec` folder with the right boilerplate:

```
bundle install
bundle exec rails g rspec:install
```

This is the Rails equivalent of the usual `rspec --init`.


# Model Tests

These go in `spec/models`, one file per model.

Model tests use the least amount of special features, since all you really need
is the model class itself. The most common usage for model tests is to make sure
you have set up your validations correctly.

Suppose we're working with this model:

```ruby
# app/models/monster.rb

class Monster < ActiveRecord::Base
  validates :name, presence: true
  validates :size, inclusion: { in: ["tiny", "average", "like, REALLY big"] }
  validates :taxonomy, format: { with: /\A[a-zA-Z]+ [a-zA-Z]+\z/,
    message: "must include genus and species, like 'Homo Sapien'" }
end
```

## Testing for Validity

First, we'll make sure that it understands a valid Monster:

```ruby
# spec/models/monster_spec.rb

describe Monster do
  let(:attributes) do
    {
      name: "Dustwing",
      size: "tiny",
      taxonomy: "Abradacus Nonexistus"
    }
  end

  it "is considered valid" do
    expect(Monster.new(attributes)).to be_valid
  end
end
```

Some questions to answer first:

### What is `let`?

If you haven't seen `let` before, it is a [standard helper method][rspec_let]
that takes a symbol and a block. It runs the block **once per example** in which
it is called, and saves the return value in a local variable named according to
the symbol. This means you get a fresh copy in every test case.

[rspec_let]: http://www.relishapp.com/rspec/rspec-core/docs/helper-methods/let-and-let

### Why is `let` better than `before :each`?

It's more fine-grained, which means you have better control over your data. It
can be used in combination with `before` statements to set up your test data
*just right* before the examples are run.

### Why did we use `let` to make an attribute hash?

We could have put the entire `Monster.new` call inside our `let` block, but
using an attribute hash instead has some advantages:

- If we want to tweak the data first, we can just pass `attributes.merge(name:
  "Other")` while preserving the rest of the attributes.
- We can also refer to `attributes` when making assertions about what the actual
  object should look like.

It's a good balance between saving keystrokes and maintaining flexibility over
your test data.

### Where does `be_valid` come from?

RSpec provides plenty of builtin matchers, which you can peruse in their [API
docs][rspec_api_matchers], but `be_valid` is conspicuously absent from the list.

[rspec_api_matchers]: http://rspec.info/documentation/3.4/rspec-expectations/frames.html#!RSpec/Matchers.html

This code uses a neat trick that RSpec refers to as "[predicate
matchers][rspec_predicate]", and you'll see it **a lot** in Rails testing.

In Ruby, it's conventional for methods that return `true` or `false` to be named
with a question mark at the end. These methods are called **predicate methods**,
because "predicate" is an English grammar term for the part of a sentence that
makes a statement about the subject.

[rspec_predicate]: https://www.relishapp.com/rspec/rspec-expectations/docs/built-in-matchers/predicate-matchers

As you learned earlier in this unit, Rails provides a `valid?` method that
returns `true` or `false` depending on whether the model object in question
passed its validations.

In RSpec, when you call a nonexistant matcher (such as `be_valid`), it strips
off the `be_` (`valid`), adds a question mark (`valid?`), and checks to see if
the object responds to a method by that name (`monster.valid?`).

## Testing for Validation Failure

Now, let's add some tests to make sure our validations are working in the
opposite direction:

```ruby
# spec/models/monster.rb

  let(:missing_name) { attributes.except(:name) }
  let(:invalid_size) { attributes.merge(size: "not that big") }
  let(:missing_species) { attributes.merge(taxonomy: "Abradacus") }

  it "is invalid without a name" do
    expect(Monster.new(missing_name)).not_to be_valid
  end

  it "is invalid with an unusual size" do
    expect(Monster.new(invalid_size)).not_to be_valid
  end

  it "is invalid with a missing species" do
    expect(Monster.new(missing_species)).not_to be_valid
  end
```

Note that each of these `let` blocks rely on the first one, `attributes`, which
contains all of our valid attributes. `missing_name` uses the Rails Hash helper
`except` to exclude the `name`, while the other two use the Ruby standard
`merge` to overwrite valid attributes with invalid ones.

## I saw some search results using `should`. What is that?

`should` is an older RSpec syntax that has been deprecated in favor of `expect`.

## Any other RSpec features to know about?

Several RSpec features have been moved over time into the
[rspec-collection_matchers][collection_matchers] gem, which can make some
detailed assertions more readable.

[collection_matchers]: https://github.com/rspec/rspec-collection_matchers


# Controller Tests



# Feature Tests

## Capybara

"Feature" is a Capybara term. Features encompass views, and sometimes also
controllers.

When you see key words like `visit`, `fill_in`, and `page`, you 


<a href='https://learn.co/lessons/ruby-rails-testing' data-visibility='hidden'>View this lesson on Learn.co</a>

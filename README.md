# Testing in Rails

One of the most fundamental aspects of programmer productivity is **the feedback
loop**. "Scripting" languages like Ruby and Python are great for this because
you can run your code immediately after writing it. Conversely, lower-level
languages like C must be compiled before being run.

![Two programmers swordfight from their office chairs, using "Compiling!" as an
excuse](http://imgs.xkcd.com/comics/compiling.png "Compiling!")

## The Feedback Loop

1. Decide what to do next
2. Think of an approach
3. Write some code
4. Compile/run the code
5. Observe the output
  - If it looks good, move on to Step 1 for the next task.
  - If there are problems, go back to Step 1 for this task.

Having a short feedback loop –– from brain to fingertips to running
process –– lowers the resistance for trying new things and protects you from
distractions that sneak in while you're waiting between steps.

Unfortunately, there's more than one way to distract a programmer!

## The Rails Feedback Loop

1. Decide what to do next
2. Read every related Rails Guide at least twice
3. Spend an hour poring over StackOverflow answers from two major versions ago
4. Copy-paste some sketchy-looking code
5. Run the code, which immediately screws up your database
6. Run `bundle exec rake db:reset`
7. Get a migraine because you forgot to put your painstakingly-designed seed
   data in `db/seeds.rb`, and now it's all gone
8. Go for a walk
9. Write some code
10. Refresh browser
11. Scratch your head over why it says there's a missing template
12. Spend another hour tweaking your new code until you realize you made a typo
   in the filename and nothing you've been doing for the past three hours had a
   chance of working in the first place
13. Take a break to read *The Hitchhiker's Guide to Rails*:

> *Rails is big. Really big. You just won't believe how vastly, hugely,
> mind-bogglingly big it is. I mean, you may think it's a long way down the road
> to the chemist, but that's just peanuts to Rails.*
>
> -- Douglas Adams

(Okay, so maybe Mr. Adams was talking about space, but he totally *would have*
said it this way if he'd been writing about web development instead of interstellar
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

And use the built-in generator to add a `spec` folder with the right boilerplate:

```
bundle install
bundle exec rails g rspec:install
```

This is the Rails equivalent of the usual `rspec --init`.


# An Automobile Analogy

It's not enough to know *how* to test applications; we must also understand
*why* it makes sense to test them in a certain way. To help with this, try
thinking of Rails applications like cars:

- **Models** are the **basic parts** that make cars useful, like the fuel tank,
  engine, and tires.

- **Views** are the **interactive parts** that the driver (user) can see and
  touch, like the steering wheel, pedals, and gear shift.

- **Controllers** are the rest of the **connecting parts** under the hood that
  connect the views to the models, like how the steering column (along with the
  rest of the steering assembly) connects the wheel to the tires. The users don't
  really see them, think of them, or even necessarily know they exist, but they're
  just as necessary.

Models are not too difficult to test because they have very specific purposes
that can be easily separated from the rest of the application.

This holds true for cars as well: a good tire has a tread that grips the road in
adverse conditions, doesn't puncture easily, and so on. You don't really need to
think about the rest of the car when you're testing what makes a good wheel.

This gets a little trickier for views and controllers, but we'll talk more about
that once we're done with models.


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
  validates :taxonomy, format: { with: /\A[A-Z](\.|[a-z]+) [a-z]{2,}\z/,
    message: "must include genus and species, like 'Homo sapiens'" }
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
      taxonomy: "Abradacus nonexistus"
    }
  end

  it "is considered valid" do
    expect(Monster.new(attributes)).to be_valid
  end
end
```

Some questions to answer first:

### What is `let`?

If you haven't seen `let` before, it is a [standard helper method](http://www.relishapp.com/rspec/rspec-core/docs/helper-methods/let-and-let)
that takes a symbol and a block. It runs the block **once per example** in which
it is called and saves the return value in a local variable named according to
the symbol. This means you get a fresh copy in every test case.

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

It's a good balance between saving keystrokes and maintaining the flexibility of
your test data.

### Where does `be_valid` come from?

RSpec provides plenty of built-in matchers, which you can peruse in their [API
docs](http://rspec.info/documentation/3.4/rspec-expectations/frames.html#!RSpec/Matchers.html), but `be_valid` is conspicuously absent from the list.

This code uses a neat trick that RSpec refers to as "[predicate
matchers](https://www.relishapp.com/rspec/rspec-expectations/docs/built-in-matchers/predicate-matchers)", and you'll see it **a lot** in Rails testing.

In Ruby, it's conventional for methods that return `true` or `false` to be named
with a question mark at the end. These methods are called **predicate methods**,
because "predicate" is an English grammar term for the part of a sentence that
makes a statement about the subject.

As you learned earlier in this unit, Rails provides a `valid?` method that
returns `true` or `false` depending on whether the model object in question
passed its validations.

In RSpec, when you call a nonexistent matcher (such as `be_valid`), it strips
off the `be_` (`valid`), adds a question mark (`valid?`), and checks to see if
the object responds to a method by that name (`monster.valid?`).

## Testing for Validation Failure

Now, let's add some tests to make sure our validations are working in the
opposite direction:

```ruby
# spec/models/monster_spec.rb

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
contains all of our valid attributes. `missing_name` uses the Rails hash helper
`except` to exclude the `name` key while the other two use the standard Ruby
`merge` method to overwrite valid attributes with invalid ones.

## I saw some search results using `should`. What is that?

`should` is an older RSpec syntax that has been deprecated in favor of `expect`.

## Any other RSpec features to know about?

Several RSpec features have been moved over time into the
[rspec-collection_matchers](https://github.com/rspec/rspec-collection_matchers) gem, which can make some
detailed assertions more readable.


# Controller Tests

The biggest risk in writing controller tests is redundancy: controllers exist
to connect views and models, so it's difficult to test them in isolation.

First, we'll go over **how** to write controller tests. Then, our discussion of
the **why** will bring us into the final subject, "feature tests".

```ruby
# spec/controllers/monsters_controller_spec.rb

describe MonstersController, type: :controller do
  let(:attributes) do
    {
      name: "Dustwing",
      size: "tiny",
      taxonomy: "Abradacus nonexistus"
    }
  end

  it "renders the show template" do
    monster = Monster.create!(attributes)
    get :show, id: monster.id
    expect(response).to render_template(:show)
  end

  describe "creation" do
    before { post :create, monster: attributes }
    let(:monster) { Monster.find_by(name: "Dustwing") }

    it "creates a new monster" do
      expect(monster).to_not be_nil
    end

    it "redirects to the monster's show page" do
      expect(response).to redirect_to(monster_path(monster))
    end
  end
end
```

You can use the `get` and `post` methods (along with `patch` and `delete`) to
initiate test requests on the controller. A `response` object is available to
set expectations on, such as `render_template` or `redirect_to`.

The tests above are great, especially while we're still getting used to how
controllers are wired. However, almost these exact tests could be copied for
*any* controller set up according to Rails' RESTish conventions. There's nothing
inherently wrong with that, but the redundance, along with the need to test
views, inspired the creation of a new type of test supported by Capybara known
as a "Feature Test".


# Feature Tests

If you were going to write tests for a car's steering wheel, what would you
start with?

Here's one idea:

> When the steering wheel is rotated to the left, the tires rotate to the left.

This makes sense, but it's testing much more than the steering wheel. This test
relies on the view (steering wheel), the model (tires), *and* the controller
(steering column)!

This is called an **acceptance test** because it is phrased in terms of
features that provide value to the user. (It could also be called an
**integration test** because it tests more than one piece of the system at
once.)

Can we *isolate* the steering wheel while still testing its functionality?

Not really –– the whole point of the steering wheel is to control the tires. We
could talk about how it looks or what it's made of, but the functionality is
inherently tied to the underlying system, just like the views in a Rails app.
All of those forms and templates are meaningless without controllers and models
to populate them.

In the last section, we did our best to isolate the controller, and, as a result,
we wrote many of our tests in terms of the controller's internal parts (such as
redirects and request methods). We don't care what the HTML looks like, what
button the user pressed, or how the models are behaving.

This is called a **unit test**, because it tests a single unit of functionality.

For a car, it might look like this:

> When the steering column's flange rotates, the steering shaft transmits the
> rotation to the steering box.

Phew. Good thing they covered steering assemblies on the last episode of Car
Talk, or I wouldn't have known where to start!

By now you're probably realizing that if you say "steering" enough times, it
stops sounding like a real word.

But, more importantly, it can be difficult to write isolated **unit tests**, and
it's not always clear whether they're useful. Compare the jargon-heavy, extremely
specific unit test, above, to this test covering the steering wheel
(view) *and* steering column (controller):

> When the steering wheel is rotated to the left, the steering column transmits
> the rotation to the steering box.

This is a **feature test**.

The **acceptance test** at the top of this section covers too much ground,
making it brittle and difficult to maintain.

The **unit test** in the middle of this section is so specific that it almost
feels like we just rewrote the controller code with different phrasing.

The **feature test**, on the other hand, is Just Right. It lets us think like a
user (in terms of the steering wheel, or view) while still making intelligent
assertions about how the underlying system should respond to input (in terms of
the steering column, or controller).

Now, on to the *how*.


## Capybara

When you see key words like `visit`, `fill_in`, and `page`, you know you're
looking at a [Capybara](https://github.com/jnicklas/capybara) test.

To set up Capybara, one must first add the gem to the `Gemfile`:

```
gem 'capybara'
```

Then set up Capybara-Rails integration in `spec/rails_helper.rb`:

```ruby
require 'capybara/rails'
```

Then set up Capybara-RSpec integration in `spec/spec_helper.rb`:

```ruby
require 'capybara/rspec'
```

Feature tests are traditionally located in `spec/features`, but you can put them
anywhere if you pass the `:type => :feature` option to your `describe` call.

To test our monster manager with Capybara, we'll start by setting up a `GET`
request and then use Capybara's convenient helper functions to interact with the
page just like a user would:

```ruby
# spec/features/monster_creation.rb

describe "monster creation", type: :feature do
  before do
    visit new_monster_path
    fill_in "Name", with: "Dustwing"
    select "tiny", from: "monster_size"
    fill_in "Taxonomy", with: "Abradacus nonexistus"
    click_button "Create Monster"
  end
```

When `click_button` is called, this will trigger the `POST` request to the
controller's `create` action, just as if a user had clicked it in their browser.

Now, we can write our original controller tests like usual:

```ruby
  let(:monster) { Monster.find_by(name: "Dustwing") }

  it "creates a monster" do
    expect(monster).to_not be_nil
  end

  it "redirects to the new monster's page" do
    expect(current_path).to eq(monster_path(monster))
  end
```

And because we're in Capybara land, we also have a very convenient way of making
assertions about the final `GET` request:

```ruby
  it "displays the monster's name" do
    within "h1" do
      expect(page).to have_content(monster.name)
    end
  end
```

`within` sets the context for our next expectation, restricting it to the first
`<h1>` tag encountered on the page. This way, our `expect` call will only pass
if the specified content (`"Dustwing"`) appears inside that first heading.

One interesting thing about this approach is that we're being much *less*
explicit about certain expectations. For example, we're testing the redirect not
with the initial `302` response but instead by examining the current path in
Capybara's virtual "browser session". This is much more powerful and intuitive,
and it doesn't sacrifice much in the way of expressivity.


# Summary

The hardest part about testing usually ends up being the "why" and not the
"how". Why write the test this way and not that way?

You will probably see Capybara feature tests in wider usage than direct
controller and view tests, but they're not universal. If you're doing something
unusual, like a series of complex redirects that change based on authorization
level, it makes sense to write a more isolated controller test. But for standard
CRUD functionality, Capybara is designed to save you a lot of time and mental
effort.

These can serve as fairly reliable guidelines:

- Models should always be thoroughly unit tested.
- Controllers should be as thin as possible to keep your feature tests simple.
- If you can't avoid making a controller complex, it deserves its own isolated
  test.
- Capybara's syntax is much more powerful than Rails's built-in functionality for
  view tests, so stick with it whenever possible.
- Don't get carried away with the details when testing views: you just need to
  make sure the information is in the right place. If your tests are too strict,
  it will be impossible to make even simple tweaks to your templates without
  breaking the build.
  

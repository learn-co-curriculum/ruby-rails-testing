# Testing in Sinatra/Rails

## Models

The model spec essentially tests the core building blocks of a Rails application, which in this case, refers to the model object. Having a well-tested application at the model level ensures that you have a big step towards a reliable code base.

A model spec tests specifically for three primary pieces of information:
    * `create` method, when passed valid attributes, is valid 
    * Validations fail when when passed invalid attributes
    * Class and instance methods perform as expected

Let's go through an example of what validation tests could be written for a model:



```ruby
describe Contact do
  it "is valid with a firstname, lastname and email" do
  end

  it "is invalid without a firstname" do 
  end

  it"is invalid without a lastname" do
  end

  it "is invalid without an email address" do
  end
end
```

These are blank boilerplate model tests. These are model validation RSpec tests. We're testing the data integrity and quality of the attributes of a `Contact` object. 

What is each test doing?

* Each line describes a **set of expectations**.
* Each example expects only **one** thing.
* Each example is **explicit**.
* Each description begins with a **verb**, not `should`.

Let's write out the tests in the body.

```ruby
describe Contact do
  it "is valid with a firstname, lastname and email" do
    expect(Contact.new(firstname: "Josh", lastname: "Owens", email: "josh@flatironschool.com")).to be_valid
  end

  it "is invalid without a firstname" do 
    expect(Contact.new(firstname: nil)).to have(1).errors_on(:firstname)
  end

  it"is invalid without a lastname"do
    expect(Contact.new(lastname: nil)).to have(1).errors_on(:lastname)
  end

  it "is invalid without an email address" do
    expect(Contact.new(email: nil)).to have(1).errors_on(:email)
  end
end
```

Let's write out a test for the following instance method on the `Contact` model:

```ruby
def name
  [firstname, lastname].join(' ')
end
```

We have a `name` method that puts the `firstname` and `lastname` attributes into an array, and then joins them into a string. Let's write the description out below.

```ruby
it"returns a contact's full name as a string" do
  ...
end
```

And here's the final spec.

```ruby
it"returns a contact's full name as a string" do
  contact = Contact.new(firstname: 'Josh', lastname: 'Owens',
    email: 'josh@flatironschool.com.com')
  expect(contact.name).to eq 'Josh Owens'
end
```

What's an example of a happy versus a sad path for a model test? Look at the descriptions for the test below.

```ruby
describe Contact do
  # validation examples ...
  describe "filter last name by letter" do 
    context "matching letters" do
      # matching examples ...
    end

    context "non-matching letters" do 
      # non-matching examples ...
￼￼  end
  end
end
```

Now let's write out the tests.

```ruby
describe Contact do
  before :each do
    @smith = Contact.create(firstname: 'John', lastname: 'Smith',
    email: 'jsmith@example.com')
    @jones = Contact.create(firstname: 'Tim', lastname: 'Jones',
    email: 'tjones@example.com')
    @johnson = Contact.create(firstname: 'John', lastname: 'Johnson',
    email: 'jjohnson@example.com')
  end

  context "matching letters" do
    it "returns a sorted array of results that match" do
      expect(Contact.by_letter("J")).to eq [@johnson, @jones]
    end
  end

  context "non-matching letters" do
    it "returns a sorted array of results that match" do
      expect(Contact.by_letter("J")).to_not include @smith
    end
  end
end
```

These tests are testing both the happy path (matching letters) and the sad path (non-matching letters).

## Controllers and Feature Tests

Testing controllers, in my opinion, is probably the hardest aspect of testing. Why is this? This is because there's so much you can do with controller tests, and it becomes more of a question of, *what should I test*?. There are two ways of testing controllers: via explicit controller tests, and also via feature tests.

Why should we test controllers? They're also classes with methods, and should be on equal footing with your Rails models too. The controller tests can be written more quickly than feature tests, and is much more straightforward than a feature test. The final benefit is that they run more quickly than feature tests, because feature tests typically interact with a headless browser via Selenium Webdriver (or another engine for Capybara).

Why should we not test controllers? Controllers should be skinny (fat models, skinny controllers). A feature test can achieve the results of multiple controller tests. It could potentially be easier to write and maintain a feature spec.

It's a good practice to do a mix of both. I like to use controller tests for the low-level process of a controller method, and then use a feature test to test the full end-to-end interaction of our controllers with the models, the views, and the database.

### Controllers

We'll review simple examples of controller tests in this section.

#### `GET` Requests

The simplest controller test involves the `GET` request. In Rails, there are four standard actions involving this: `index`, `show`, `edit`, and `new`. Let's do an example for the `show` action.

```ruby
describe'GET#show'do
  before(:each) do
    contact = create(:contact)
  end

  it "assigns the requested contact to @contact" do
    get :show, id: contact
    expect(assigns(:contact)).to eq contact
  end
  
  it "renders the :show template" do
    get :show, id: contact
    expect(response).to render_template :show
  end 
end
```

In this example, we're creating a `Contact` object using the FactoryGirl gem. 

In the first test, we send a get request to the `show` template, and we've passed in the `id` attribute of the contact object. We then set up the expectation that the instance variable `@contact` in the controller show action is the same as the contact object that we created. 

In the second test, we send a get request to the 'show' template again, passing in the `id` attribute of the contact object. Then we set up the expectation that the response actually renders the `show` template (contacts/show.html.erb).

Each test should have just one expectation (unless it's a feature test).

#### `POST` Requests

```ruby
describe "POST #create" do
  before(:each) do
   @phones = [
      attributes_for(:phone),
      attributes_for(:phone),
      attributes_for(:phone)
  end

  context "with valid attributes" do
    it "saves the new contact in the database" do
      expect{
        post :create, contact: attributes_for(:contact,
          phones_attributes: @phones)
      }.to change(Contact, :count).by(1)
    end
  
    it "redirects to contacts#show" do
      post :create, contact: attributes_for(:contact,
            phones_attributes: @phones)
      expect(response).to redirect_to contact_path(assigns[:contact])
    end 
  end
end
```

In a controller test for a `POST` request (which happens for the `create` action), we're interested in sending our form parameters to the `create` action, and then making sure that the controller handles that data properly.

In the first test, we've sent a post request to the `create` action, and we've sent along a contact object with nested attributes (for phone numbers). We're expecting that the POST request should increase the total number of Contact objects in the database by 1.

Then in the second test, we're testing that a successful form submission will redirect the user to the show page for the new Contact object. We're expecting that the response from the POST request will redirect to the contact show page for the newly created Contact object.

#### `PATCH` Requests

```ruby
describe 'PATCH#update' do 
  before :each do
    @contact = create(:contact,
      firstname: 'Lawrence', lastname: 'Smith')
  end
  
  context "valid attributes" do
    it "located the requested @contact" do
      patch :update, id: @contact, contact: attributes_for(:contact)
      expect(assigns(:contact)).to eq(@contact) 
    end

    it "changes @contact's attributes" do
      patch :update, id: @contact,
        contact: attributes_for(:contact,
          firstname: "Larry", lastname: "Smith")
      @contact.reload
      expect(@contact.firstname).to eq("Larry")
      expect(@contact.lastname).to eq("Smith")
    end
  end
end
```

The controller test for a `PATCH` request is similar to the one for the `POST` request, except that this is for an `update` action instead of a `create`. It is updating a currently existing record in the database.

The first test is about locating the correct Contact object. When a PATCH request is sent for a specific Contact object, `@contact`, we're checking to make sure that the updated object is, in fact, the correct Contact object.

The second test is about ensuring that the attributes of the correct Contact object were changed. The `@contact` object's attributes are as follows: `firstname` is Lawrence, and `lastname` is Smith. We're sending a PATCH request that updates the `firstname` attribute to Larry. Once the PATCH request is sent to the server, we have to reload the `@contact` object, since it has not been loaded since the PATCH request was sent. Once we reload it, then we can run the expectations to check if `firstname` has, in fact, been updated. 

#### `DELETE` Requests

```ruby
describe 'DELETE #destroy' do
  before(:each) do
    @contact = create(:contact)
  end
  
  it "deletes the contact" do expect{
    delete :destroy, id: @contact
    }.to change(Contact,:count).by(-1)
  end
  
  it "redirects to contacts#index" do
    delete :destroy, id: @contact
    expect(response).to redirect_to contacts_url
  end
end
```

We've set up a Contact object, `@contact`, to be created before each test runs. 

On the first test, we send a `DELETE` request for `@contact`, and we've set up an expectation that the `#count` method on the `Contact` class should be changed by one. If it doesn't change accordingly, then the `DELETE` action hasn't been set up properly.

On the second test, we send the same `DELETE` request as in the first test. The only difference is the expectation. We're expecting that, on successful deletion of the `@contact` object, we should be redirected to the `contacts_url` path.

### Features

A feature test is dependent on Capybara, which uses the browser to interact with your application. Let's do a simple example of a user logging in. 

```ruby
feature 'User management' do
  scenario "adds a new user" do
    admin = create(:admin)
    visit root_path
    click_link 'Log In'
    fill_in 'Email', with: admin.email
    fill_in 'Password', with: admin.password
    click_button 'Log In'
    visit root_path
    
    expect{
      click_link 'Users'
      click_link 'New User'
      fill_in 'Email', with: 'newuser@example.com'
      find('#password').fill_in 'Password', with: 'secret123'
      find('#password_confirmation').fill_in 'Password confirmation',
        with: 'secret123'
    click_button 'Create User'
    }.to change(User, :count).by(1)
    expect(current_path).to eq users_path
    expect(page).to have_content 'New user created'
    within 'h1' do
      expect(page).to have_content 'Users'
    end
    expect(page).to have_content 'newuser@example.com'
  end
end
```

This is a full-fledged feature test for adding a new user. Instead of a controller unit test, which is specifically testing a controller action, this is testing page interactions, and expecting a certain outcome based on those interactions.

We have a feature called `User Management`. Inside of each feature, we account for scenarios in which a user would interact with a page. For example, on a user registration page, a user might log in correctly, or log in incorrectly. This would represent two separate scenarios, and we want to test to make sure that each scenario results in the correct response from the application. If a user logs in correctly, the user should expect to see their profile page, or something that shows that they've logged in. If a user fails to log in correctly, then they should not be logged in, and they should have an error message that says that their attempt to log in was unsuccessful.

Following each of the scenarios, we set up the expectation. This is the component of the test that's checking to see if the scenario has executed correctly according to the test's specifications.

## Resources

* [RSpec](http://rspec.info) - [Documentation](http://rspec.info/documentation/)
* [Github](http://www.github.com/) - [Rspec Rails Examples](https://github.com/eliotsykes/rspec-rails-examples)
* [Flatiron School Books](http://books.flatironschool.com/) - [Everyday Rails Testing with RSpec](http://books.flatironschool.com/books/114)


<a href='https://learn.co/lessons/ruby-rails-testing' data-visibility='hidden'>View this lesson on Learn.co</a>

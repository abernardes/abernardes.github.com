---
layout: post
title:  "Decoupling forms from persistence"
date:   2014-10-07 20:00:00
---

I’ve been using a different approach to handle form data in the project I’m currently working on, with the awesome folks of [Pure Charity](http://purecharity.com).

This approach is called **Form Objects**. It’s based on Adam Hawkings’ [Rediscovering the Joy of Design](http://hawkins.io/2014/01/rediscovering-the-joy-of-design/) series.

In this post I’d like to share a couple of use cases where Form Objects have been helpful. This is by no means a comprehensive list. In fact, I plan to write a couple of articles focusing on other cases where I used this technique and got interesting results.

## Rationale

I have reasons to dislike how Rails couples the view layer to the persistence layer. If I build an app using ActiveRecord and later I decide to replace it with DataMapper, my views fall apart. This isn’t acceptable, in my opinion. My views should be completely oblivious about if and how I persist data.

I agree the default strategy of coupling forms with models helps a great deal most of the time. But then there are the cases where, at least in my experience, this coupling becomes a burden. Every workaround to make models and forms play nicely together - even The Rails Way® of doing it - were hard to reason about and difficult to maintain. For the cases listed below I found that Form Objects is a straightforward and flexible solution.

## Use cases

Before going through the use cases, I’d like to clearly state that **I don’t use Form Objects for all my forms, and neither should you**. Vanilla Rails forms backed by ActiveRecord objects are my tool of choice for the day-to-day forms. So before you point a finger at me and start yelling “BOILERPLATE CODE”, please consider if a Form Objects is not an overkill for the case you have in mind.

### Forms without models

If for example, I have to build a short contact form with fields like name, email, and a message. The contents of this form will be emailed to the person in charge.

I don’t really like the approach of using `form_tag` and then massaging giant hash structures to get the data I need when the user submits the form.

A Form Object that helps in this situation looks like this:

```ruby
class ContactForm
  # The following includes make this object an ActiveModel,
  # which means it will play nice with Rails’ form_for.
  include ActiveModel::Conversion
  include ActiveModel::Validations
  extend  ActiveModel::Naming

  # Getter/setter for each form field.
  attr_accessor :name, :email, :message

  # The constructor accepts a hash to initialise the attributes.
  # This is designed to work with Rails params hash. When the
  # user fills out and submits the form, I can create the Form
  # Object like so:
  #   @contact_form = ContactForm.new(params[:contact_form])
  #
  # The default value allows me to create an empty instance of
  # this Form class.
  def initialize(attrs={})
    attrs.each do |k, v|
      send("#{k}=", v)
    end
  end

  # Required by Rails forms. Make sure you set persisted?
  # correctly so the form uses the expected HTTP verb. Use
  # false for POST, true for PUT.
  def persisted?
    false
  end
end
```

All I have to do now is instantiate this form in `ContactsController#new` and use `form_for @contact_form` in the view. In `ContactsController#create`, I can create a new instance of the Form Object using the `params` hash, and then I can access the attributes directly.

```ruby
@contact_form = ContactForm.new(params[:contact_form])

@contact_form.name
#=> John Doe

@contact_form.email
#=> johndoe@example.com
```

One cool side effect of this strategy is that I get validations for free. I can use `validate :email, presence: true` and friends, and I’m guarded against bad data.

You might have noticed that I’m not using Rails 4 strong parameters in this example. Even if I were to persist the information in a model, it wouldn’t be needed. You don’t have to worry about protecting sensitive attributes because the form object only exposes attributes that should be set by the form.

### Forms for multiple models

I could (and I do) also use a Form Object for forms that with fields for two or more models. For example, you might have a signup form that has fields for login information, profile information, and payment information.

Instead of using the highly magical and not at all intention-revealing `accepts_nested_attributes_for`, it’s possible to gather all attributes in a Form Object, perform validations on them and then create each model individually.

To make this operation almost trivial, I add methods to Form Objects that return the data in a hash format as expected by ActiveRecord’s constructor. For example:

```ruby
class SignUpForm
  …
  def user_hash
     {
       login: login
       password: password
       email: email
     }
  end
end
```

And then, in the controller:

```ruby
@signup_form = SignupForm.new(params[:signup_form])
if @signup_form.valid?
  @user = User.create!(@signup_form.user_hash)
end
…
```

### Processing input data

I like to imagine my Form Objects are the gatekeeper of my application. Their primary role is to prevent bad data from coming in. Their secondary role is to get whatever data my app is getting from the external world and convert to the format the rest of the code expects.

Back to the contact form example, imagine I also have to send a “thanks for your contact” email back to the user. Without Form Objects, I would have to do something like this:

```ruby
if params[:contact][:email].present? &&
   params[:contact][:email]  =~ /"^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+$”/
  # Send email
end
```

Reaching two levels deep in a hash structure already gives me the creeps. Matching it against a regular expression in a procedural format like this is even worse (and this is a [simplified version of the regex](http://stackoverflow.com/questions/201323/using-a-regular-expression-to-validate-an-email-address)). Knowing this could be controller code, I’m sold to any other idea.

Fortunately, with a Form Object, all I have to do is add validations to make sure (1) the email field isn’t blank and (2) it is valid (to some extent, of course):

```ruby
class ContactForm
  …
  validate :email, presence: true,
                   format: /"^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+$”/
end
```

This technique is especially useful for dates and numbers, which could come in different formats for different locales. Form Objects can validate and perform the conversion so I can be sure I get valid information and in the format I expect.

The basic rationale is: I check the data coming into my system **at the border** of the the system and I check it **only once**. Once data passed the barrier of a form object, I can confidently perform operations on it without constantly having to double check `if expiration_date.nil?` or `if expiration_date.is_a?(Date)`.

## Closing Thoughts

I wish I could write more on how to use Form Objects for validation and conversion, but this post is already long enough.  I intend to expand on that in a future post.

This is what I have for now on Form Objects. Thoughts? Comments? Hit me up [on twitter](http://twitter.com/abernardes) or comment on Hacker News.

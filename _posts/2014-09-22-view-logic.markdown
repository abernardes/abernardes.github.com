---
layout: post
title:  "View Logic"
date:   2014-09-22 21:00:00
---

We all know: in Rails, or any other web framework for that matter, we have to keep business logic away from views. In this post I'd like to explore this subject a little more.

### Business logic in views

Suppose we are working in a service that has subscription plans. Each plan has a price. Price is a dollar amount charged monthly from the subscribers' credit cards. The product manager asked us to implement the following business rule:

> The price of a plan can't change if the plan has subscribers.

So I could fire up the editor, go to the edit form of plans and add the conditional:

```erb
<% f.text_field :price, disabled: @plan.subscribers.any? %>
```

Oops, now I have business logic in my view. In this scenario, it looks like a necessary evil. It's business logic, but I need that logic so the view knows whether it should enable or disable the price field.

So I get over it by telling myself this is probably OK. I'll just add some tests to make sure I won't accidentally break that in the future.

### Testing views

View testing can be a long topic of discussion. I see two ways of doing it in this case: acceptance testing (testing the app throught the UI) or view testing. Let's explore this options:

#### 1. Acceptance testing

Let's start with an example of an acceptance test. I'm using Capybara + Rspec, but you can use whatever combination of tools you like, you probably will end up with very similar results.

```ruby
feature 'editing plans' do
  scenario 'disabling the price field when plan has subscribers' do
    plan = FactoryGirl.create(:plan)

    visit edit_plan_path(plan)

    expect(page).to have_field 'Price', disabled: false

    subscriber = FactoryGirl.create(:user)
    plan.subscribers << subscriber

    visit edit_plan_path(plan)

    expect(page).to have_field 'Price', disabled: true
  end
end
```

Things to notice:

1. This is an acceptance test, so in order to test the disabled status of a field in a view, I had to exercise all layers of my application, all the way down to the database.
2. Altought not explicit here, this kind of test usually includes setting up a user, setting permissions, logging in and cleaning the database afterwards.
3. This test takes too long to run. It takes more time to run than it took me to implement the requirement. This is just wrong.

This is an overkill, no doubt about it. So let's see if we have better luck with view testing.

#### 2. View testing

So, here's an example of a spec for the view above. Again, I'm using RSpec.

```ruby
describe 'plan/_form.html.erb' do
  context 'the plan has subscribers' do
    it "disables the 'price' field" do
      assign(:plan, FactoryGirl.build(:plan))

      render

      expect(rendered).to have_disabled_field 'Price'
    end
  end

  context "the plan doesn't have subscribers" do
    it "enables the price field" do
      plan = FactoryGirl.build(:plan, subscribers: [FactoryGirl.build(:user)])
      assign(:plan, plan)

      render

      expect(rendered).to have_enabled_field 'Price'
    end
  end
end
```

Things to notice:

1. I decided to use FactoryGirl, but I could have done the setup by hand (`Plan.new`). Both are fine choices. I don't use test stubs in view testing because, in Rails, forms are highly coupled to ActiveModel. The setup would involve stubbing methods like `persisted?` and `model_name` and whatnot. Not to mention that views are usually dependent on other attributes too. Just the idea of building such a test stub makes me want a shower.
2. This is an oversimplified example that disregards other sources of complexity in this view. We know (Rails) views can get hairy. They may render a subtree of partials, or depend on other objects, or both. Some cases might demand much more setup than I needed here.

If I absolutely had to choose one of these solutions, I'd go with the view test because it's faster to run and cheaper to maintain. Trying to cover every possible scenario with acceptance tests in a recipe for turning your test suite into a slow and unmaintainable mess.

There's still one aspect of the situation that still bothers me though: testing the view still feels cumbersome and way too expensive compared to the feature complexity. So I'd like to reflect a little on this situation and the proposed solution.

### View logic

There's no denying that rendering templates (whenever it's erb, haml, jst, you-name-it) is procedural code. It **has** to be, otherwise they wouldn't support dynamic content. Their dynamic nature is where the fun (and usefulness) comes from.

It takes conditionals and loops to build anything dynamic. However, the amount of logic we put in those templates is up to us. I came up with a distinction that helps me a lot when deciding what goes in the view and what doesn't.

Take for example handlebars templates. In case you are not familiar, here's what Handlebars conditionals look like:

```
{% raw %}
{{#if isFriday}}
  It's friday! \o/
{{/if}}
{% endraw %}
```

Notice the conditional only checks for a boolean called `isFriday`. There are no conditional operators such as `weekday==6` or alike. In fact, Handlebars doesn't support conditional operators. The whole thing is based on a boolean variable or attribute. Loops are similar:

```
{% raw %}
{{#each user in users}}
  <li>{{user.name}}</li>
{{/each}}
{% endraw %}
```

Again, it's the simplest possible thing. It loops through a collection and inside the loop you can access attributes from each object inside the collection. There's no grouping, filtering or mapping, only a list of items through which we iterate.

This is what I call view logic: it's the smallest possible set of operations we can use in a template in order to keep it simple without losing its dynamic nature. In other words:

* Conditionals should be based on a simple question: "Should I render this or not?".
* Loops iterate through items, and each iteration yields an item to be rendered.

### Turning business logic into view logic

Quick recap of the whole story. I had this view code:

```erb
<% f.text_field :price, disabled: @plan.subscribers.any? %>
```

I thought I was happy with it when I found out that testing this change would be far from optimal. Then I reflected a bit on what view logic should look like. Now I can conclude this is not acceptable view logic. Why, you ask?

First of all, this is the implementation of a rule that came from a client, and it is, therefore, business logic. Clients defined the behavior, and they can (and will) change their mind at any moment. So I better be prepared.

Having a business rule implemented in our view is not being prepared. Even if it looks harmless, like the one from the example. For instance, imagine that in a month from now the clients says only super admins can change the plan's price. So we add another clause to the conditional, like `|| !current_user.super_admin?`. Hard to read, eh? Then the client decides the price can always be changed, but current users have to be grandfathered in. Now things got complicated. The sky is the limit.

Ok, so how we transform the business logic into view logic? Because the view still needs a conditional. I have an idea: presenters.

A presenter is just a plain old Ruby object that complements the model with presentation logic. I can extract the logic that defines if the price field should be enabled to a presenter like so:

```ruby
class PlanPresenter

  def initialize(plan)
    @plan = plan
  end

  def allow_price_change?
    @plan.subscribers.any?
  end
end
```

Now I just have to instantite `@plan_presenter` in the controller, and use it the view:

```erb
  <% f.text_field :price, disabled: @plan_presenter.allow_price_change? %>
```

The `allow_price_change?` is 100% compliant to my definitions of view logic: a boolean method that just answers the question *Should I render this?*. And since we're now dealing with a ruby object with a single responsibility, I can write fast and focused tests.

But wait: My presenter object is tested, but my view logic isn't. That's because, in general, I don't test view logic like this. The reasoning is: once the method providing the boolean is tested and I did some manual testing (and you should do manual testing too), it's really hard for something to go wrong. It's all too simple! So, unless this was something very critical, and I was really paranoid about it (think "Launch Missile" button), I'd probably add view testing. But for this case, I can confidently say I'm done with it.

### Closing thoughts

When dealing with view logic, think hard and then think hard again if you're not dealing with business logic in disguise.
For when you're in doubt:

* Is it code that expresses a requirement that came from the client? It's business logic.
* Is it a yes or no question that states whether something should be rendered? It's presentation logic.

Whenever possible find, isolate and add unit tests to business logic. Presenters are a great way to turn business rules into presentation logic.

Test the business rule, and avoid more expensive testing strategies. If absolutely necessary, prefer view testing over acceptance testing.

Questions? Thoughts? Comment on the HN thread:

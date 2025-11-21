---
layout: post
title:  "Using Page Object Pattern in rspec"
date:   2019-03-01 11:06:51
---

The page object pattern is a handy technique you can use to simplify the interaction with forms in a web app.

Take a look at this code in rspec.

{% highlight ruby %}
  it "can create posts" do
    login_as(user, :scope => :user)

    visit new_post_path
    within("#new_post") do
      fill_in 'post_title', with: "My super Blog title"
      fill_in 'post_input_content', with: "##loren loren\n ipsum ipsum"
    end
    click_button 'Publish'
    expect(page).to have_content 'Post was successfully created'
  end
{% endhighlight %}

Although capybara has a really great syntax, this can be troublesome with complex forms and prevents any code reuse.

This is the final code after refactored to use PO pattern.

{% highlight ruby %}
  it "can create posts -page object pattern" do
    new_post_form = NewPostForm.new
    new_post_form.login(user).visit_page.fill_in_with(
      post_title: "My super Blog title",
      post_input_content: "##loren loren\n ipsum ipsum"
    ).submit

    expect(page).to have_content 'Post was successfully created'
  end
{% endhighlight %}

The first thing is creating a spec/support folder.
{% highlight bash %}
  mkdir spec/support
  touch spec/support/new_post_form.rb
{% endhighlight %}

The class is a simple PORO 
{% highlight ruby %}
class NewPostForm
end
{% endhighlight %}

The first thing is to require devise, capybara and rails helpers we need.
{% highlight ruby %}
class NewPostForm
  include Capybara::DSL
  include FactoryBot::Syntax::Methods
  include Warden::Test::Helpers
  include Rails.application.routes.url_helpers
end
{% endhighlight %}

Then add the methods to the class, to chain methods just remember return self:
{% highlight ruby %}
class NewPostForm
  include Capybara::DSL
  include FactoryBot::Syntax::Methods
  include Warden::Test::Helpers
  include Rails.application.routes.url_helpers

  def login(user)
    login_as(user, :scope => :user)
    self
  end

  def visit_page
    visit new_post_path
    self
  end

  def fill_in_with(params={})
    within("#new_post") do
      fill_in 'post_title', with: params.fetch(:post_title)
      fill_in 'post_input_content', with: params.fetch(:post_input_content)
    end

    self
  end

  def submit
    click_button 'Publish'
  end
end
{% endhighlight %}






----------


links:
[article by Martin Fowler](https://martinfowler.com/bliki/PageObject.html)
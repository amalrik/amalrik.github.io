---
layout: post
title:  "Setting up your Rails 6, Rspec, Factorybot"
date:   2019-03-14 18:16:51
---

This is a base line guide to setup a solid rails project in no time.
I'm going to install rails, rspec and factory bot. I will use only the latest and the greatest versions on this guide. Lets do this.

### Install Rails
{% highlight bash %}
$ gem install rails --pre
$ rails _6.0.0.beta3_ new test_app -T -d postgresql
{% endhighlight %}

The first line will install rails 6 beta and the second will create a rails application, the options are for:
 - -T: skips default rails testing framework, I'm going to use Rspec.
 - -d postgresql: informs rails to use postgres as database. 

### Install Rspec
Change the Gemfile, adding the gems on the group development and test, like this:
{% highlight ruby %}
#Gemfile
group :development, :test do
  # Call 'byebug' anywhere in the code to stop execution and get a debugger console
  gem 'byebug', platforms: [:mri, :mingw, :x64_mingw]
  gem 'rspec-rails', '~> 3.8'
  gem 'factory_bot_rails'
  gem 'capybara'
end
{% endhighlight %}

Then run in the terminal:
{% highlight bash %}
$ bundle install
$ bin/rails db:create db:migrate
$ bin/rails g rspec:install
{% endhighlight %}

The output should look like:
![rspec generator output](/assets/img/settingup-rails6-rspec/output_install_rspec.png)


To complete the installation, we have to inform rspec to use capybara so add this line on spec_helper:
{% highlight ruby %}
#spec/spec_helper.rb
require 'capybara/rspec'
{% endhighlight %}

Also, I want configure rspec-rails to use chrome headless so we test javascript too.

{% highlight bash %}
$ mkdir spec/support
$ touch spec/support/capybara.rb
{% endhighlight %}

{% highlight ruby %}
#spec/support/capybara.rb
RSpec.configure do |config|
  config.before(:each, type: :system) do
    driven_by :rack_test
  end

  config.before(:each, type: :system, js: true) do
    driven_by :selenium_chrome_headless
  end
end
{% endhighlight %}

on spec/rails_helper.rb uncomment that line:
{% highlight ruby %}
#spec/rails_helper.rb
Dir[Rails.root.join('spec', 'support', '**', '*.rb')].each { |f| require f }
{% endhighlight %}


## Convenience configurations(optional)
This configurations add some convenience to our work-flow, feel free to change anything. 

### Better Rspec Output
add this line to .rspec file to get a more readable output:

{% highlight ruby %}
--format documentation
{% endhighlight %}

### Configure Generators
I'm basically disabling some tests we don't need by default so we can create by hand if it is appropriate.
{% highlight ruby %}
#config/application.rb
config.generators do |g|
  g.test_framework :rspec,
  fixtures: false,
  view_specs: false,
  helper_specs: false,
  routing_specs: false
end
{% endhighlight %}

### FactoryBot Helpers
Add this line on rails_helper.rb
{% highlight ruby %}
#spec/rails_helper.rb
config.include FactoryBot::Syntax::Methods
{% endhighlight %}

### Generate Binstubs
Finally run this command to generate binstubs for rspec so can run our specs with bin/rspec instead of the verbose bundle exec rspec:
{% highlight bash %}
$ bundle binstubs rspec-core
{% endhighlight %}



### Smoke Test
Lets see if everything works as expected. At moment as I write this rspec-rails hasn't a generator for system specs on the latest release. So I will do it by hand.

{% highlight bash %}
$ mkdir spec/system
$ touch spec/system/hello_system_spec.rb
{% endhighlight %}

{% highlight ruby %}
#/spec/system/hello_system_spec.rb
require "rails_helper"

RSpec.describe "Hello", type: :system do
  it 'it says hello' do
    visit "/hello/index"
    expect(page).to have_text("Hello#index")
  end
end
{% endhighlight %}

{% highlight bash %}
$ bin/rails g controller Hello index
{% endhighlight %}

To run the tests:

{% highlight bash %}
$ bundle exec rspec spec/system
{% endhighlight %}

if everything is fine you should see this screen on your terminal:
![smoke test output](/assets/img/settingup-rails6-rspec/output_smoke_test.png)


### FactoryBot
FactoryBot is already installed, but is good to test if everything works as expected, to do this lets create a simple model:
{% highlight bash %}
rails g model Post title:string content:text
rails db:migrate
{% endhighlight %}

Our simple model spec should look like:
{% highlight ruby %}
require 'rails_helper'

RSpec.describe Post, type: :model do
  it "has a valid factory" do
    expect(FactoryBot.build(:post)).to be_valid
  end
end
{% endhighlight %}

If its working we should see this:
![factorybot test output](/assets/img/settingup-rails6-rspec/factory-bot-output.png)

Great factorybot is working.


### Shoulda Matchers
As the final step lets install shoulda matchers

Add this to Gemfile
{% highlight ruby %}
#Gemfile
group :test do
  gem 'shoulda-matchers'
end
{% endhighlight %}

On terminal run:
{% highlight bash %}
$ bundle install
{% endhighlight %}

Add on spec/rails_helper.rb
{% highlight ruby %}
# spec/rails_helper.rb
Shoulda::Matchers.configure do |config|
  config.integrate do |with|
    with.test_framework :rspec
    with.library :rails
  end
end
class ActiveModel::SecurePassword::InstanceMethodsOnActivation; end;
{% endhighlight %}

The last line here is a temporary fix to an issue with rails 6 for more details [see](https://github.com/thoughtbot/shoulda-matchers/issues/1167)

alter spec/models/post_spec.rb to make it look like this:
{% highlight ruby %}
#spec/models/post_spec.rb
require 'rails_helper'

RSpec.describe Post, type: :model do
  it "has a valid factory" do
    expect(build(:post)).to be_valid
  end

  it { should validate_presence_of(:title) }
end
{% endhighlight %}

This test will fail unless we add this validation on post model:
{% highlight ruby %}
#app/models/post.rb
class Post < ApplicationRecord
  validates :title, presence: true
end
{% endhighlight %}

If we run rspec again this should be the output:
![shoulda test output](/assets/img/settingup-rails6-rspec/shoulda-output.png)

Thats conclude our setup. As a last step I will remove the post model used for testing the setup with:
{% highlight bash %}
$ rails destroy model Post
{% endhighlight %}

On part 2 we going to lint code with Rubocop and use SimpleCov to gather code coverage.

----------


additional links:

[rspec rails docs](https://bit.ly/2EVeJqm)

[rspec model test template](https://gist.github.com/kyletcarlson/6234923)

[shoulda matchers](https://github.com/thoughtbot/shoulda-matchers)

[factory bot cheatsheet](https://devhints.io/factory_bot)

[Getting started with chrome headless](https://developers.google.com/web/updates/2017/04/headless-chrome)

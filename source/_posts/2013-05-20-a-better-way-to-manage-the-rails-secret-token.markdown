---
layout: post
title: "A better way to manage the Rails secret token"
date: 2013-05-20 07:35
comments: true
categories: Heroku Security
---

**tl;dr** Don't hardcode secret tokens. Load them from the environment like this:

{% codeblock /config/initializers/secret_token.rb %}
# Be sure to restart your server when you modify this file.

# Your secret key for verifying the integrity of signed cookies.
# If you change this key, all old signed cookies will become invalid!
# Make sure the secret is at least 30 characters and all random,
# no regular words or you'll be exposed to dictionary attacks.
MyApp::Application.config.secret_token = if Rails.env.development? or Rails.env.test?
  ('x' * 30) # meets minimum requirement of 30 chars long
else
  ENV['SECRET_TOKEN']
end
{% endcodeblock %}

## Insecure Defaults

When you create a new Rails project, one of the files created will be `/config/initializers/secret_token.rb`. This file will look something like this:

{% codeblock /config/initializers/secret_token.rb %}
# Be sure to restart your server when you modify this file.

# Your secret key for verifying the integrity of signed cookies.
# If you change this key, all old signed cookies will become invalid!
# Make sure the secret is at least 30 characters and all random,
# no regular words or you'll be exposed to dictionary attacks.
MyApp::Application.config.secret_token = '3eb6db5a9026c547c72708438d496d942e976b252138db7e4e0ee5edd7539457d3ed0fa02ee5e7179420ce5290462018591adaf5f42adcf855da04877827def2'
{% endcodeblock %}

This token is used for…

{% blockquote The Twelve-Factor App http://www.12factor.net/config III. Config %}

An app’s config is everything that is likely to vary between deploys (staging, production, developer environments, etc). This includes:

Resource handles to the database, Memcached, and other backing services
Credentials to external services such as Amazon S3 or Twitter
Per-deploy values such as the canonical hostname for the deploy
Apps sometimes store config as constants in the code. This is a violation of twelve-factor, which requires strict separation of config from code. Config varies substantially across deploys, code does not.

A litmus test for whether an app has all config correctly factored out of the code is whether the codebase could be made open source at any moment, without compromising any credentials.

{% endblockquote %}

link to github search

In order to set the secret_token securely, we want to load it from the application's environment.

## Loading Rails Configuration from the Environment

The simplest method is to replace the hardcoded token with a reference to Ruby's `ENV`:

{% codeblock /config/initializers/secret_token.rb %}
# Be sure to restart your server when you modify this file.

# Your secret key for verifying the integrity of signed cookies.
# If you change this key, all old signed cookies will become invalid!
# Make sure the secret is at least 30 characters and all random,
# no regular words or you'll be exposed to dictionary attacks.
MyApp::Application.config.secret_token = ENV['SECRET_TOKEN']
{% endcodeblock %}

While this has the advantage of maintaining better [development/production parity](http://www.12factor.net/dev-prod-parity), it does require a slightly more complex development workflow (see below). If `ENV['SECRET_TOKEN']` isn't set in `development` or `test`, ActionDispatch will raise an exception like:

    ArgumentError (A secret is required to generate an integrity hash for cookie session data. Use config.secret_token = "some secret phrase of at least 30 characters"in config/initializers/secret_token.rb):
    

Alternatively, the tokens could be set in the respective `/config/environments/*.rb` files. This has the advantage of not requiring `SECRET_TOKEN` to be set in your `test` or `development` environment, but has the drawback of duplicating logic between the `development.rb` and `test.rb`.

{% codeblock /config/environments/development.rb %}
MyApp::Application.configure do
  # Settings specified here will take precedence over those in config/application.rb
  # ...
  config.secret_token = 'x' * 30
end
{% endcodeblock %}

{% codeblock /config/environments/test.rb %}
MyApp::Application.configure do
  # Settings specified here will take precedence over those in config/application.rb
  # ...
  config.secret_token = 'x' * 30
end
{% endcodeblock %}

{% codeblock /config/environments/production.rb %}
MyApp::Application.configure do
  # Settings specified here will take precedence over those in config/application.rb
  # ...
  config.secret_token = ENV['SECRET_TOKEN']
end
{% endcodeblock %}

Possibly the best option is to use `/config/initializers/secret_token.rb` to conditionally load the token from `ENV` depending on the `Rails.env`

{% codeblock /config/initializers/secret_token.rb %}
# Be sure to restart your server when you modify this file.

# Your secret key for verifying the integrity of signed cookies.
# If you change this key, all old signed cookies will become invalid!
# Make sure the secret is at least 30 characters and all random,
# no regular words or you'll be exposed to dictionary attacks.
MyApp::Application.config.secret_token = if Rails.env.development? or Rails.env.test?
  ('x' * 30) # meets minimum requirement of 30 chars long
else
  ENV['SECRET_TOKEN']
end
{% endcodeblock %}

 This also removes any pretense that the hard-coded token is secure. (For some reason, this token seems pretty popular on [Github](https://github.com/search?q=config.secret_token+%3D++&type=Code&ref=searchresults))

## Managing an Application's ENV

### Production

On Heroku, the ENV is managed by the `heroku` command:
    $ heroku config:set SECRET_TOKEN=3eb6db5a9026c547c72708438d496d942e976b252138db7e4e0ee5edd7539457d3ed0fa02ee5e7179420ce5290462018591adaf5f42adcf855da04877827def2
(or you could use something like the [HerokuConfigVars engine](/blog/2013/05/19/managing-heroku-config-vars-from-the-web/))

For [Phusion Passenger](https://www.phusionpassenger.com/support#documentation), the ENV is set in the webserver config. See this [blog post](http://blog.phusion.nl/2008/12/16/passing-environment-variables-to-ruby-from-phusion-passenger/) for more information.

For Unicorn, see [docs](http://unicorn.bogomips.org/unicorn_1.html#environment-variables).

### Development

[Foreman](https://github.com/ddollar/foreman) is a helpful gem for managing an application's environment variables in multiple contexts. Heroku uses this anyway, both for managing process and configuration. By default, it loads environment variables from the `.env` file.

{% codeblock .env %}
# .env should NOT be checked in to source control
SECRET_TOKEN=3eb6db5a9026c547c72708438d496d942e976b252138db7e4e0ee5edd7539457d3ed0fa02ee5e7179420ce5290462018591adaf5f42adcf855da04877827def2
{% endcodeblock %}

The key to leveraging this in your development workflow is the `foreman run` command.

> foreman run is used to run one-off commands using the same environment as your defined processes. 

    $ foreman run rails server
    $ foreman run rspec
    etc...



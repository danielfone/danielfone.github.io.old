---
layout: post
title: "A better way to manage the Rails secret token"
date: 2013-05-20 07:35
comments: true
categories: Heroku Security
---

**tl;dr** Don't hardcode secret tokens. Load them from the environment like this…

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
… and use the [Dotenv](https://github.com/bkeepers/dotenv) gem in production if needed.

## Insecure defaults

When you create a new Rails project, one of the files created will be `/config/initializers/secret_token.rb`. This file will look something like this:

{% codeblock /config/initializers/secret_token.rb %}
# Be sure to restart your server when you modify this file.

# Your secret key for verifying the integrity of signed cookies.
# If you change this key, all old signed cookies will become invalid!
# Make sure the secret is at least 30 characters and all random,
# no regular words or you'll be exposed to dictionary attacks.
MyApp::Application.config.secret_token = '3eb6db5a9026c547c72708438d496d942e976b252138db7e4e0ee5edd7539457d3ed0fa02ee5e7179420ce5290462018591adaf5f42adcf855da04877827def2'
{% endcodeblock %}

This token is used to sign cookies that the application sets. Without this, it's impossible to trust cookies that the browser sends, and hence difficult to rely on session based authentication.

## Why this is bad

Firstly, hard-coding configuration conflates config and code. Although this may not cause much pain in a very simple context, as the application and infrastructure grow this anti-pattern will make configuration increasingly complex and error prone.

{% blockquote The Twelve-Factor App http://www.12factor.net/config III. Config %}
An app’s config is everything that is likely to vary between deploys (staging, production, developer environments, etc). … Apps sometimes store config as constants in the code. This is a violation of twelve-factor, which requires strict separation of config from code. Config varies substantially across deploys, code does not.
{% endblockquote %}

### Security risk

More importantly though is the security implication. Knowing the secret token allows an attacker to trivially impersonate any user in the application.

The only system that *needs* to know the production secret token is the production infrastructure. In this case, the attack vector is limited to the production infrastructure, which is likely to be the most secure part of the infrastructure anyway.

By hardcoding the production secret token in the code base, the following attack vectors are opened:

* Every developer that has had access to the code base
* Every development workstation that has a local copy of the code
* The source control repository (whether private or 3rd-party e.g. Github)
* The continuous integration server
* Any 3rd-party services that have access to the source code, e.g. [Code Climate](https://codeclimate.com/) or [Gemnasium](https://gemnasium.com/)
* The people involved with all of the above services

If an attacker wishes to obtain the application's secret token, there are vastly more opportunities when the secret token is stored in the code.

The prevalence of this bad practice can be seen by searching [Github](https://github.com/search?l=Ruby&p=1&q=application.config.secret_token+%3D+%27&ref=searchbar&type=Code) or [Google](https://www.google.co.nz/search?q=secret_token.rb+-ENV+site%3Agithub.com). It's trivial to gain administrative access to many live applications simply by browsing those search results.

*Update: [Bryan Helmkamp](http://twitter.com/brynary) helpfully [notes below](#comment-902646816) that with these tokens it's actually possible to execute arbitrary code on the web server.*

## Loading Rails configuration from the environment

In order to set the secret token securely, we want to load it from the application's environment. The simplest method is to replace the hardcoded token with a reference to Ruby's `ENV`:

{% codeblock /config/initializers/secret_token.rb %}
# Be sure to restart your server when you modify this file.

# Your secret key for verifying the integrity of signed cookies.
# If you change this key, all old signed cookies will become invalid!
# Make sure the secret is at least 30 characters and all random,
# no regular words or you'll be exposed to dictionary attacks.
MyApp::Application.config.secret_token = ENV['SECRET_TOKEN']
{% endcodeblock %}

While this has the advantage of maintaining [development/production parity](http://www.12factor.net/dev-prod-parity), it can be inconvenient for simple apps. If `ENV['SECRET_TOKEN']` isn't set locally — for example in the development or testing workflow — ActionDispatch will raise an exception like:

    ArgumentError (A secret is required to generate an integrity hash for cookie session data. Use config.secret_token = "some secret phrase of at least 30 characters"in config/initializers/secret_token.rb):

One solution to this is managing a full set of environment variables within the development and test workflows. See below for more details on this.

Alternatively, a token could be hard-coded for the `development` and `test` environments, and loaded from the ENV in `production`.

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

This also removes any pretense that the hard-coded token is secure.

Occasionally, the following solution is used:

{% codeblock /config/initializers/secret_token.rb — Don't do this! %}
MyApp::Application.config.secret_token = ENV['SECRET_TOKEN'] || '3eb6db5a9026c547c72708438d496d942e976b252138db7e4e0ee5edd7539457d3ed0fa02ee5e7179420ce5290462018591adaf5f42adcf855da04877827def2'
{% endcodeblock %}

However, if `ENV['SECRET_TOKEN']` isn't set in production, **this will use the insecure token with no warning**.

## Managing an Application's ENV

[Dotenv](https://github.com/bkeepers/dotenv) is an excellent gem for managing an application's environment. Heroku's [foreman](https://devcenter.heroku.com/articles/procfile#setting-local-environment-variables) uses this behind the scenes. Install it with:

{% codeblock Gemfile %}
gem 'dotenv-rails'
{% endcodeblock %}

By default, it loads environment variables from the `.env` file. Simply create this file in the `RAILS_ROOT` on the production web server.

{% codeblock .env %}
# .env should NOT be checked in to source control
SECRET_TOKEN=3eb6db5a9026c547c72708438d496d942e976b252138db7e4e0ee5edd7539457d3ed0fa02ee5e7179420ce5290462018591adaf5f42adcf855da04877827def2
{% endcodeblock %}

As the application configuration and infrastructure grows more complex, the gem also provides a consistent method to manage configuration across multiple developers, CI, staging and production servers. [Brandon Keepers](https://github.com/bkeepers) wrote more on [the rationale for the gem](http://opensoul.org/blog/archives/2012/07/24/dotenv/).

### Heroku

On Heroku, the application's environment variables are managed from the `heroku` CLI:
    $ heroku config:set SECRET_TOKEN=3eb6db5a9026c547c72708438d496d942e976b252138db7e4e0ee5edd7539457d3ed0fa02ee5e7179420ce5290462018591adaf5f42adcf855da04877827def2
(or you could use something like the [HerokuConfigVars engine](/blog/2013/05/19/managing-heroku-config-vars-from-the-web/))

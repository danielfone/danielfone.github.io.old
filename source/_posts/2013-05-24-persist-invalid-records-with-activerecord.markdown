---
layout: post
title: "Persist Invalid Records with ActiveRecord"
date: 2013-05-24 12:38
comments: true
categories: ActiveRecord Validation
---

Some time ago, I had an unusual design brief for a Rails app:

> Even if a user submits an invalid record, we need to
>   (a) save a copy to the database, along with the validation errors, and
>   (b) re-render the form with the error messages as per the default Rails behavior.

The idea behind this design was that administrators could see the unsuccessful submissions and intervene to assist the users if necessary.

If anyone ever finds themselves in a similar situation, here's what I ended up doing:

I'm going to use a model called Subscription by way of example. This same pattern will work on any ActiveRecord model.

### Add a `record_errors` text column

We'll serialize the validation errors and store them in a field called `record_errors`.

{% codeblock /db/migrations/20130524012635_add_record_errors_to_subscriptions.rb %}
class AddRecordErrorsToSubscriptions < ActiveRecord::Migration
  def change
    add_column :subscriptions, :record_errors, :text
  end
end
{% endcodeblock %}

### Add an ActiveRecord extension to handle persistence

This module will catch validation failures and persist the record anyway. It's completely independent of the domain/business logic of the application so I like to store it in /lib. This is the main logic for achieving our required behaviour.

{% codeblock /lib/save_with_errors.rb %}
require 'active_record'
require 'active_support'

module SaveWithErrors

  def save_with_errors!(*args)
    save_without_errors! *args
  rescue ActiveRecord::RecordInvalid
    save_anyway
    raise # this re-raises the exception we just rescued
  end

  def save_with_errors(*args)
    save_without_errors *args or save_anyway
  end

  def self.included(receiver)
    receiver.serialize :record_errors, Hash
    receiver.alias_method_chain :save, :errors
    receiver.alias_method_chain :save!, :errors
  end

private

  def save_anyway
    dup.tap { |s| s.record_errors = errors.messages }.save(validate: false)
    false
  end

end
{% endcodeblock %}

### Include our ActiveRecord extension on our model

All we need to do is `include` the `SaveWithErrors` module into our ActiveRecord model. We could also require 'save_with_errors' in `config/application.rb`.

{% codeblock /app/models/subscription.rb %}
require 'save_with_errors'

class Subscription < ActiveRecord::Base
  include SaveWithErrors

  attr_accessible :email, :name, :token

  validate :valid_token

private

  def valid_token
    # something meaningful
    errors.add :token, 'is invalid' unless token == '12345'
  end

end
{% endcodeblock %}

### The Result

    Loading development environment (Rails 3.2.13)

    > Subscription.create! name: 'My Name', email: 'test@example.com'
      SQL (3.6ms)  INSERT INTO "subscriptions" ("created_at", "email", "name", "record_errors", "token", "updated_at") VALUES (?, ?, ?, ?, ?, ?)  [["created_at", Fri, 24 May 2013 01:55:07 UTC +00:00], ["email", "test@example.com"], ["name", "My Name"], ["record_errors", "--- !omap\n- :token:\n  - is invalid\n"], ["token", nil], ["updated_at", Fri, 24 May 2013 01:55:07 UTC +00:00]]
    
    ActiveRecord::RecordInvalid: Validation failed: Token is invalid
      [... backtrace ...]

Notice that a record has been inserted, but the exception that we'd expect from `create!` has still been raised. We can verify this by inspecting the last Subscription record:

    > y Subscription.last
      Subscription Load (0.3ms)  SELECT "subscriptions".* FROM "subscriptions" ORDER BY "subscriptions"."id" DESC LIMIT 1
    --- !ruby/object:Subscription
    attributes:
      id: 19
      name: My Name
      email: test@example.com
      token: 
      created_at: 2013-05-24 01:55:07.747035000 Z
      updated_at: 2013-05-24 01:55:07.747035000 Z
      record_errors: !omap
      - :token:
        - is invalid

Exactly what we want!




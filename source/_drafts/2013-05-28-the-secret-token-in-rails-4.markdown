---
layout: post
title: "The Secret Token in Rails 4"
date: 2013-05-28 06:21
comments: true
categories: 
---

Rails 4 has changed the way cookies are managed, and the secret token is now deprecated.


```ruby
# config/initializers/secret_token.rb
Myapp::Application.config.secret_token =    ENV['SECRET_TOKEN']
Myapp::Application.config.secret_key_base = ENV['SECRET_KEY_BASE']
```

    $ rake secret
    da726780ef84b454892e...
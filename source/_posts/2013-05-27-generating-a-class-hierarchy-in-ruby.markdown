---
layout: post
title: "Generating a Class Hierarchy in Ruby"
date: 2013-05-27 12:59
comments: true
categories: Metaprogramming
---

Here's a way to see what classes descend from a particular superclass in Ruby. You can use it to list all ActiveRecord models defined in your application, every implementation on a particular base class, or whatever else might be of interest.

```
> puts ClassHierarchy.new ActionController::Base
ActionController::Base
  ApplicationController
    UsersController
    UserSessionsController
    OrdersController
    ...
```

If you're using Rails, bear in mind that most of your classes won't be loaded by default in the development environment. You can get around this by manually requiring the files you're interested in:

```
Dir.glob File.join(Rails.root, 'app/models/**/*.rb'), &method(:require)
```

To use in your code, simply put `class_hierarchy.rb` somewhere that makes sense (perhaps /lib) and `require 'class_hierarchy'`.

{% gist 5654633 class_hierarchy.rb %}

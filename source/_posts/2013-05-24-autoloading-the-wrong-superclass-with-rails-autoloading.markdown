---
layout: post
title: "Autoloading The Wrong Superclass with Rails"
date: 2013-05-24 10:14
comments: true
categories: Autoloading Debugging
---

Autoloading in Ruby on Rails can cause all kinds of grief if you don't watch out. Here's a really quick tip to avoid hours of debugging down the track.

> Always use the full class namespace when specifying a superclass inside a module

For example:

```ruby
  module MyEngine
    class WidgetController < ApplicationController
      # BAD! ApplicationController is ambiguous
    end
  end
```

```ruby
  module MyEngine
    class WidgetController < MyEngine::ApplicationController
      # Good - inherits from the module specific ApplicationController
    end
  end
```

```ruby
  module MyEngine
    class WidgetController < ::ApplicationController
      # Good - inherits from the top-level ApplicationController
    end
  end
```

In the first example, the superclass for WidgetController **will depend on which classes are already loaded**. Ruby will look for the following classes in order:

  0. `MyEngine::ApplicationController`
  0. `::ApplicationController`

The trouble occurs when Rails has already autoloaded `::ApplicationController` and not `MyEngine::ApplicationController`. In this case, Rails will never need to autoload `MyEngine::ApplicationController` because `::ApplicationController` already matches the superclass.

Autoloading can occur in a different order between execution environments. For example, tests could all pass and while production system fails because of the different autoloading behavior.

The examples below demonstrate how the order of the class definitions affect which superclass is used if the superclass is ambiguous. When written out like this, the behavior is fairly obvious.

```ruby
class Base
  def initialize
    raise "This is the wrong super class!"
  end
end

module Widget
  class MyWidget < Base
    # Inherits from ::Base because Widget::Base isn't defined yet
  end
end

module Widget
  class Base
    def initialize
      puts "Success!"
    end
  end
end

```

    > Widget::MyWidget.new
    RuntimeError: This is the wrong super class!

```ruby
class Base
  def initialize
    raise "This is the wrong super class!"
  end
end

module Widget
  class Base
    def initialize
      puts "Success!"
    end
  end
end

module Widget
  class MyWidget < Base
    # Inherits from Widget::Base
  end
end

```

    > Widget::MyWidget.new
    Success!


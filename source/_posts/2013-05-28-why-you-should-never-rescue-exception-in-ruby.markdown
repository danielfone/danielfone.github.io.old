---
layout: post
title: "Why You Should Never Rescue Exception in Ruby"
date: 2013-05-28 04:27
comments: true
categories: Debugging
---

**tl;dr** **`rescue Exception => e` will turn your code into a brain eating zombie.**

The equivalent of `rescue` with an argument is `rescue => e` or `rescue StandardError => e`. Use these, or better still, figure out exactly what you're trying to rescue and use `rescue OneError, AnotherError => e`

## What's the deal?

A common pattern for rescuing exceptions in Ruby is:

```ruby
def do_some_job!
  # ... do something ...
  job_succeeded
rescue
  job_failed
end
```

This is *okay*, but when developers need to know the exception details, a horrible, *nasty*, **brain-devouring** pattern tends to emerge:

```ruby
def do_some_job!
  # ... do something ...
  job_succeeded
rescue Exception => e
  job_failed e
end
```

I have been caught out by that code on at least three separate occasions. Twice when I wrote it. I write this post in the hope that I (and perhaps others) will finally wise up about exception handling and that my fingers will never, ever type that code again.

*(Just to confirm this is a actually bad practice, here's [~200k results](https://github.com/search?l=ruby&o=asc&p=1&q=%22rescue+Exception+=%3E+%22&ref=searchresults&type=Code) for `rescue Exception => ` on Github)*

## What is this I don't even…

`Exception` is the root of the exception class hierarchy in Ruby. Everything from signal handling to memory errors will raise a subclass of Exception. Here's the full list of exceptions from ruby-core that we'll inadvertently rescue when rescuing Exception.

```ruby
SystemStackError
NoMemoryError
SecurityError
ScriptError
  NotImplementedError
  LoadError
    Gem::LoadError
  SyntaxError
SignalException
  Interrupt
SystemExit
  Gem::SystemExitException
```

Do you really want to rescue a `NoMemoryError` and send an email saying the job failed?!? Good luck with that.

## Better: Rescue StandardError

`rescue => e` is shorthand for `rescue StandardError => e` and is almost certainly the broadest type of Exception that we want to rescue. In almost every circumstance, we can replace `rescue Exception => e` with `rescue => e` and be better off for it. The only time when that's *not* a good idea is for code that's doing some kind of exception logging/reporting/management. In those rare cases, it's possible we'll want to rescue non-StandardErrors — but we still need to think pretty hard about what happens after we've rescued them.

Most of the time though, we don't even want to rescue StandardError!

## More Self-Inflicted Fail

Imagine a scenario where we're connecting to a 3rd-party API in our application. For example, we want our users to upload their cat photos to twitfaceagram. We definitely want to handle the scenarios where the connection times out, or the DNS fails to resolve, or the API returns bogus data. In these circumstances, we want to present a friendly message to the user that the application couldn't connect to the remote server.

```ruby
def upload_to_twitfaceagram
  # ... do something ...
rescue => e
  flash[:error] = "The internet broke"
end
```

Most of the time, this code will do what we expect. Something out of our control will go wrong, and it's appropriate to present the user with a friendly message. However, there's a major gotcha with this code: we're still rescuing many exceptions we're not aware of.

Here's an abridged list of StandardErrors defined in ruby-core 2.0.0 (1.9 is not materially different):

```ruby
StandardError
  FiberError
  ThreadError
  IndexError
    StopIteration
    KeyError
  Math::DomainError
  LocalJumpError
  IOError
    EOFError
  EncodingError
    Encoding::ConverterNotFoundError
    Encoding::InvalidByteSequenceError
    Encoding::UndefinedConversionError
    Encoding::CompatibilityError
  RegexpError
  SystemCallError
    Errno::ERPCMISMATCH
    # ... lots of system call errors ...
    Errno::NOERROR # errrr.... what?
  RangeError
    FloatDomainError
  ZeroDivisionError
  RuntimeError
    Gem::Exception
      # ... lots of gem errors ...
  NameError
    NoMethodError
  ArgumentError
    Gem::Requirement::BadRequirementError
  TypeError
```

In a fresh Rails 3.2.13 application, there are **[375 StandardErrors defined](https://gist.github.com/danielfone/5654600)**.

Now let's say we're refactoring the API integration and we make a typo with a method name. What's going to happen?

If we've wrapped the entire process in a `rescue => e` (which is rescuing StandardError) the NoMethodError is going to be swallowed and our graceful error handling code is going to be run instead. When we run our well written tests, they'll fail. But rather than raising a straight-forward NoMethodError, it'll look like there was an gracefully handled connectivity problem.

Now *that* is going to take some debugging.

If our tests are poorly written there'll be no exception and perhaps the tests will just pass. Granted, in production our users won't be seeing ugly 500 errors, but they sure won't be uploading their cat photos either.

## Best: Rescue Specific Exceptions

Every part of our code is *qualified* to rescue from certain exceptional circumstances. If we want to catch connectivity problems in an API integration, our code will be qualified to rescue from a [long list](http://tammersaleh.com/posts/rescuing-net-http-exceptions) of Net related exceptions. It is *not* qualified to rescue from an ArgumentError, which is a code-time problem and not a run-time problem!

Every time we write a rescue, we need to think hard about what exceptions this code is actually qualified to handle.

In the case of HTTP, we can make it easier on ourselves and use a wrapper like [faraday](https://github.com/lostisland/faraday). In this case we'll have a [much shorter list](https://github.com/lostisland/faraday/blob/master/lib/faraday/error.rb) of possible exceptions to rescue.

## So…

… if you encounter `rescue Exception => e` in an existing codebase, you can almost certainly replace it with `rescue => e`.

… if you find yourself about to type `rescue Exception => e`, slap yourself in the face, figure out exactly what exceptions you're dealing with and rescue those instead. 

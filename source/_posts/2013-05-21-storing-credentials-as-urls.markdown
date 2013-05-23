---
layout: post
title: "Storing Credentials as URLs"
date: 2013-05-21 17:30
comments: true
categories: 
published: false
---

```
$ heroku config
=== my-app Config Vars
MANDRILL_PASSWORD:            xxxxxxxx-a19b-4278-a67c-c9d83977bc24
MANDRILL_USERNAME:            myaccount
```

becomes:

    $ heroku config:set MANDRILL_URI='http://myaccount:xxxxxxxx-a19b-4278-a67c-c9d83977bc24@mandrill.com'

     > y URI.parse MANDRILL_URI
    --- !ruby/object:URI::HTTP
    password: xxxxxxxx-a19b-4278-a67c-c9d83977bc24
    path: ''
    user: myaccount
    host: mandrill.com
    scheme: http
    parser: 
    fragment: 
    registry: 
    opaque: 
    query: 
    port: 80
     => nil 

```
$ heroku config
=== my-app Config Vars
SMTP_ADDRESS:                 smtp.mailgun.org
SMTP_PASSWORD:                5qtss3urhj12
SMTP_PORT:                    587
SMTP_USERNAME:                postmaster@shipping.etailer.co.nz
```

becomes:

  SMTP_URI="smtp://postmaster%40shipping.etailer.co.nz:5qtss3urhj12@smtp.mailgun.org:587"



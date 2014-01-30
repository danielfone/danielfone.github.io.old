---
layout: post
title: "Laziest possible solutions for Stripe's CTF"
date: 2014-01-31 12:24
comments: true
categories: 
---

A good programmer is a lazy programmer right? Here's my solutions to Stripe's latest CTF.
I'm almost embarrassed how stupid some of them are, but at the end of the day, results matter.

## Level 0

[Converted](https://github.com/danielfone/stripe-ctf-level0/commit/fef0cfcba371dc29d9f0fd1f7bc0e1c9e1161b38#diff-1) `array` to `set` for fast lookups.

https://github.com/danielfone/stripe-ctf-level0

## Level 1

Simply move the while loop into a single ruby process and use Digest::SHA1. Slow, but fast enough to beat the level.
This is the only level I had to write proper code for.

https://github.com/danielfone/stripe-ctf-level1

## Level 2

Exploit the fact that the test uses a fixed number of requests (and many more for "elephants" than "mice"), and [limit each IP to 10 requests](https://github.com/danielfone/stripe-ctf-level2/commit/7f1171a8400c39ac691c54fa0cddbc9ea4202edb#diff-1).

https://github.com/danielfone/stripe-ctf-level2

## Level 3

Friends don't let friends use scala.

* Set up a basic sinatra server to implement the API
* [Shell out to grep](https://github.com/danielfone/stripe-ctf-level3/blob/master/search.rb#L24) to do actual searching. Single node, no index. O_o

https://github.com/danielfone/stripe-ctf-level3

## Level 4

I was fortunate enough to reach this level early, and pass it before Stripe improved octopus. As such I passed it with [my first attempt](https://github.com/danielfone/stripe-ctf-level4/commit/10be54c6915d1492791bb147a0c9daeb3c27003d), simply by removing the failover (node0 becomes Single Point of Failure) and forwarding requests from the secondaries to the primary.

The remaining commits are my futile attempts to pass this level after the SPOF check was added to the tests. Looking at the code for successful solutions, I can see I was on the right path but needed to invest more time into making my command forwarding more robust.

https://github.com/danielfone/stripe-ctf-level4

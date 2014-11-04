---
layout: post
title:  "Dependency Injection and Gamification - Part 2"
date:   2014-03-21 14:46:00 +0500
author: Steve Shi
comments: true
---

##Dependency Injection - Part 2

Continue on the topic of Dependency Injection, let’s take a look at another repo:

https://github.com/jaredstehler/dropwizard-quartz

Not only is it a practical example of using Guice and Quartz to schedule jobs using cron syntax in Dropwizard, and it also shows how logging, managed objects , and testing with mockito work in Dropwizard,
It uses Guice to create dependency injection into a scheduler manager at run time using the @inject annotation, and then dynamically injects any scheduler classes annotated with the @Scheduled type.

Unlike Spring, Guice does not require an XML. I find this post explaining it very well the difference between Spring and Guice: The fundamental difference between Guice and Spring’s approaches to DI can probably be summed up in one sentence: Spring wires classes together by bean, whereas Guice wires classes together by type.
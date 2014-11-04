---
layout: post
title:  "In Strem Event Processing Architecture"
date:   2014-03-17 14:46:00 +0500
author: Steve Shi
comments: true
---

The in stream event processing model allows a single message to be processed in multiple downstream stations. There is a very good blog from http://highlyscalable.wordpress.com/2013/08/20/in-stream-big-data-processing/ demonstrating an in stream event processing system at massive scale. At its core, every in stream event processing system requires three components: <!--more--> 

1. a fault tolerant pub sub system, so that message can be broadcasted downstream without the upstream station being aware of the receivers, and that transmission of message is fault tolerant. A very nice feature to have would be to have the ability to play back messages. At very high scale, Kafka is an option. On the other end of the scale, SNS/SQS provides a simple solution. Relevant readings:

http://stackoverflow.com/questions/16449126/kafka-or-sns-or-something-else

On SNS/SQS:  
<iframe width="420" height="315" src="https://www.youtube.com/embed/zwLC5xmCZUs" frameborder="0" allowfullscreen></iframe>

2. distributed message processing and computation services. At very high scale and complexity, one can consider Storm, http://www.slideshare.net/gschmutz/kafka-andstromeventprocessinginrealtime; on a smaller scale, one can implement their own distributed event processor classes, however, it is important these processors are structured so that their responsibilities are clear. for example, in a gamification application, a data processor may handle incoming device data persistence; an action processor may process the immediate user actions such as checking off a user’s to do list items, or award some points for having performed the action; an achievement processor is called by an index operation to check if aggregated action results may have earned the user a badge; and a reward processor is responsible for financial transactions when certain levels are attained by the user in the system; etc. 3. persistence. Cassandra is used for persistence because it is flexible as a big data analytics engine, allowing additional analytics applications like Shark and Hive to be added. http://shark.cs.berkeley.edu/


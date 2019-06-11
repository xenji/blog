---
title: Stateful web applications pt.2
tags: ['web']
---
This is a followup of my first article about stateful web applications, which you can find [here](/tech/2012/6/18/stateful-webapps-pt1), you may want to read it first.

## Push state - poll state

We are still talking about stateful web applications. In my previous article I’ve talked about a challenge - response method to keep client and server side in sync. This method had some drawbacks but was simple to achieve.

The push state - poll state method is more complex concerning the setup, but way simple when it comes to the development part. Instead of using a single, serial channel of communication, we break up things and use two separate channels for our needs. This simple change gives us the possibility to acquire more change state requests as we could to with the sequential version (meaning the challenge - response method).

## The setup

You will need something that accepts a high number of concurrent request. I would not recommend an Apache httpd for that. I’ve tried node.js and it worked. (nginx might fit as well, depending on your software stack.) This server, let’s call it “change-bucket”, gets all the change state requests from all clients via PUT request. It responds with some 2xx state, maybe 204 or 201\. This is just the info, that the put-request reached the server. As you might imagine, this might lead to an eventual consistency state. The change-bucket passed the change request to a shared queue, which could be a redis server or something like ApacheMQ, RabbitMQ, or similar. The queue server maintains one list per application (of which multiple per user can exists side by side). By the way: redis is awsome for such a job, because it supports list operations natively - even blocking ones - and produces less overhead than a real message queue. The last thing you need is something like memcache. You can use redis or a real memcache server - depending on your setup. The key is: it must be fast and at least persistent as long as it has a power supply.

Now, that we have all change requests of the user in a queue, we need to get them out and change the server-side application state and reflect those changes to the client. This is also a point in this setup, where you can decide which way to go.

We will now switch perspectives and take a view from the client side now. This helps us to identifiy the point where those two ways will come together again.

## The client

The client need a poll mechanism. You know, something like setTimeout() with a propper tail recursion. I would not recommend using call(), as it is of lower priority in the most JavaScript Engines. The intervall depends on your technology stack, but I recommend to make it possible to set it to 250ms or less. This enables you to fake a real-time feeling on the customer’s side. This poll request must go somewhere and it needs to receive something. What does it receive? The full application state, where does it go? Now we reached the other side of our two way split.

## Path 1 - blocking list operations

This way is of moderate complexity and lacks of scalable performance. You will need only one additional piece for your architecture puzzle. You need an endpoint for the client to ask for the latest application state and you will need to make this endpoint fetch the stack of change requests from redis. This fetch operation must be blocking to prevent a concurrency issue. The concurrency issue comes from requesting the same endpoint twice. The first requests has not persisted the new application state yet and the second request loads the old state from the memcache server. The last one who persists the state wins. In this case we would get a not recoverable, inconsistent state. Another problem: you will need to solve the unsuccessful put-requests in the change-bucket with e.g. a retry loop in the client.

## Path2 - everything non-blocking

In this setup you will need another, daemon like instance to solve the problem of concurrency. We need to decouple the polling process from persisting the new state. We could do this by a cronjob or a node.js server or something completly different. The important aspect is, that this cronjob just takes data from the queue and merges it into the new state. The polling mechanism may see the same state twice, but that is OK as far as we got rid of our concurrency issues.

## Conclusion

We’ve build a circle, a round-trip for our application state. You might now have the feeling, that this is a shit load of overhead regarding the task of “just” persisting an application state, but that is OK. It is a rather complex setup with it’s own issues and I must admit, that it works only in theory at the moment.

## Future

One of the big advantages of this setup is the possibility to scale it. If the polling mechnism reached it’s scale, you can easily exchange the polling with persistent socket connections. If you do so, you might not even need a single change-bucket, but you could use the same socket to do both parts of the communication over a single architectual piece.
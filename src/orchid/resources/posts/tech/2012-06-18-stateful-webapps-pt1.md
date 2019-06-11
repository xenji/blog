---
title: Stateful web applications pt.1
tags: ['web']
---
## Preface
In modern web application we mostly use two sided setups. One is a client - which will be a browser in this case - the other is some sort of server with some sort of application. The languages used on the server side do not matter at all, but the client side language will be JavaScript.

## It is all about state
The application has a state. This might mean some sort of dataset which the client works on or something which is brought to the client by the server and is fully or partly persisted to enable the client further requests in the same data context. This topic is highly connected to caching those datasets and working on those cached results rather than working on live datasets.

HTTP is stateless, which means that we must rely on other techniques to synchronize the server and the client state. You might call for (web-)sockets now, but not all companies are able to cope with a change of their infrastructure to use sockets for the number of users that are concurrently browsing the app.

## Challenge - Response
One way of getting this done is a challenge - response communication protocol between the server and the client. The client always adds a token to identify itself on the server side. In addition, when the client sends a request it adds a ticket id to it. The server processes the response either in realtime or asynchronous. When doing it in realtime, the server responds directly to the client request. The async request must trigger polling mechanism where the server answers at some time with the requested ticket id. This type of communication works well, as long as your requests do not depend on each other. If you need state changes in a certain order, you need to add some kind of counter to it. The server needs to keep up with the counter and cannot process counter number 5 before counter number 4 was processed successfully.

This might lead to a deadlock situation, where counter 4 is lost in space for some reason and the server got stuck at holding back anything above counter 4.

Another drawback might be the danger to slip into some sort of interpretation of the client state. The server gives you a bunch of values to set some of the states that were asked for in the last response. You might be forced to start guessing the state of former components based on the partial response.

Too abstract? Ok, here is an example. You open some sort of info box and it’s content must be loaded from a server via an ajax call. The response comes back and you display your text. This box is one of three in a left column UI element. The UI concept says, that not more than one box should be open at the same time. OK, what happens next is common sense at the moment, but from my point of view it is a great danger. The client decides, that, on opening another box, the actual box has to be closed. This state never reaches the server. The client interpreted, that he must close it, because another box want be gain the state “open”.

Lets get one step further. We have another, dependent view element that also requires a server request. The answer to that request could possibly open one of the three boxes. As the server’s answer enforces the client to do that, the actual state of the client (which is unknown to the server) gets lost. The client need to interpred the result again and concludes, that the active open box must be closed to achieve the requirements of the former server response.

Where is the drawback now, you might ask. Try to let the client send a server request based on data that cannot be interpreted correctly be the client:

  * You have a entity to show via an id.
  * You need to show meta data based on the id.
  * You have a default setting on the root level of your application (“/”)
  * You do not want to show the meta-data to the default id, but another one

If you just deliver the id, the client is not able to decide if it’s default or not. The amount of “please client: if default do that, if not do this” raises and will kill you/your app some day.

The next article will be about using a “Push / Poll State Sync” approach.
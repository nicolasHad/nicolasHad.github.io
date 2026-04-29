---
layout: post
title: "Building asynchronous, non blocking and scalable applications with Celery and Redis: A case study"
date: 2026-02-01 09:00:00 +0300
categories: [Software development, Backend, API design]
tags: []
description: "Designing and developing scalable APIs to handle long running tasks with grace."
---

*Modern web applications routinely serve thousands of concurrent users. A single architectural decision can make the difference between humming along under load and grinding to a halt: how the system deals with slow, long-running work. In a blocking architecture, a server worker only handles one request at a time. This means that any operation that takes a while — encoding a video, generating a report, sending a batch of emails — makes every other user behind it wait. This is turned on its head by a non-blocking architecture: heavy work is delegated to background workers and the main process is freed almost instantly to keep serving incoming requests. That difference sounds academic until you see it in practice.*

## A simplified example

Imagine this: someone on YouTube is excited to watch the music video for their favorite band's new song after a fifteen-year break. Just ten seconds in, someone else on the other side of the world starts uploading their nine-hour, no-death Dark Souls 3 speedrun. As soon as the upload begins, the whole platform freezes for everyone. Videos stop playing, comments won't post, search doesn't work, and recommendations won't load. The first user, stuck in the first seconds of the song, just sees a buffering icon—and keeps waiting, because nothing will work until all nine hours of footage of the other user's upload finish uploading. This is a prime example of an application that does not follow a non-blocking architecture to ensure that smooth user-UI interactions that are not affected by the load of backend processes running on the server side of the application.

In modern applications, one can take this challenge one step further and ask the following question. 

*"My application not only needs to serve users without blocking or freezing when a background process is running, but is also needs to robustly accommodate a large number of such processes as efficiently as possible, while the end users can freely navigate the UI wihout any obstructions."*

Therefore, the task is now updated to not only ensuring the non-blocking behaviour of our app, but also how do we accommodate a large number of such non-blocking processes in a manageable way. 

Typically, an application based on a non-blocking architecture operates in the following manner:
- A user will initiate a long running process via the UI.
- The server does 2 things:
   - Registers the process to a broker who will in turn hand the job of executing this process to one of many available workers.
   - Lets the user know that their request was received and handed over to a worker for execution (instant acknowledgment).
- The user proceeds to do something else, while being able to go back and check on the progress of the process they submitted earlier.

But what exactly is a broker and a worker?
A **worker** is a separate process whose entire job is to execute long-running tasks. A **message broker** is a system that is able to take in the requests coming from the server and distribute them to different workers for execution.
The message broker is used to implement a **task queue** which is simply the component which is in charge of temporarily registering processes to be executed until workers can pick them up for execution.

This is precisely where Celery and Redis enter the story. Celery, is a distributed task queue library for Python and Redis is a popular broker choice.
Specifically, Celery provides the worker processes and the machinery for defining, dispatching, and executing background jobs. Redis, in this setup, plays the important role of the message broker: an in-memory data store fast enough to act as the inbox where Celery tasks are queued, claimed, and tracked. When the example Youtube server receives the nine-hour upload from above, it doesn't process the video itself, it creates a Celery task, Redis adds it into the queue, and immediately returns a response to the uploader. A Celery worker running in its own process (and optionally on its own machine) and sees the new task in Redis, picks it up, and begins the long work of saving and encoding the file. Meanwhile, the web server is free, the music video keeps playing, and our first user never knows any of this happened.

---

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

Modern programming languages and frameworks offer several tools and libraries for developing non-blocking applications with asynchronous process execution. 

---

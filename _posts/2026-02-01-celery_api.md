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

![alt text](/_posts/images/celery_task_queue_flow.drawio.svg)

The diagram above provides a simple representation of the non-blocking workflow achieved with Celery. The duplication of the User/UI and Web server entities is not literal, it signifies their state at two different times. The top row at time when the user asks for a task to be executed and the bottom row when the user asks for information about the completion status of the task, with the option of retrieving the result if the task has finished.

---

## Implementing the workflow with Celery

Now that we've outlined the architecture, let's see what it actually looks like in code. A Celery setup has three moving parts that we need to wire together: the Celery application (which has information about the broker and the registered tasks), the task definitions (these are the Python functions that the workers will execute), and the client-side component (the web server code that hands work off to the queue). We'll build each one in turn, using the long video upload from our example as the task to be processed.

1. **The Celery application**

   The Celery app is the central object that ties everything together. It tells Celery which broker we want to use and where it lives, where to store results, and which functions are available as background tasks. In a typical project, it sits in its own module (something like tasks.py) so that both the web server and the worker process can import it.

   ```python
   # tasks.py
   from celery import Celery

   celery_app = Celery(
      <APP_NAME>,
      broker="redis://localhost:6379/0",
      backend="redis://localhost:6379/1",
   )
   ```

   - redis://localhost:6379/0 is where the broker lives, it will be used to add tasks to the queue.
   - redis://localhost:6379/1 is the backend, which is where workers will write the outcome of a finished task so the web server can later retrieve it.


2. **Defining a task**
   
   A task is just a Python function. It is exactly the process that a worker is supposed to execute in the background. It could be any kind of algorithm - in our Youtube case, this could be the actual processing and upload of a video to a database - and it is decorated with @celery_app.task. The decorator registers the function with the Celery app, which means workers can look at this module and know what operations they have to run if they receive this task from the broker.

   ```python
   # tasks.py (continued)
   import time

   @celery_app.task
   def process_video_upload(video_id: str, file_path: str) -> dict:
      """Encode an uploaded video and generate thumbnails."""
      # In a real system: read the file, transcode to multiple
      # resolutions, generate thumbnails, write to object storage…
      time.sleep(60)  # stand-in for the actual heavy work
      return {
         "video_id": video_id,
         "status": "ready",
         "duration_seconds": 32400,
      }

   ```

   As you can see, this is an ordindary Python function. The key to this is the Celery decorator which will let us call this function with an additional .delay() call which will tell Celery to send the call to the queue instead of executing it locally.

3. **Dispatch task from the web server**

   Now, we want to build the bridge between the user and Celery. In our server, we define a relevant endpoint which when requested, will initialise the task to be executed.

   ```python
   # views.py (Flask-style for brevity; the same pattern works in Django, FastAPI, etc.)
   from flask import Flask, request, jsonify
   from tasks import process_video_upload

   app = Flask(__name__)

   @app.post("/upload_video")
   def upload_video():
      video_id = save_uploaded_file(request.files["video"])
      task = process_video_upload.delay(video_id, f"/uploads/{video_id}.mp4")
      return jsonify({"task_id": task.id, "video_id": video_id}), 202
   
   ```

   .delay() does the following under the hood: (a) it serializes the function name and arguments, (b) pushes the resulting message onto the broker, and (c) returns an AsyncResult object almost instantly. From that object we can grab task.id (a UUID generated by Celery) and hand it back to the client with HTTP 202 ("Accepted"), which is the standard response code for the server saying to the client that "I've received your request and started working on it, but I'm not done yet".

   The web server is now free. It has spent milliseconds on this request, regardless of whether the upload was a three-minute music video or a nine-hour speedrun.

4. **Running the worker**

   In order for jobs to be picked up and be processed, we need active workers that will watch the queue and pick up these jobs.



---
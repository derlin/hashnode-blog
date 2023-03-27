---
title: "FastAPI + Celery = ♥"
seoDescription: "Interested in Python FastAPI? Wondering how to execute long-running tasks in the background? Let's discover FastAPI+Celery through a practical use case."
datePublished: Mon Mar 27 2023 12:00:39 GMT+0000 (Coordinated Universal Time)
cuid: clfqs2c0r019t00nv753r0eqj
slug: introduction-to-fastapi-and-celery
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1679598158400/393fcf19-2e4e-46e0-9c33-f0c1a53e871a.png
tags: tutorial, python3, fastapi, python-beginner

---

## The use case

I learned about [FastAPI](https://fastapi.tiangolo.com/) and [Celery](https://docs.celeryq.dev/en/stable/getting-started/introduction.html) when confronted with a simple yet interesting use case

![use case](https://derlin.github.io/introduction-to-fastapi-and-celery/assets/00-goal.excalidraw.png align="left")

I had a Jupyter Notebook that connected to a database, ran some heavy processing on the data (using machine learning and everything), and saved aggregated data back to the database.

Since notebooks are great for developing, the requirement was to **keep using notebooks for development but to be able to trigger the processing from an API call**. The notebook should **never be executed twice in parallel** though. In other words, the API should return an error if the notebook were already being executed. Note that the notebook would be provided once at deployment time: it won't change during the lifecycle of the app.

---

## The implementation

I was initially planning to use a simple [Flask](https://flask.palletsprojects.com/) app but soon got into trouble. How can I (1) run the notebook in a background thread and (2) restrict its execution to one at a time?

This is how I discovered FastAPI and Celery. I implemented an MLP (***M****inimum* ***L****ovable* ***P****roduct*) based on those technologies available on GitHub.

⮕ ✨✨ [github.com/derlin/fastapi-notebook-runner](http://github.com/derlin/fastapi-notebook-runner) ✨✨

---

## The tutorial

The use case was perfect for learning. This is why I cooked a complete tutorial based on it, along with schemas and explanations. The [tutorial repository](https://github.com/derlin/introduction-to-fastapi-and-celery) can be used as a base to follow along. Not only will you learn about FastAPI and Celery, but also [Poetry](https://python-poetry.org/), [ruff](https://beta.ruff.rs/docs/), and other nice tips and tricks.

⮕ ✨✨ [derlin.github.io/introduction-to-fastapi-and-celery](http://derlin.github.io/introduction-to-fastapi-and-celery) ✨✨

Jump to the main sections:

* [Introduction](https://derlin.github.io/introduction-to-fastapi-and-celery/)
    
* [Poetry](https://derlin.github.io/introduction-to-fastapi-and-celery/01-poetry)
    
* [FastAPI](https://derlin.github.io/introduction-to-fastapi-and-celery/02-fastapi)
    
* [Celery](https://derlin.github.io/introduction-to-fastapi-and-celery/03-celery/)
    
* [Executing Notebooks](https://derlin.github.io/introduction-to-fastapi-and-celery/04-notebook/)
    
* [Finishing touches](https://derlin.github.io/introduction-to-fastapi-and-celery/05-more/)
    

---

I used the above website as a base for a talk at the [GDG Fribourg](https://gdg.community.dev/gdg-fribourg/) and figured some of you could also benefit from it. Don't forget to leave a ⭐!
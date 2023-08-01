---
title: "How to auto reload & debug Django and Django Celery workers running in Docker (VS Code)"
seoDescription: "Learn how to debug and auto-reload Django or Django celery workers (running in docker) using debugpy, watchdog, and django.utils.autoreload."
datePublished: Tue Aug 01 2023 11:58:25 GMT+0000 (Coordinated Universal Time)
cuid: clks8wnas002k09l52vbxct8b
slug: auto-reload-plus-debug-django-and-django-celery-workers
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1690874048762/ccf00d28-596f-4105-9945-b1a8b11b8f0f.jpeg
tags: python, django, developer, debugging

---

I run Django projects in Docker containers and use Visual Studio Code as my IDE. In this article, I share how I debug and auto-reload both Django and Celery workers. The solutions are based on `debugpy`, `watchdog`, and `django.utils.autoreload`.

---

In this article:

* [Example docker-compose](#heading-example-docker-compose)
    
* [🔄🐞 Debugging a Django app running in Docker](#heading-debugging-a-django-app-running-in-docker)
    
* [🐞 Debugging a Django celery worker running in Docker (no auto reload)](#heading-debugging-a-django-celery-worker-running-in-docker-no-auto-reload)
    
* [🔄 Auto-reloading a Django celery worker running in Docker (no debug)](#heading-auto-reloading-a-django-celery-worker-running-in-docker-no-debug)
    
* [🔄🐞 Debugging a Django celery worker running in Docker with auto-reload](#heading-debugging-a-django-celery-worker-running-in-docker-with-auto-reload)
    

---

## Example docker-compose

To better understand the rest of the post, let's assume your `docker-compose.yml` looks similar to this:

```yaml
services:
  web: # Django App
    build: .
    command: ./manage.py runserver 0.0.0.0:80
    volumes:
      - .:/app # mount the code inside the container
    links:
      - postgres

  worker: # Celery Worker
    build: .
    command: >-
         celery -A my.package.worker worker 
         -l info --concurrency=6 --queues a,b
    volumes:
      - .:/app
    links:
      - postgres

  postgres: # Database
    image: postgres:15-alpine
    ports:
      - '5432'
    volumes:
      - .data/postgres:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: ...
      POSTGRES_USER: ...
      POSTGRES_PASSWORD: ...
```

## 🔄🐞 Debugging a Django app running in Docker

Live reload is enabled when running `manage.py runserver`, but what about debugging?

The easiest way to make breakpoints work is to install `debugpy` in the container, and open a remote port for debugging that vscode can connect to.

Since the docker-compose file and the `Dockerfile` are usually version controlled, let's use a `docker-compose.override.yml` for the debug setup. From [docker compose](https://docs.docker.com/compose/reference/)'s documentation:

> If you don’t provide the `-f` flag on the command line, Compose traverses the working directory and its parent directories looking for a `docker-compose.yml` and a `docker-compose.override.yml` file. \[...\] If both files are present on the same directory level, Compose combines the two files into a single configuration.
> 
> (see also [Merge and override](https://docs.docker.com/compose/compose-file/13-merge/)).

Create a `docker-compose.override.yml` alongside your `docker-compose.yml` with the following:

```yaml
# in docker-compose.override.yml
services:
  web:
     command: >- 
        sh -c "pip install debugpy && 
        python -m debugpy --listen 0.0.0.0:3000 
        manage.py runserver 0.0.0.0:80"
     ports:
       - 3003:3000 # set 3003 to anything you want
```

Now, when running `docker compose up`, you should see the following in the logs of the `web` container, before the normal startup logs:

```bash
web-1  | Collecting debugpy
web-1  |   Downloading debugpy-1.6.7-py2.py3-none-any.whl (4.9 MB)
web-1  | Installing collected packages: debugpy
web-1  | Successfully installed debugpy-1.6.7
```

On vscode, create a new debug configuration, either using the UI (*debug* &gt; *add configuration*) or by creating the file `.vscode/launch.json`:

```json
{
    "version": "0.2.0",
    "configurations": [
      {
        "name": "CHANGEME",
        "type": "python",
        "request": "attach",
        "pathMappings": [
          {
            "localRoot": "${workspaceFolder}",
            "remoteRoot": "/app"
          }
        ],
        "port": 3003,
        "host": "127.0.0.1",
        "django": true,
        "justMyCode": false,
      }
    ]
  }
```

The important things:

* `pathMappings.remoteRoot` should match the folder where your code is mounted in the container
    
* `port` should match the one you mapped to the container's port `3000`, i.e. the port debugpy listens to
    
* `justMyCode` determines if breakpoints outside of your code (e.g. in libraries you use) work or not.
    

With this configuration, you can start a debug session (or hit `F5`) whenever you want and all your breakpoints should work. Once you are done debugging, simply "detach" the debugger using the detach icon.

ℹ️ if you need to debug something that only happens during startup, pass `-wait-for-client` to debugpy. When set, the Django app won't start until you start the debugger.

## 🐞 Debugging a Django celery worker running in Docker (no auto reload)

For the celery workers, the same principles apply. Simply change the `docker-compose.override.yml` to:

```yaml
services:
  worker:
    command: >-
      sh -c "pip install debugpy && 
      python -m debugpy --listen 0.0.0.0:3000
      /usr/local/bin/celery -A my.package.worker worker
      -l info -P solo --queues a,b"
    ports:
      - 3003:3000
```

Compared to the initial celery command, the big differences are:

* celery must be called using an absolute path (e.g. `/usr/local/bin/celery`) since debugpy looks for scripts in the working directory (in my case `/app`). Using just `celery` will raise:
    
    ```bash
     No such file or directory: '/app/celery'
    ```
    
* passing `-P solo` to celery instead of `--concurrency N` simplifies debugging, as only one celery thread is used.
    

❗⚠️❗ This won't AUTO RELOAD your worker ❗⚠️❗(keep reading 😉)

## 🔄 Auto-reloading a Django celery worker running in Docker (no debug)

To only auto-reload a celery worker, it is possible to use the awesome utility `watchmedo auto-restart` from the [watchdog](https://pythonhosted.org/watchdog/) package. `watchmedo` watches a set of files and/or directories and automatically restarts a process upon file changes.

The `docker-compose.override.yml` becomes:

```yaml
services:
  worker:
    command: >-
      sh -c "pip install "watchdog[watchmedo]" && 
      python -m watchdog.watchmedo auto-restart 
      -d src/ -p '*.py' --recursive
      celery -A my.package.worker worker -l info -P solo --queues a,b"
    ports:
      - 3003:3000
```

Common options of `watchmedo` are:

* the `-d` or `--directory` option is the directory to watch. It can be repeated.
    
* the `-p` or `--patterns` option restricts the watch to the matching files. Use `;` to list multiple patterns, for example `*.py;*.json`
    
* the `-R` or `--recursive` option monitors the directories recursively.
    

Note that we do not need to specify an absolute path for `celery` anymore.

❗⚠️❗ The debugger won't work ❗⚠️❗(keep reading 😉)

## 🔄🐞 Debugging a Django celery worker running in Docker with auto-reload

To make live reload and the debugger work, combining `debugpy` and `watchmedo` is not enough. I believe it has to do with debugpy restarting every time (thanks to watchmedo), hence losing the connection and context. In other words, we need the restart to happen in the debugged context.

*How does Django do auto-reload*? Looking at the source code, we can see that the `runserver` command uses [`django.utils.autoreload`](https://github.com/django/django/blob/main/django/utils/autoreload.py) under-the-hood (see [runserver.py::run](https://github.com/django/django/blob/main/django/core/management/commands/runserver.py#L118)). The cool thing is, this utility can also be used to run other processes!

Here is a simple python file that uses `autoreload` to run a celery worker:

```python
import django
# ❗ needs to be called *before* importing autoreload
django.setup()

from django.utils import autoreload

def run_celery():
    # ↓ import the Celery app object from your code
    from my_package.celery.app import app as celery_app
    # ↓ usual celery arguments
    args = "-A my.package.worker worker -l info -P solo --queues a,b"

    celery_app.worker_main(args.split(" "))

print("Starting celery worker with autoreload...")
autoreload.run_with_reloader(run_celery)
```

Don't forget to adapt the `args` and `Celery` object import to suit your needs.

Assuming this file is saved at the root of your project, the `docker-compose.override.yml` becomes:

```yaml
services:
  worker:
    command: >-
      sh -c "pip install debugpy && 
      python -m debugpy --listen 0.0.0.0:3000 worker.py"
    ports:
      - 3003:3000
```

For other ways of achieving the same, have a look at [\[StackOverflow\] Celery auto reload on ANY changes](https://stackoverflow.com/questions/21666229/celery-auto-reload-on-any-changes).
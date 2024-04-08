---
title: "Exploring The Magic of Python Through The Awesome Slumber Library"
seoDescription: "Learn about the slumber library and delve into the Python magic by re-implementing one of its core features."
datePublished: Mon Apr 08 2024 12:00:50 GMT+0000 (Coordinated Universal Time)
cuid: cluqwikbd000c08lb9amiajev
slug: slumber-and-python-magic
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1712512200292/a2c64108-439a-4554-be51-4970f4b2aaf5.png
tags: python, developer, tips

---

Slumber is one of those libraries you don't need, but can't live without once you learn about it (much like [attrs](https://www.attrs.org/en/stable/)!). It covers a generic use case - interacting with RESTful services - and is a prime example of what only the Python language lets you do. Intrigued? Let me explain!

🤲 (NOTE) I recently published a new library - [mantelo](https://github.com/derlin/mantelo) - that leverages the magic of slumber to provide a fully-fledged Keycloak Admin Client. Read more in my other article → [https://blog.derlin.ch/introducing-mantelo](https://blog.derlin.ch/introducing-mantelo).

---

* [Introduction to slumber](#heading-introduction-to-slumber)
    
    * [In a few words](#heading-in-a-few-words)
        
    * [Slumber in action: an example with the dev.to API](#heading-slumber-in-action-an-example-with-the-devto-api)
        
    * [A more formal explanation of the "translation"](#heading-a-more-formal-explanation-of-the-translation)
        
* [How the URL translation magic is implemented](#heading-how-the-url-translation-magic-is-implemented)
    
    * [Theory first: dunder methods](#heading-theory-first-dunder-methods)
        
    * [A simple implementation (&lt; 20 lines!)](#heading-a-simple-implementation-lt-20-lines)
        
* [Conclusion](#heading-conclusion)
    

---

## Introduction to slumber

### In a few words

From the documentation:

> Slumber is a python library that provides a convenient yet powerful object oriented interface to ReSTful APIs. It acts as a wrapper around the excellent [requests library](http://python-requests.org/) and abstracts away the handling of urls, serialization, and processing requests.

In short, slumber wraps a `requests.Session` and allows you to use Pythonic constructs to call RESTful APIs. Put even more simply, it *translates Python to HTTP calls*.

Still a bit fuzzy, isn't it? Let's look at a concrete example!

### Slumber in action: an example with the dev.to API

To better understand, let's assume we need to interact with [dev.to's Forem API v1](https://developers.forem.com/api/v1). First, let's spin up a REPL, import slumber (after a `pip install`) and create an API object:

```python
>> import slumber
>> api = slumber.API("https://dev.to/api")
```

Remember slumber doesn't know about the dev.to API. Yet, we can use it to query any of its endpoints in the following manner:

```python
# Get my profile image
# ⮕ GET https://dev.to/api/profile_images/{username}
>>> api.profile_images("derlin").get()
{'type_of': 'profile_image', ...}

# Get one of my articles on dev.to
# ⮕ GET https://dev.to/api/articles?username=derlin&per_page=1
>>> api.articles.get(username="derlin", per_page="1")
[{'type_of': 'article',
  'id': 1750725,
   ...
}]

# Try to create a user (oops, I am not allowed!)
# ⮕ POST https://dev.to/api/admin/users <body>
>>> api.admin.users.post({"email": "a@a.com", "name": "a"})
HttpClientError: Client Error 401: https://dev.to/api/admin/users/
```

The last call raised an exception, `401: unauthorized`. Normal, I am not authenticated. To change this, let's recreate the `api` object, this time passing some *auth*.

Since the Forem API uses custom headers for authentication instead of a known authentication mechanism, we can't rely on the [built-ins](https://requests.readthedocs.io/en/latest/user/authentication/) offered by requests (and thus slumber). So let's create our own authentication class:

```python
from requests.auth import AuthBase

# You can create an API key in dev.to's Settings > Extensions
DEVTO_API_KEY = "<xxx>"

# A custom Auth class that adds the right header
# to every request
class DevToAuth(AuthBase):
   def __call__(self, request):
      request.headers['api-key'] = DEVTO_API_KEY
      return request

# Tell slumber to use the custom auth
api = slumber.API("https://dev.to/api", auth=DevToAuth())
```

To test it, let's query my articles again:

```python
>>> api.articles.me.get()
[{'type_of': 'article',
  'id': 1750725,
  # ...
}]
```

Magical, right?

Under the hood, slumber uses a `requests.Session` , which can be tuned in case we need to add headers or other things. Another (simpler) way of authenticating would thus be:

```python
import requests

session = requests.Session()
session.headers['api-key'] = DEVTO_API_KEY

api = slumber.API("https://dev.to/api", session=session)
```

This example shows all you need to know about slumber.

### A more formal explanation of the "translation"

How does this translation from Python to an HTTP call actually works?

As you may have guessed from the dev.to example, the slumber `api` object starts with the base URL. Every property or method is then used to add to this base. When it reaches a method that looks like an HTTP method (`.get()`, `.post()`, `.delete()`, ...), slumber puts together the final URL, makes the call, parses the response, and returns either the response body (as a dictionary) or an exception.

And since an image is worth a thousand words:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1711986071189/160dc081-0f06-439f-a752-8ce2d53c9d11.png align="center")

## Delving into the magic

The "translation" explained above seems rather complex. Yet, the whole slumber library is less than 1000 lines of code! Let's see how the "Python magic" makes it possible by re-implementing the translation ourselves.

### Theory first: dunder methods

In Python, *dunder methods*, short for "double underscore" methods, are *special methods* (also called *magic methods*) that define behavior for built-in Python operations. For example, `__init__` initializes newly created objects, `__repr__` returns a string representation of an object, and `__add__` defines the behavior of the `+` operator. The ability to define/override such methods at the class level is one of the distinguishing traits of Python.

For our purpose, we need to familiarize ourselves with two dunders:

* `__call__(self)` : this method enables instances to be called as if they were functions / lets you define what happens when using parentheses on class instances (`my_instance()`).
    
* `__getattr__(self, item)`: this method is invoked when an undefined attribute is accessed on an object. If not defined, the normal behavior is to raise an `AttributeError`.
    

### A simple implementation (&lt; 20 lines!)

First, let's create a class that has a base URL and implements two HTTP methods (body left as an exercise 😉):

```python
class Resource:
    def __init__(self, url):
        self.url = url

    def get(self, **query_params): ...  # GET <url>
    def post(self, body=None, **query_params): ...  # POST <url>
```

With this base, we now have to implement the URL construction. How? Let's start with the addition of a path to the URL. In slumber, we do this by using an attribute *unknown* to the instance. Does it ring a bell?

```python
class Resource:
    # ... rest of the implementation ...
    def __getattr__(self, item):
        return Resource(f"{self.url}/{item}")
```

Now, whenever we call an attribute on a `Resource`, it returns a new `Resource` with the path segment added to the URL. What about path parameters? Well, same principle, just another dunder:

```python
class Resource:
    # ... rest of the implementation ...
    def __call__(self, path_param):
        return Resource(f"{self.url}/{path_param}")
```

The final touch is to bootstrap the whole thing by making the API object also return a `Resource` when an unknown attribute is accessed. A full implementation would thus look like:

```python
class Resource:
    def __init__(self, url):
        self.url = url

    def get(self, **query_params):
        qs = "&".join([f"{k}={v}" for k, v in query_params.items()])
        print(f"GET {self.url}?{qs}")

    def post(self, body=None, **query_params):
        qs = "&".join([f"{k}={v}" for k, v in query_params.items()])
        print(f"POST {self.url}?{qs}")

    def __getattr__(self, item):
        return Resource(f"{self.url}/{item}")

    def __call__(self, path_param):
        return Resource(f"{self.url}/{path_param}")


class API:
    def __init__(self, base_url):
        self.base_url = base_url

    def __getattr__(self, item):
        return Resource(self.base_url)
```

Let's try this out!

```python
>>> api = API("https://example.com/api/v1")
>>> api.users("lala").foo_bar.get(x=1, y="buzz")
GET https://example.com/api/v1/lala/foo_bar?x=1&y=buzz
```

With these 20 lines of code, we demystified all of slumber's magic. For this kind of thing, Python is quite awesome 😎.

## Conclusion

Even though I am more and more drawn to typed languages, Python has some nice tricks in its sleeves. I love how slumber leveraged it to provide a simple yet useful library. It is a prime example of a *good* use of dunder methods.

I hope you'll remember slumber the next time you need to interact with an API!

---

If you liked this article, leave a comment or a thumbs up, or share it around ... This would help keep my motivation up!
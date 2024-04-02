---
title: "Introducing Mantelo - The Best Keycloak Admin Client for Python"
seoDescription: "Let me introduce mantelo, the best Keycloak Admin Client for Python. We will see how it works and why it is so cool."
datePublished: Tue Apr 02 2024 12:00:24 GMT+0000 (Coordinated Universal Time)
cuid: cluibuvux000808juc5rv7ojq
slug: introducing-mantelo
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1711729219676/a6bd64ba-c2ff-4b71-a6cd-c78b15c78c4a.png
tags: python, opensource, keycloak, python-libraries

---

I'm thrilled to present my newest open-source project! [Mantelo](https://mantelo.readthedocs.io/en/latest/) is a super small yet super powerful Python library for interacting with the Keycloak Admin API.

[![](https://github.com/derlin/mantelo/blob/main/docs/_static/images/mantelo-social-preview.png?raw=true align="center")](https://github.com/derlin/mantelo)

> Mantelo \[manˈtelo\], from German "*Mantel*", from Late Latin "*mantum*" means "*cloak*" in Esperanto.

Born from the frustration of encountering incomplete support in existing libraries, this is **the only Python client that guarantees full coverage of all the routes in the Keycloak Admin RESTful API**. How? By *wrapping instead of implementing*. Let me explain 😊.

---

* [The motivation behind mantelo](#heading-the-motivation-behind-mantelo)
    
* [Getting started with mantelo](#heading-getting-started-with-mantelo)
    
    * [**🏁** Installing](#heading-installing)
        
    * [**🔐** Authenticating](#heading-authenticating)
        
    * [📞 Making calls](#heading-making-calls)
        
* [The best library out there, really?](#heading-the-best-library-out-there-really)
    
* [Mantelo needs YOU](#heading-mantelo-needs-you)
    

---

## The motivation behind mantelo

When I started working with Keycloak in Python, I first opted for the renowned [python-keycloak](https://python-keycloak.readthedocs.io/en/latest/), considered the most complete library.

It worked well until I had to query a resource that was not supported. I diligently submitted a [pull request](https://github.com/marcospereirampj/python-keycloak/pull/478) to add a `get_idp` method, which took three months to be merged, and a bit more to be released. This doesn't mean the maintainers are incompetent (they are great!), just that PRs in significant open-source projects take time.

Everything proceeded smoothly until I encountered another wall: python-keycloak implements none of the endpoints related to TOTP. I had three choices:

1. Submit PR(s) and postpone everything for months (*slow*),
    
2. Use raw HTTP requests instead of python-keycloak in part of the code (*ugly*),
    
3. Create a library that supports *ALL* the endpoints, so I will never be in this position again (*fun!*).
    

It took a weekend for the code, and a few more for the publishing, documentation, and all the other ornaments of an open-source project. But there it is!

## Getting started with mantelo

Mantelo is hosted on [GitHub](https://github.com/derlin/mantelo), available on [PyPi](https://pypi.org/project/mantelo/), and hosts its documentation at readthedocs: [https://mantelo.readthedocs.io/en/latest/](https://mantelo.readthedocs.io/en/latest/).

### **🏁** Installing

To no one's surprise:

```bash
pip install mantelo
```

### **🔐** Authenticating

The first step is to create a `KeycloakAdmin` and specify a way of authenticating. Mantelo supports natively username/password and client credentials (OpenID), taking care of tokens and refreshes behind the scenes.

More details are available in the docs, but for the demo let's use the default `admin` user with the default `admin-cli` client and connect to the `master` realm running on a local Keycloak instance:

```python
from mantelo import KeycloakAdmin

adm = KeycloakAdmin.from_username_password(
    # base Keycloak URL
    server_url="http://localhost:8080",
    # realm to interact with (and authenticate to if not specified otherwise)
    realm_name="master",
    # Client to use for authentication
    client_id="admin-cli",
    # User to use for authentication
    username="admin",
    # Password to use for authentication
    password="CHANGE-ME",
)
```

ℹ This code doesn't interact with Keycloak: Mantelo is lazy and only fetches/refreshes the token when needed (e.g. on the first request).

### 📞 Making calls

You are now ready to interact with the Admin API. One advantage of mantelo is that it ***doesn't provide any specific method, it only offers an object-oriented interface to the RESTful API***. It ensues any endpoint present in the Admin API is callable using the `adm` object.

First, let's look at an example. Let's say you need to manage users. Here is a sample of what you can do:

```python
# Get the list of users
# ⮕ GET /admin/realms/{realm}/users
adm.users.get()
# Search users for "foo bar"
# ⮕ GET /admin/realms/{realm}/users?search=foo+bar
adm.users.get(search="foo bar")
# Get user count
# ⮕ GET /admin/realms/{realm}/users/count
adm.users.count.get()
# Get a specific user by ID
# ⮕ GET /admin/realms/{realm}/users/725209cd-9076-417b-a404-149a3fb8e35b
adm.users("725209cd-9076-417b-a404-149a3fb8e35b").get()
# Create a new user, with credentials
# ⮕ POST /admin/realms/{realm}/users
adm.users.post({
    "username": "example",
    "firstName": "Winston",
    "lastName": "Smith",
    "email": "orwell@1984.uk",
    "emailVerified": True,
    "enabled": True,
    "credentials": [
        {
            "type": "password",
            "value": "juliaMyLove",
        }
    ],
})
```

Did you get it? Instead of implementing methods to reach endpoints, mantelo uses simple rules to translate Python code to HTTP calls. More formally:

1. Each *property* is a path appended to the *base URL*. If the `server_url` is `https://localhost:8080` and the realm is `master`, the *base URL* is `https://localhost:8080/admin/realms/master`. Underscores are automatically translated to dashes.
    
2. The same goes for *methods*, useful for specifying path parameters: the first argument is appended to the path.
    
3. The actual HTTP call is triggered by ending a "chain" with a call to one of:
    
    * `.get(**kwargs)`
        
    * `.options(**kwargs)`
        
    * `.head(**kwargs)`
        
    * `.post(data=None, files=None, **kwargs)`
        
    * `.patch(data=None, files=None, **kwargs)`
        
    * `.put(data=None, files=None, **kwargs)`
        
    * `.delete(**kwargs)`
        
4. All the above methods have `kwargs` which are appended as query parameters.
    
5. POST/PUT/PATCH support `data` (a dictionary converted to JSON) or `files` for specifying the body of the request.
    

Every request returns either the content of the response's body (parsed from JSON) or an instance of `HttpException` if the response's status code is not in the `2XX` range.

And that's it! This magic is possible thanks to [slumber](https://github.com/samgiles/slumber), a library I will feature in a separate article (stay tuned 😉).

## The best library out there, really?

I stated earlier I believe mantelo is the best Keycloak Admin library. Let me summarize the premises of such a bold claim:

1. mantelo's footprint is ridiculous: it is currently less than 1000 lines of code and has only 3 direct dependencies.
    
2. mantelo is the only library that can rightfully claim it supports the whole Keycloak Admin interface and is always up-to-date.
    
3. mantelo abstracts away authentication (and refresh tokens), which is always tricky to get right.
    
4. mantelo gives you access to the exact URL that was called (or the `requests.Response` in case of an error) so debugging is easy.
    
5. mantelo is flexible: you can tweak it easily if you need to.
    

I am aware *wrapping instead of implementing* has a drawback: if Keycloak changes the Admin API, your code has to be updated (no encapsulation). But I believe it is a small price to pay, especially since Keycloak proved to be careful about retro-compatibility. What do you think?

## Mantelo needs YOU

I've put a lot of work into this library, not just on the code, but everything around it—like documentation, making it easy to test, setting up the GitHub repo, and more.

But mantelo isn't complete without users and a community. So please, give it a shot, star it, criticize it, open an issue... I'm excited to see where we can take it together!
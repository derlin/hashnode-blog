---
title: "Let's RickRoll!"
seoDescription: "A showcase of rickroller, a website to RickRoll your friends like a pro. I will discuss usage, design, implementation, and more!"
datePublished: Mon Jun 12 2023 12:00:42 GMT+0000 (Coordinated Universal Time)
cuid: clissyzif01snw2nvg1ql7utj
slug: rickroll-your-friends
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1686291630057/da54711e-4669-4ab2-8ce5-e6db16d1fefa.png
tags: programming-blogs, python, showhashnode, rickroll

---

I while back, I was looking for a fun project to test open-source best practices (docker, GitHub Actions, etc). I came up with "rickroller", the perfect tool to rickroll your friends like a pro!i

In this article, I want to share with you my vision of a perfect RickRoll prank (demo ⮕ [https://tinyurl.eu.aldryn.io](https://tinyurl.eu.aldryn.io)), and some interesting aspects of its implementation. If you want to learn more about open-source best practices, have a look at the README (⮕ [https://github.com/derlin/rickroller](https://github.com/derlin/rickroller)).

---

* [What is RickRolling?](#heading-what-is-rickrolling)
    
* [My vision of a perfect rickroller](#heading-my-vision-of-a-perfect-rickroller)
    
* [The final result](#heading-the-final-result)
    
* [How to Rick Roll your friends like a pro](#heading-how-to-rick-roll-your-friends-like-a-pro)
    
    * [Rendering the "original" page](#heading-rendering-the-original-page)
        
    * [Instrumenting the page (ie. redirect)](#heading-instrumenting-the-page-ie-redirect)
        
    * [The "You got rick rolled" page](#heading-the-you-got-rick-rolled-page)
        
    * [The URL shortener disguise](#heading-the-url-shortener-disguise)
        
    * [Avoiding SSRF attacks](#heading-avoiding-ssrf-attacks)
        
* [The deployment](#heading-the-deployment)
    
* [Conclusion](#heading-conclusion)
    

*(TOC generated with* [*bitdowntoc*](https://derlin.github.io/bitdowntoc/) *using the Gitlab preset and the anchor prefix option set to* `heading-`*)*

---

## What is RickRolling?

I am sure you already got RickRolled at least once in your life. But for the lucky ones out there, Rick Roll is an internet prank that started around 2007 on online bulletin boards like 4chan and Reddit, where users would post a link that unexpectedly directed to a video of Rick Astley's ["*Never Gonna Give You Up*"](https://www.youtube.com/watch?v=dQw4w9WgXcQ).

Since 2007, the prank bubbled and it is now everywhere. There are so many fun websites and tools available around Rick Roll, such as [r.mtdv.me](http://r.mtdv.me), a Rick Roll Link generator, or [rickrollrc](https://github.com/keroserene/rickrollrc), a Bash script playing the video with ANSI 256-color coded UTF-8 characters + audio (if available). There is even an [AntiRickroll Chrome Extension](https://chrome.google.com/webstore/detail/antirickroll/mpnckpmpddjcgkpjkmmakcamjhceadne)!

## My vision of a perfect rickroller

Most of the tools out there are based on disguising a link that opens directly on Rick's video. Which is fine, but lacks subtlety. I wanted to go further.

The idea? Have people think they are on a real (and interesting) website *before* getting redirected. More precisely, here is how I envision the prank:

1. you send a link to a friend that looks like the shortened URL of a very interesting web page,
    
2. the friend opens the link and lands on what looks like the real stuff,
    
3. the friend interacts with the page (click, scroll), and BOOM, RickRoll!
    

This makes for a more subtle and fun Rick Roll, don't you think?

To make this prank possible (because of course I implemented it), I need a web server with three different views:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1686073241536/1506f815-2cc3-48f4-8dd1-a7e03ce9d904.png align="center")

View 1 is the prankster interface, where he can choose which original URL (web page) to fake. In view 2, the server determines the original URL to masquerade (e.g. thanks to a query parameter - keep reading!) and renders the original URL content but with slight changes making it redirect to view 3 upon interaction. That's "rickroller"!

## The final result

My Rickroller implementation is a simple [Python Flask](http://flask.palletsprojects.com/) app. You can test it at [https://tinyurl.eu.aldryn.io](https://tinyurl.eu.aldryn.io) (hosted by the awesome PaaS ♡ [**Divio**](https://docs.divio.com) ♡).

Here is the view to create a rickroll (view 1):

[![preview](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wx03wal2ruyr9dv1tj88.png align="left")](https://tinyurl.eu.aldryn.io)

Try to enter a URL, then interact with the fake page. You should end up on:

![rickroll!](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/btmt91u9762pxh06uavy.png align="left")

Cherries on the cake, the social preview from the original site shows up when you share a fake link. Here is the result of sharing the malicious https://dev.to URL on Telegram:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1686302802919/3dbce45b-fd2c-43a4-b0c7-14b71f75641a.png align="center")

The best is to try it yourself, it will become clearer. The code and much more details are available on GitHub ⮕ [https://github.com/derlin/rickroller](https://github.com/derlin/rickroller).

IMPORTANT: the fake links may become obsolete after a few days.

## How to Rick Roll your friends like a pro

So, how did I end up with this? I won't go into all the details, but let's dive into some interesting points.

### Rendering the "original" page

The first challenge is to render a "clone" of the original URL. We can easily get the HTML from the original URL and return it. However, without modifications, it will be completely broken. Why?

1. most pages use relative links (to images, scripts, CSS, etc).
    
2. some pages load resources protected by [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) - cross-origin request sharing.
    

For 2, we can't do much. For 1, though, my first idea was to go through all the links in the HTML and *absolutize* them (prepend the original URL to it). The full implementation is available [here](https://github.com/derlin/rickroller/blob/e15f2a1437a0fdff245b48a48e0414aa1acc577a/rickroller.py), but basically:

```python
import requests
from bs4 import BeautifulSoup
from urllib.parse import urljoin

def absolutize(tagname, attr, soup, url):
  for elt in soup.find_all(tagname, **{attr: True}):
    elt.attrs[attr] = urljoin(url, elt.attrs[attr])

# get the original HTML
response =  requests.get(original_url)
if response.status_code != 200: raise 
soup = BeautifulSoup(response.text, 'html.parser')

# absolutize all important tags
args = [soup, original_url]
absolutize('a', 'href', *args)
absolutize('link', 'href', *args)
absolutize('script', 'src', *args)
absolutize('img', 'src', *args)

# + fix 'img' with 'srcset', or else images using it
# won't load
```

Good, but complex and error-prone... And it won't work for relative URLs located inside scripts or CSS! How could we make it better?

**Meet the** [`<base>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/base) **tag!**

> *The* `<base>` HTML element specifies the base URL to use for all relative URLs *in a document.*

Yup, just adding a `<base href="{base}">` in the header can replace all the code from my first attempt. Cherries on the cake, it will also work for relative links in CSS files!

```python
def absolutize(soup, url):
  # ... add <head> if missing ...
  base = soup.head.find("base")
  if base is None:
      tag = soup.new_tag("base")
      tag.attrs["href"] = url
      soup.head.insert(0, tag)
  else:
      base.attrs["href"] = urljoin(url, base.attrs["href"])
```

Done :)

### Instrumenting the page (ie. redirect)

Now that we rendered a clone, it is time to modify it so it redirects to a "rickrolled" page.

At first, I just wanted to redirect the user on click. I thus naturally thought about changing all the links on the page to some Rick Roll video.

Since I already got a "soup" (`BeautifulSoup` object), I can easily alter all the `<a>` elements *in the body*. This last part is important because the `<head>` contains links to resources (CSS, js, etc.) that need to be loaded properly for the page to look remotely normal.

```python
for a in soup.find('body').find_all('a'):
  a.attrs['href'] = __RICK_ROLL_URL__
```

Simple enough, but the result is far from perfect...

Most problems are caused by this increasingly in-vogue habit of using `<a>` elements for "page logic" instead of "real navigation": opening a popup, animating a menu, controlling accordions, etc. In this case, changing the `href` breaks the original behaviour (the menu doesn't work, or the accordion stays closed) *and* doesn't redirect the user either (because of a sneaky JS event handler in the original page calling `event.preventDefault` on click).

So, how do you ensure any click on an `a` element redirects to Astley's without breaking the page? **Javascript!**

Instead of changing links, I can insert a JS script that hijacks all click events and redirects to the rickrolled page. If I add it at the very bottom of the page, it will take precedence over everything else.

The Javascript looks like this:

```javascript
document.addEventListener("DOMContentLoaded", function(event) {
    document.addEventListener("click", e => {
        e.stopPropagation();
        e.preventDefault();
        window.location = "%s"
    }, true);
```

And can be inserted as the last child of the body with:

```python
tag = soup.new_tag("script")
tag.attrs["type"] = "text/javascript"
tag.string = __JS_SNIPPET__

soup.body.insert(len(soup.body.contents), tag)
```

I went further to also redirect on scroll. I won't talk about it here, but have a look at the code if you are interested.

### The "You got rick rolled" page

I talked about redirecting a lot. But redirecting where?

Usually, a RickRoll means redirecting to a YouTube video. Apart from choosing one from the zillions available, there are two things I don't like: advertisements and mobile behaviour. On mobile, YouTube links (slowly) open in the YouTube app, tipping the prankee before the video even starts.

So, I will redirect to a page of mine, which plays the rickroller video.

I first thought of using a *video embed*, such as a [YouTube embed](https://support.google.com/youtube/answer/171780?hl=en). However, did you know it is now **nearly impossible to autoplay a video with sound**? From [Chrome's blog](https://developer.chrome.com/blog/autoplay/):

> *web browsers are moving towards stricter autoplay policies in order to improve the user experience, minimize incentives to install ad blockers, and reduce data consumption on expensive and/or constrained networks \[...\].*

In Chrome, autoplay with sound is only allowed on very strict conditions and always requires some action from the user. All other major browsers are implementing similar policies. This was a huge bummer. No sound it is!

Even worse, the `&autoplay=1` parameter on YouTube embeds is buggy as hell: the video started neither on my mobile phone nor my Vivaldi browser. Instead, I got a dumb preview with a play button. I played with other sources providing embeds for the Astley video, but none offered a good enough experience. And more problematic, those obscure websites could go down anytime.

But wait, if a video doesn't autoplay, a GIF definitely will! If we can't have sound anyway, a good-quality GIF is as good as any. I thus ended up using a [giphy](https://giphy.com) Rick Roll GIF embedded into a very simple web page. It worked perfectly for a while, until the GIF disappeared from giphy, leaving in its place:

![oops giphy](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ox2pzxnrq9gec4sygo5p.png align="left")

To avoid another abrupt disappearance, I downloaded a gif from [tenor.com](http://tenor.com) that is now served directly by my server. This is what you see in the final result: [https://tinyurl-stage.eu.aldryn.io/gotcha](https://tinyurl-stage.eu.aldryn.io/gotcha)

### The URL shortener disguise

For rickroller to work, the original URL needs to be retrievable in some way from the URL that is shared with your friends (view 2 in the schema), so that rickroller knows what to instrument+display. I can't use sessions, since the URL is meant to be shared.

The easiest is to use a query parameter: `?url=<escaped-url>`. It works, but asking a friend to click on an URL that looks like`https://some.server.com?u=https%3A//discuss.kotlinlang.org/t/react-in-kotlin-js-what-i-learned-long-but-useful-read/16168` is unlikely to land.

Without adding a layer of persistence, there is however nothing we can't do: there is no lossless compression that exists for URL strings... If I have a database, though, I could disguise rickroller into an URL shortener!

The idea is simple: when a prankster inputs a URL in view 1, I generate a unique (and short) UUID for it and store the tuple (URL, UUID). The prankster can then use `https://some.server.com/t/UuIDxxxx` . When the prankee requests `https://some.server.com/t/UuIDxxxx` , a simple lookup for `UuIDxxxx` in the database returns the original URL that rickroller can then display.

Since I didn't know how and where I would deploy the final version of RickRoller, I decided to provide multiple persistence layers:

1. SQLite in-memory → can be used for local testing,
    
2. SQL-like (SQLite, PostgreSQL, MySQL),
    
3. MongoDB (only because [mongodb.com](http://mongodb.com) has a [free plan](https://www.mongodb.com/pricing)!)
    

### Avoiding SSRF attacks

Since rickroller does HTTP GET on URLs provided by the users, it is a perfect target to Server-Side Request Forgery (SSRFs).

To give you an example, let's say rickroller is deployed on an AWS EC2 instance and a malicious user passes the URL `http://169.254.169.254` - the [metadata service](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html). The user doesn't have access, but rickroller does! This could leak very sensitive information (security groups, user data, ...). If this is unclear, [\[Hacking the Cloud\] Steal EC2 Metadata Credentials via SSRF](https://hackingthe.cloud/aws/exploitation/ec2-metadata-ssrf/) gives a more thorough explanation.

Long story short, I need some kind of security in place. The easiest is to simply deny any "private" URL. But checking the URL provided by the user is not enough! Why? Because by default `requests.get()` will follow redirects. A hacker could set up a small server with a public address that answers with a redirect to a private URL!

My solution is thus:

1. do the `requests.get` call, and record the *history* (that is, all the potential redirects, or hops)
    
2. ensure that no private URL is found in the history
    

Here is the code:

```python
from ipaddress import ip_address
from socket import gethostbyname
from urllib.parse import urlparse
import requests

def __ensure_is_safe(url: str):
    hostname = urlparse(url).hostname
    if hostname is None:
        raise Exception(url, f'Could not extract hostname from "{url}"')
    ip = gethostbyname(hostname)

    if ip is None or ip_address(ip).is_private:
        raise Exception(
            url,
            f"{url} maps to an unknown or private ip address: {ip}.",
        )

def safe_get(url):
    response = requests.get(url, timeout=30)
    [__ensure_is_safe(r.url) for r in response.history]
    return response

print(safe_get('https://dev.to'))
```

## The deployment

With all this in place, time to deploy. I manage the release cycle and the publication of the docker images (ARM + AMD) to DockerHub with GitHub Actions.

I have one instance running on Google Cloud Run that uses the free MongoDB database from mongodb.com. I can redeploy it at any time using a manually triggered GitHub Action. However, since I do not want to invest too much into it, I use the cheapest Cloud Run instance and everything is slow (the instance stops and starts for each request).

I explain everything at length in the [README](https://github.com/derlin/rickroller) if you want more details.

For the actual demo, I host another instance on [divio.com](https://docs.divio.com), which is an awesome PaaS with far better response times. More precisely, divio deploys rickroller on AWS for me (I chose the European region), taking care of all the heavy work (CloudFront, DNS, backups, security, etc). It also comes with a nice UI and CLI.

## Conclusion

What started as a dumb and easy project (code-wise) turned up to be a bit challenging. I really enjoyed going the extra mile (deployment, release management, etc) and learned a lot along the way. I can only encourage you to start your own fun project and do the same!

I hope you enjoyed the read and learned something. Please, go ahead and play with the demo: Long live RickRoller!

With love, [@derlin](https://derlin.ch)
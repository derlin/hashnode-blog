---
title: "Understanding Cloudflare Caching: What Gets Cached and How to Control It"
seoDescription: "Let's understand Cloudflare's default caching behaviors and how to make it cache HTML pages the right way."
datePublished: Mon Mar 17 2025 13:00:09 GMT+0000 (Coordinated Universal Time)
cuid: cm8d2o0ww000808i5hadm4sj3
slug: understanding-cloudflare-caching
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1742146543215/d170b795-bddd-45d9-9b00-f39ebbf6be74.png
tags: cloudflare, webdev, caching

---

I used to think that Cloudflare caching just worked out of the box - set your website behind Cloudflare, and boom! Your HTML and media pages are cached, performance increases, and your boss is happy.

Well… not quite. Cloudflare is a great tool, but to truly take advantage of it, you must take the time to understand how it works. Especially, you may have to set up a somewhat counter-intuitive rule. Curious? Let’s break it down!

ℹ **BEFORE WE START** ℹ This article is far from complete, I couldn’t cover all of Cloudflare’s cache nuances without losing my audience (and myself). Instead, I will focus on key concepts and configurations, assuming you already know how to set up an application behind Cloudflare.

---

In this article:

* [Cloudflare Cache Statuses](#heading-cloudflare-cache-statuses)
    
* [What is a “resource”](#heading-what-is-a-resource)
    
* [Which resources are eligible for cache](#heading-which-resources-are-eligible-for-cache)
    
* [Which eligible resources are cached](#heading-which-eligible-resources-are-cached)
    
* [How to extend cached resources to HTML (and others)](#heading-how-to-extend-cached-resources-to-html-and-others)
    

*TOC created using* [*https://bitdowntoc.derlin.ch*](https://bitdowntoc.derlin.ch)

---

## Cloudflare Cache Statuses

First, let’s understand the headers and tools Cloudflare gives us to understand how our pages are cached (or not). When a page is served through Cloudflare, it adds some `Cf-*` cache headers.

The most important one is `Cf-Cache-Status`, which can take the following values:

* `HIT` → cacheable content, served from the cache.
    
* `MISS` → cacheable content, never cached (or at least not in the cache at the time of the request). Served from the origin.
    
* `EXPIRED` → cacheable content, was in the cache but has an expired TTL. Served from the origin.
    
* `BYPASS` → cacheable content, not cached due to directive, cookie, etc …
    
* `DYNAMIC` → uncacheable content
    

So, resources (or URLs) are organized into *cacheable* and *uncacheable* assets. Cacheable resources have more nuanced statuses, depending on the state of the Cloudflare cache and other settings such as the TTL, the Cookies, etc.

(Another useful header is the `Cf-Ray`, which is a unique identifier of this request. You can use this ID in the CloudFlare console to find out more about the request)

## What is a “resource”

*👉 By default, Cloudflare considers URLs with different query parameters as different pages.*

In the previous section, we talked about *resources* and hinted they mapped to URLs. To clarify, the definition of a resource depends on the **Cache Level** of your Cloudflare zone. The Cache Level can be:

* **Basic**, no query string → only cache when the URL has no query string
    
    * `derlin.ch/main` is cached, `derlin.ch/main?foo=bar` isn’t
        
* **Simple**, ignore query strings → deliver the same resource regardless of query strings
    
    * `derlin.ch/main?ignore=this` is cached and the same resource as `derlin.ch/main`
        
* **Standard**, aggressive → deliver a different *resource* with each unique query string
    
    * `derlin.ch/main?foo=bar`, `derlin.ch/main` and `derlin.ch/main?foo=bar&x=1` are all separate resources cached separately
        

The Cache Level is **Standard** by default, meaning a CloudFlare resource maps 1-1 to a URL.

## Which resources are eligible for cache

*👉 Resources eligible for cache must end with a whitelisted extension; JSON and HTML are not cached.*

What makes a resource eligible for cache?

Let’s say you have your portfolio website behind Cloudflare. You would expect your public home page, e.g. `https://derlin.ch`, to be cached automatically, right? Nope!

> Cloudflare only caches based on file extension and not by MIME type.  
> The Cloudflare CDN ***does not cache HTML or JSON by default***.

When we say *extension*, we talk about the last bit of the URL (think file extensions). The `Content-Type` header (MIME type) is never taken into account. Furthermore, Cloudflare has a list of [default cached file extensions](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/#default-cached-file-extensions) (`.jpg`, `.css`, `.js`, …), and URLs not part of this list are *de facto* uncacheable.

To better understand:

* `derlin.ch/data/cv.pdf` is cacheable (`.pdf` is part of the list)
    
* `derlin.ch/github-logo.png` is cacheable (`.png` is part of the list)
    
* `derlin.ch/index.html` is *not* cacheable (`.html` is not part of the list)
    
* `derlin.ch` or `derlin.ch/main` is *not* cacheable (no extension)
    

## Which eligible resources are cached

*👉 Cloudflare strictly respects cache control headers returned by the origin.*

Once a resource is eligible for cache (cacheable), Cloudflare still needs to decide whether or not to cache it. For this, it **strictly respects cache-control headers**. In other words, the cache / not cache decision is based on the following information:

1. the [`Cache-Control`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control) and `Expires`headers
    
2. the origin status code
    

From the Mozilla documentation:

> The HTTP [`Cache-Control`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control) holds *directives* (instructions) in responses (and requests!) that control caching in browsers and shared caches (e.g., Proxies, CDNs).
> 
> The HTTP [`Expires`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Expires) response header contains the date/time after which the response is considered expired in the context of HTTP caching.

Most web frameworks these days come with pre-configured cache-control directives. If your application doesn’t have them, you have work to do!

---

*QUICK DIGRESSION*

I have to cite the [docs](https://developers.cloudflare.com/cache/concepts/cache-control/) on this, just because I love the “*its an option, but you can’t do anything about it on basically all subscription levels*” 😂:

> When enabled on an Enterprise customer's website, it indicates that Cloudflare should strictly respect `Cache-Control` directives received from the origin server. Free, Pro, and Business customers have this **option enabled by default** and cannot disable it.
> 
> Cloudflare's [Cache Rules](https://developers.cloudflare.com/cache/how-to/cache-rules/) allows users to either augment or override an origin server's `Cache-Control` headers or [default policies](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/) set by Cloudflare.

---

When fetching a resource from the origin, Cloudflare looks at the response and applies the following [default cache behavior](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/):

* Cloudflare **does** cache the resource if one of the following is true:
    
    * The `Cache-Control` header is set to `public` and `max-age` is greater than 0
        
    * The `Expires` header is set to a future date
        
        *(If both* `max-age` and an `Expires` header are present, Cloudflare uses `max-age`)
        
* Cloudflare **does not** cache the resource if one of the following is true:
    
    * The `Cache-Control` header is set to `private`, `no-store`, `no-cache`, or `max-age=0`
        
    * The [Set-Cookie header](https://developers.cloudflare.com/cache/concepts/cache-behavior/#interaction-of-set-cookie-response-header-with-cache) exists.
        
    * The HTTP request method is anything other than a `GET`.
        

If there is no `Cache-Control` or `Expires`, CloudFlare caches responses (with varied TTL) only when the origin status code is one of `200`, `206`, `301-303`, `404` and `410`.

In other words:

* all resources with cacheable extensions will be cached unless you have headers that tell Cloudflare otherwise
    
* redirects and 404s will be cached
    
* errors such as 500 won’t be cached
    

(**NOTE**: Your zone settings may override this behavior, and there are additional specifics I haven’t covered - e.g. [cacheable size limits](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/#cacheable-size-limits), `Set-Cookie` handling, and more. When in doubt, always refer to the official documentation.)

## How to extend cached resources to HTML (and others)

You have understood by now that most of the pages making up your website (HTML, JSON, …) are not even considered for caching - even though they would likely benefit from it!

You may believe changing this behavior is tricky and requires a deep knowledge of your pages and Cloudflare. What if I tell you it only requires a teeny-tiny generic rule at the right place?

Indeed, assuming your application is configured to return relevant cache-control headers, the magic comes in one of two forms:

1. either create a new *Page Rule* matching `*` and setting `Cache-Level: Cache Everything`
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1742125823023/5f1d79bb-1a84-4db5-98fe-1a76ec78a7ff.png align="center")
    
2. or use a *Cache Rule* with a similar yield:
    
    ![](https://cdn.hashnode.com/res/hashnode/image/upload/v1742125888669/52c7828a-b410-41bb-b64c-5263f5b457c3.png align="center")
    

(Cache Rules are now preferred over Page Rules).

It may sound weird (especially the page rule), but this will **promote all resources to cacheable** (a.k.a. eligible for cache). If you followed along, you know that it doesn’t impact the next step of the process: Cloudflare will still use the Cache-Control headers and status code to make the final cache / no cache decision.

In other words, your admin pages (with a proper `Cache-Control: private, no-store, no-cache`) and pages with a session cookie won’t be cached, but now your HTML home page (with a proper `Cache-Control: public, max-age=3600`) will!

Once this rule is on, the number of `DYNAMIC` pages (uncacheable) will reduce drastically. You may still need to fine-tune the behaviors, but at least it will provide a decent baseline to work with.

---

I was initially confused by Cloudflare’s caching behavior, especially when HTML pages returned a `DYNAMIC` cache status. The documentation is extensive but overwhelming and I spent quite a few hours on it, trying to figure it out. The redaction of this article helped clarify everything in my head.

The rule shared in the last section is something we use in production. Without it, the cache gains would be far less tangible.

I can only hope it helps you too. Don’t hesitate to drop a like or a comment, your reactions always make me smile 🤗.
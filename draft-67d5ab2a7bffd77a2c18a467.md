---
title: "Let's talk about CloudFlare caching"
slug: lets-talk-about-cloudflare-caching

---

Cloudflare caching is weird… Or at least, counter-intuitive. Don’t believe me? Read on!

## The weird bit

We are managing a CloudFlare zone in production that is used to cache different client websites. We do not control the websites; we only control the CloudFlare setup.

To make it work as expected, we use this “magic” Page Rule: *cache everything on all URLs*.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1742056942298/30292bbf-f612-4c86-8d39-21a490bde93f.png align="center")

You may not find it obvious, but think about it: what about pages with CSRF tokens? And private pages? Or pages only accessible after a login? If any of those is cached, consequences go from errors to tremendous security flaws!

To understand why we allow such rules in production, and why this is not an issue but a needed feature, let’s dive into CloudFlare caching.

## An overview of CloudFlare caching

### Cache level

For each zone in CloudFlare, you have to configure a Cache Level:

* **Basic**, no query string → only cache when the URL has no query string
    
    * `derlin.ch/main` is cached, `derlin.ch/main?foo=bar` isn’t
        
* **Simple**, ignore query strings → deliver the same resource regardless of query strings
    
    * `derlin.ch/main?ignore=this` is cached and the same as `derlin.ch/main`
        

* **Standard**, aggressive → delive a different resource with each unique query string
    
    * `derlin.ch/main?foo=bar`, `derlin.ch/main` and `derlin.ch/main?foo=bar&x=1` are all cached separately
        

The Cache Level is **Standard** by default.

### **Origin Cache-Control is king**

Cloudflare respects the origin web server’s cache headers, set in the `Cache-Control` headers in the origin’s responses. I have to cite the [docs](https://developers.cloudflare.com/cache/concepts/cache-control/) on this, just because I love the “*its an option, but you can’t do anything about it on basically all subscription levels*” 😂:

> When enabled on an Enterprise customer's website, it indicates that Cloudflare should strictly respect `Cache-Control` directives received from the origin server. Free, Pro, and Business customers have this **option enabled by default** and cannot disable it.
> 
> Cloudflare's [Cache Rules](https://developers.cloudflare.com/cache/how-to/cache-rules/) allows users to either augment or override an origin server's `Cache-Control` headers or [default policies](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/) set by Cloudflare.

The [default cache behavior](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/) treats the `Cache-Control` directives as such:

* Cloudflare **does** cache the resource when:
    
    * The `Cache-Control` header is set to `public` and `max-age` is greater than 0.
        
    * The `Expires` header is set to a future date
        
        *(If both* `max-age` and an `Expires` header are set, Cloudflare uses `max-age`)
        
* Cloudflare **does not** cache the resource when:
    
    * The `Cache-Control` header is set to `private`, `no-store`, `no-cache`, or `max-age=0`.
        
    * The [Set-Cookie header](https://developers.cloudflare.com/cache/concepts/cache-behavior/#interaction-of-set-cookie-response-header-with-cache) exists.
        
    * The HTTP request method is anything other than a `GET`.
        

If there is no `Cache-Control` or `Expires`, CloudFlare caches responses (with varied TTL) only when the origin status code is one of `200`, `206`, `301-303`, `404` and `410`.

### Extensions’ all that matter

Those rules look great, BUT here is another specificity. Before attempting to use the rules above, Cloudflare first looks at the resource type, which it determines based on the extension and not the MIME type set in the response (`Content-Type` header). Only some extensions are “cacheable” by default.

> Cloudflare only caches based on file extension and not by MIME type. The Cloudflare CDN ***does not cache HTML or JSON by default***.

When we say extension, we talk about the last bit of the URL. To better understand:

* `derlin.ch/report.pdf` is cacheable (`.pdf` is part of the list)
    
* `derlin.ch/image.png` is cacheable (`.png` is part of the list)
    
* `derlin.ch/index.html` is not cacheable (`.html` is not part of the list)
    
* `derlin.ch/index` is NOT cacheable (no extension)
    

So, if your website is serving HTML or is modern enough to strip extensions on most pages, then nothing is cached by default, regardless of your `Cache-Control` headers.

### Other rules

There are so many more specificities and rules that I have not covered. Please refer to the documentation to know more about [cacheable size limits](https://developers.cloudflare.com/cache/concepts/default-cache-behavior/#cacheable-size-limits), [`Set-Cookie` behaviors](https://developers.cloudflare.com/cache/concepts/cache-behavior/#interaction-of-set-cookie-response-header-with-cache) and more.

## Making sense of all of this

On any modern website based on a decent framework, you usually:

* serve HTML pages without extensions
    
* have proper `Cache-Control` headers set, often by the framework itself (e.g. the Django Admin pages will always have a `Cache-Control: private, no-store, max-age=0`)
    

With this setup, and only using the default Cloudflare setup:

* your private and admin pages will never be cached, which is what you want.
    
* your images and PDF files with proper `Cache-Control: public, max-age=3600` will be cached, which is what you want.
    
* your public HTML pages with proper `Cache-Control: public, max-age=3600` headers will never be cached, which is NOT what you want.
    

So adding Cloudflare will not boost your performance that much, unless you are serving lots of images and files to your clients.

The only way to change this behavior is to set a Page Rule like the one at the beginning of the article, or a Cache Rule that similarly promotes all resources to “Eligible for cache”.

You do not take risks, as CloudFlare will respect the private directives in `Cache-Control` headers. The only difference will be that HTML and pages without extensions will be considered when applying the origin cache-control rules.

## In summary

If you want to take the most advantages of Cloudflare:

1. ensure your web application is properly setting `Cache-Control` headers
    
2. add a rule to cache everything (Page Rule or Cache Rule)
    
3. let the magic happen.
    

Skipping 2, most of your pages will never be cached.

BONUS: I was wondering about session cookies at some point. Even though I never found a proof in the docs, it seems Cloudflare is aware of all the notorious frameworks, and will never cache a page if it finds a known session cookie, such as `sessionid` (Django), `PHPSESSID` (Laravel), `JSESSIONID` (Java).
# PWA-ize: open web pages in their own window on mobile

## TL;DR

[PWA-ize](https://derlin.github.io/pwa-ize/) lets you add any web page to your mobile home screen, that opens in a dedicated browser window (not a browser tab!). This makes them feel more like regular apps, without any installation from the store.

Need an example ? Here is how dev.to behaves when added directly:

![without_pwa_open](https://cdn.hashnode.com/res/hashnode/image/upload/v1671094552420/KOZPy6_iDF.gif align="center")

and the same dev.to added through PWA-ize (Android 11):

![with_pwa_open](https://cdn.hashnode.com/res/hashnode/image/upload/v1671094554897/_PLGCiR6m.gif align="center")

In case you don't want too many shortcuts on your home screen, the [shortcuts page](https://derlin.github.io/pwa-ize/shortcuts.html) can be used as a launcher instead. It is a simple PWA page that displays a clickable list of your favorite sites.

## PWA... Kezaco ?

PWA, short for **P**rogressive **W**eb **A**pp, has been around for some time now. Paraphrasing [Mozilla's doc](https://developer.mozilla.org/en-US/docs/Web/Manifest):

> PWAs are websites that can be installed to a device’s homescreen without an app store. Unlike regular web apps with simple homescreen links or bookmarks, PWAs can be downloaded in advance and can work offline, as well as use regular Web APIs.

So in other words, when you add a PWA-enabled web page to your home screen, it will **open in a separate (and dedicated) browser window**, and feel more like a regular app. For non-PWAs, though, clicking on the shortcut will launch the default browser app and open the page on a new tab.

There is way more to it (offline mode, etc), but this separation is to me the single most helpful feature.

Since many websites are not PWAs (and prefer to force us into using a native app, ikes), I [created PWA-ize](https://derlin.github.io/pwa-ize/), which makes ANY page open in a dedicated window, PWA or not. The cherry on the cake: you can decide to open a specific page directly, instead of the index of the site (URL parameters, etc. also work).

## Testimony

Ok, I am the only one currently using it. But I love it!

I have at least a half-screen of PWA-ized shortcuts for all my day-to-day activities. For example:

*   [MeteoSuisse](https://www.meteosuisse.admin.ch/content/meteoswiss/fr/home.mobile.meteo-products--overview.html),
    
*   The page displaying the menus at work,
    
*   The page for booking my fitness classes (with some filters setup thanks to URL parameters, so it opens exactly what I need),
    
*   [dev.to](https://dev.to) (obviously),
    
*   [bookfortoday, science-fiction category](https://bookfortoday.com/science-fiction/),
    
*   ... (at least 6 more ...)
    

Let me know what you think, at please leave a ⭐ !

* * *

*(For the interested devs out there)*

## Conditions for a dedicated window

For a shortcut added to home screen to open on a dedicated window, two conditions must be met:

1.  the website should declare a [Web Manifest](https://developer.mozilla.org/en-US/docs/Web/Manifest), and
    
2.  the manifest should set the `display` property to something other than `browser` (the default).
    

Given the two conditions listed above, my idea was thus to create a manifest on the fly for a given URL.

## Changing manifests using Javascript

Manifests are referenced in a `meta` tag in the header and are only used at install time (after you click *Add to Home Screen*). It can thus be instrumented via Javascript:

```html
<link rel="manifest" id="manifest-placeholder" />
```

```js
const metaTag = document.getElementById("manifest-placeholder")
// generate the manifest
const jsonManifest = JSON.stringify({
  name: "...",
  display: "fullscreen",
  // ...
});
// attach it as base64 to the meta tag
const blob = new Blob([jsonManifest], {type: "application/json"})
metaTag.setAttribute("href", URL.createObjectURL(blob))
```

## Manifest's start URL

A manifest is tightly coupled to the page that references it: when installed, it will open the page it was downloaded from. We can tweak the URL a bit, but it must be in the same domain. It is thus not possible to simply ask the user-agent to open an unrelated URL (e.g. `https://dev.to`). Here is my workaround.

From the manifest specification:

> *The manifest's* `start_url` member is a string that represents the start URL, which is URL that the developer would prefer the user agent load when the user launches the web application (e.g., when the user clicks on the icon of the web application from a device's application menu or homescreen).

This seems promising but has a limitation: the `start_url` **is ignored if not the same origin** as the document URL (aka page defining the manifest).

Thus, it is not possible to set the `start_url` to `https://dev.to`, but it is possible to set it to some random page (in the same domain) that will *redirect* to `https://dev.to`.

This is the role of the [redirect.html](https://derlin.github.io/pwa-ize/redirect.html) page:

```xml
<html>
<head><title>PWA-ize redirect</title></head>
<body>
<div id="message"></div>
<script type="text/javascript">
// simply redirect to the given URL
const params = new URLSearchParams(window.location.search)
try {
  window.location.href = new URL(params.get("url")).href
} catch (error) {
  document.addEventListener("DOMContentLoaded", function () {
    document.getElementById("message").innerHTML = 
      `An error occurred ${error}. The url option is empty or invalid ${window.location.href}`
  })
}
</script>
</body>
</html>
```

With this, we can use in the manifest:

```json
{
  "start_url": "https://derlin.github.io/pwa-ize/redirect.html?url=https://dev.to/"
}
```

The only downside of this redirect is that since we leave the manifest's domain, the browser top bar will show again. A small price to pay I guess.

## Manifest icons

A manifest can (must) provide icons, that may be used in the home screen and on the splash screen.

There are multiple services available to grab favicons, such as Google S2, duckdcuckgo icons API, and Icon Horse. I explained the first in this article (and others in the comments) if you are interested.

%[https://dev.to/derlin/get-favicons-from-any-website-using-a-hidden-google-api-3p1e] 

The only problem is that the manifest should contain not only the link to icons but also their type and size. The type can be taken from the response's `Content-Type` header, but the size is another matter.

One possibility to get the actual size of an image is to use the `Img` like this:

```javascript
const img = new Image()
img.onload = function () {
  // here we can get the actual size !
  // not the type though ...
  console.log(`${this.src} => ${this.height}x${this.width}`)
}
img.src = `<URL of the image>`
```

I currently use Google S2 because it always returns PNG, so I don't need yet another request to get the content type (as the image trick above doesn't let you access the response headers).

Google S2 returns a 16x16 icon by default, so I use the `sz` query parameter (a size hint) to loop through common image sizes, (`16`, `128`, `256`, `512`) and add the ones I find in the manifest.

If you are interested in the actual code, have a look at `src/utils/google-s2.js`.

## PWA-ize webapp

For this small project, I decided to go with [Vue3](https://v3.vuejs.org/) and [materialize-css](materializecss.com/), with some tweaks on the color scheme.

I bootstrapped the project using `vue create`, and only had to add the `.prettierc` to get a better formatting.

I needed multiple HTML pages, since my shortcuts page needs a specific and fixed manifest. This was easy to achieve: I just used the `pages` configuration of `vue.config.js`.

To deploy to Github Pages, I have a simple Github Actions workflow, and

The [shortcuts](https://derlin.github.io/pwa-ize/shortcuts.html) page is using `localStorage` to persist the list of links.

* * *

Written with ❤️ by [derlin](https://github.com/derlin)
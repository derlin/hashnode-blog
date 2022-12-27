# Get favicons from any domain using a hidden google API

Did you know?

Google offers a secret URL that can automatically pull the favicon image of any domain. Cherries on the cake, we can ask for different sizes and the images returned are in PNG format (not ICO), meaning they will render correctly in all browsers using the `<img>` tag.

The API works using a simple GET:

```bash
https://www.google.com/s2/favicons?domain=${domain}&sz=${size}
```

The query parameters are:

*   `domain`: *mandatory*, the domain you are interested in,
    
*   `sz`: *optional*, a size hint such as `256`.
    

In case the right size is not found, it will return the default one, usually 16x16.

* * *

https://cdn.hashnode.com/res/hashnode/image/upload/v1671094570561/PC1z81oNA.png

![128x128](https://www.google.com/s2/favicons?domain=dev.to&sz=128 align="left")

https://cdn.hashnode.com/res/hashnode/image/upload/v1671094571634/-T-YhxOp1.png (nothing found for 512x512, so returns a 16x16 PNG)

![16x16 fallback](https://www.google.com/s2/favicons?domain=dev.to&sz=512 align="left")

https://cdn.hashnode.com/res/hashnode/image/upload/v1671094572625/LOC9cNPMk.png (yep, sometimes the quality is far from optimal)

![stackoverflow 128x128](https://www.google.com/s2/favicons?domain=stackoverflow.com&sz=128 align="left")

* * *

### **UPDATE: similar services**

Privacy-friendly search engine [DuckDuckGo](https://duckduckgo.com/) also has one similar to Google one: https://icons.duckduckgo.com/ip3/dev.to.ico.

Another service called [Icon Horse](https://icon.horse/) has some additional features: https://icon.horse/icon/dev.to.

Finally, there is also [favicongrabber.com](http://favicongrabber.com). The call looks like:

```bash
http://favicongrabber.com/api/grab/dev.to
```

The response is a JSON with all the available icons (use `?pretty=true` for nice json formatting):

```bash
{
  "domain": "dev.to",
  "icons": [
    {
      "sizes": "192x192",
      "src": "https://res.cloudinary.com/practicaldev/image/fetch/s--t7tVouP9--/c_limit,f_png,fl_progressive,q_80,w_192/https:/practicaldev-herokuapp-com.freetls.fastly.net/assets/devlogo-pwa-512.png"
    },
    ...
}
```

However, I got some gateway timeouts... Trying [favicongrabber.com/api/grab/stacko](http://favicongrabber.com/api/grab/stacko)[...](http://favicongrabber.com/api/grab/stackoverflow.com) currently returns:

```bash
{"error":"General API error."}
```
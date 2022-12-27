# Getting notified when a string/pattern appears on a webpage

## TL;DR

Need to wait for a page to show a specific string or pattern (e.g. in pages tailing some logs) and want to be notified, so you can go about other businesses while waiting? Use the [JS Notification API](https://developer.mozilla.org/en/docs/Web/API/notification) along with [bookmarklets](https://mcdlr.com/js-inject/) to do the trick!

## Motivation

As a developer in 2021, I spend a lot of time staring at *console outputs* from CI/CD pipelines, or *logs* tabs from kubernetes-like clusters, to either wait for some state or to catch any exception or weirdness.

This is tiresome, especially since I usually know what to expect. I would like to have some sort of notification mechanism, so I can go around other businesses and only switch my attention back when the awaited condition is met.

**Let me give an example with a hypothetical Jenkins prompt**: Let's say a prompt was defined in Jenkins (with a default timeout of 1 minute), to ask whether or not to save the output of the test after e2e tests finished. If the build and e2e tests take in average 30 minutes, it is very hard not to miss the prompt window, and you will end up restarting the pipeline often and hope this time you won't forget to monitor the test's progress. Well, this is exactly the kind of thing I faced over and over.

There are lots of extensions out there in order to get notified when something appears on a page, but they are designed to check/refresh the page at a regular interval (and in the background). Moreover, the configuration is often complex and tiresome, especially if the pattern you are looking for changes every time. Those are well suited for stock values or changing prices on Amazon, but not for my use case.

## What I want

Given a page that progressively shows text (e.g. pages tailing some logs, Jenkins *Console Output* page, ...), I want to get a notification as soon as a given text or pattern appears.

This should work at least on Mac, preferably on Chrome, but another browser would work as well. Ideally, the solution would work on all OS/browsers.

## About browser notifications

### The notification API

The [Notification API](https://developer.mozilla.org/en-US/docs/Web/API/notification) is what is used under the hood by all sites showing notifications. You should be familiar with the all-too-annoying prompt:

![notification permission prompt](https://cdn.hashnode.com/res/hashnode/image/upload/v1671094580483/m-vW6d_oYY.png align="left")

The API is simple enough. You first ask for permission to show notifications (triggering the prompt above the first time), then just create a `Notification` object:

```javascript
// In your favourite dev tools console (Chrome, Firefox, ...)
// Note: run it on a page you trust, as you will have to allow notification from this origin 
Notification.requestPermission((permission) => {
   if (permission !== 'denied') {
      new Notification("test");
   }
})
```

This can be executed directly from the devtools, or from a javascript loaded on a page.

Note that if you are using **Firefox**, [a change in version **72** blocks the popup when the permission request is not triggered by user interaction](https://blog.mozilla.org/futurereleases/2019/11/04/restricting-notification-permission-prompts-in-firefox/). Instead, a small icon will wiggle at the right of the URL bar. If you still really need the popup, there are still [some nasty workarounds](https://stackoverflow.com/a/67568418/).

### Permissions and OS/Browser support for notifications

The Notification API is supported in all major browsers for a while now.

For a notification to show, two kinds of permissions must be granted:

1.  **browser permission**: your OS needs to allow permissions from your browser. See [this article](https://www.makeuseof.com/google-chrome-notifications-not-working-fixes/#:~:text=Check%20Chrome%27s%20Notification%20Permissions) for how to do it (explaining the process for Chrome, but would be the same for other browsers).
    
2.  **website permission**: even when triggering the notification from the devtools, the notification will be attached to the page currently displayed. The latter thus need to have *show notifications* granted. See [Enabling and Disabling Browser Notifications in Various Browsers](https://support.humblebundle.com/hc/en-us/articles/360008513933-Enabling-and-Disabling-Browser-Notifications-in-Various-Browsers)
    

Still no notifications? Check out those resources:

*   [Not Getting Notifications on Google Chrome? Here Are 10 Fixes to Try](https://www.makeuseof.com/google-chrome-notifications-not-working-fixes/): for Windows and Mac, the tips can be applied to other browsers as well
    
*   [Web push notifications not showing since Chrome switched to OSX notifications](https://stackoverflow.com/questions/36383154/web-push-notifications-not-showing-since-chrome-switched-to-osx-notifications) on StackOverflow
    

| ⚠️ |
| --- |
| For some reason, notifications do not work anymore since Chrome `92.0.4515.159` on my laptops (MAC OSX Catalina // Big Sur). Fortunately, Firefox and Safari are still working fine. |

## Writing the JS

With the theory in place, I need a JS code that:

1.  waits for a given text to appear on the page (a basic search and a sleep in a loop will do the trick);
    
2.  shows a persistent notification using the Notification API once the text is found;
    
3.  *the cherry on the cake*: configures the notification so that we switch automatically to the right browser tab on click.
    

Here is a code doing exactly that (supporting regular expressions as well as text):

%[https://gist.github.com/derlin/50b02e293f8dda69eb46f25ad97c63d7#file-waitfortext-js] 

To use it, open the devtools console on any page (`⌘`+`⌥`+`i` on Mac), copy-paste it and then call `waitForText` with whatever pattern(s) you want to wait for. Here are some examples:

```javascript
// be notified when an exact string appears
waitForText("BUILD SUCCESS")
// be notified when any of multiple strings appears
waitForText(["BUILD SUCCESS", "BUILD FAILURE"])
// look for a pattern
waitForText(/BUILD (SUCCESS|FAILURE)/)
// look for patterns or strings
waitForText([/hello.*world/, "foo"])

// look for string only in part of the page
waitForText("hello", document.getElementById("out"))
// customise the notification
waitForText("world", undefined, {title: "hello"});
```

## Making the script easy to use

The above script works well, but I don't want to have to copy-paste it to the devtools console each time I need it. Fortunately, Bookmarklets can save the day!

> A [bookmarklet](https://en.wikipedia.org/wiki/Bookmarklet) is a bookmark stored in a web browser that contains JavaScript commands that add new features to the browser.

I used two tools to generate the bookmarklet URL:

1.  https://jscompress.com/ to minify my script (check *ECMAScript 2021*),
    
2.  https://mcdlr.com/js-inject/ to generate the bookmarklet link
    

The two steps above give us the following URL, that you can add as a bookmark (*new bookmark*, then paste the text below in the *URL* field):

```javascript
javascript:(function(){function sleep(a){return new Promise(b=>setTimeout(b,a))}function toArrayOfPatterns(a){return(Array.isArray(a)?a:[a]).map(a=>%22string%22==typeof a?new RegExp(a):a)}async function isNotificationPermissionGranted(){return new Promise(a=>{%22granted%22===Notification.permission?a(!0):%22default%22===Notification.permission?(console.log(%22Requesting notification permission.%22),console.log(%22%25cFROM FIREFOX 72 onwards%25c\nIt is not possible to trigger the permission prompt without user interaction. A little icon should have appeared on the left of the URL bar though. Click on it to grant permission !\nSee https://blog.mozilla.org/futurereleases/2019/11/04/restricting-notification-permission-prompts-in-firefox/%22,%22font-weight: bold; color: red%22,%22%22),Notification.requestPermission().then(()=>{%22granted%22===Notification.permission?a(!0):a(!1)})):((()=>console.log(%22%25cYou didn%27t grant notification permission for this page. Aborting.%22,%22font-weight: bold; color: red%22))(),a(!1))})}async function waitForText(a,b,c){if(!(await isNotificationPermissionGranted()))return;const d=b??document.body,e=toArrayOfPatterns(a);for(console.log(`Waiting for any of ${e}.`,%22Type %27window.stopWaiting = true%27 on the developer console to stop the wait, or reload the page.%22);!window.stopWaiting;){console.log(%22...%22);let a=e.find(a=>a.test(d.innerHTML));if(a){const b=a.exec(d.innerHTML)[0];console.log(`${b} found. Showing notification.`);let e=Object.assign({title:%22Found!%22,body:`found %22${b}%22 in ${document.title}`,requireInteraction:!0},c),f=new Notification(e.title,e);return void(f.onclick=function(){window.parent.parent.focus(),f.close()})}await sleep(5e3)}console.log(%22WaitForText aborted.%22),window.stopWaiting=!1}window.stopWaiting=!1,window.waitForText=waitForText;})();
```

With this new bookmarklet, the process is now:

1.  open a page,
    
2.  click on the bookmark,
    
3.  open the console,
    
4.  call `waitForText(...)`.
    

Done !

## Going further

The script in this article is very generic. If you always need to wait for the same pattern(s), you can easily add some exports at the end of the script and re-generate the bookmarklet. For example:

```javascript
// at the end of the script
window.waitForEndOfBuild = async function() {
    await waitForText(/BUILD (SUCCESS|FAILURE)/);
};
// ... more functions ...
```

If you do not want to open the console each time, you can also call `waitForText` directly in the script, thus triggering the wait as soon as you click the bookmarklet:

```javascript
// at the end of the script
await waitForText(/BUILD (SUCCESS|FAILURE)/);
```

The possibilities are endless.

* * *

Written with ❤ by [derlin](https://github.com/derlin)
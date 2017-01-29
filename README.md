#Debugging an Electron App
After reading the official documentation on Electron's website, I was left with a number of questions as to how Electron worked under the surface and how to effectively debug an app. This document describes some techniques on how to debug an Electron app on a Mac although the same concepts would apply for Windows and Linux.

Before diving into how to debug an Electron app, it helps to understand the general architecture of one from a different perspective than that presented on Electron's website. By better understanding this, you can make a better decision on how to architect your app which in turn will play a role in how you debug it.

## Basic Structure of an Electron App
An Electron app is essentially made up of three components: Web pages, a main process and Node.js.

* **Web Pages**: One or more web pages that provide the UI for your app. You don't even really need web pages. And it's also possible to create hidden pages that run scripts in the background that can be used to process lengthy tasks.

* **Main Process**: Consider this the startup script. It's normally in the project's root folder and named main.js. Among other things, it's used to initialize your app, perform application-wide functionality like creating and handling menus and creating and managing web pages. It is the only place where web pages can be created. You will see sample code on the Electron site that appears to show the creation of web pages even within other pages. But that is done by web pages calling on the main process to handle the creation of those pages. The main process also acts as a proxy between pages allowing them to communicate with each other. Because web pages are essentially the same thing as pages shown in different tabs in the Chrome browser, they cannot communicate with each other directly. There are various ways that they can communicate with each other indirectly and going through the main process is just one of them.

* **Node.js**: When you start Electron, what is really happening is that Node is used to run the main process (main.js). Because Node also provides APIs that interact with the OS, Electron makes Node available to any part of your app as well. It is available to any code in main.js and any code in web pages.

Debugging an Electron app is essentially debugging web pages as you would normally do in Chrome by bringing up the DevTools which can be accessed by the shortcut keys alt-cmd-i (on a Mac). Debugging the main process however is virtually no different than debugging a Node.js app. Where things get tricky is in the actual details of how to setup the debugging for main process and that for web pages.

One of the biggest confusions I had in the beginning was trying to determine where to put my application's own code. Does it belong in the same process where the main process is located or should it be written as javascript modules that are included in web pages as they normally are through a script element or by some other means.

The answer to that is that almost all of your app's code should be placed in javascript code modules that make up the web pages. The main process should stick to just doing initialization stuff like launching your main web page, creating windows when requested and handling global stuff like menus. The actual initialization of your app's main web page should however be placed in a javascript module that is part of the main web page that initially gets launched from the main process.

###The Renderer Process
The Electron docs make a lot of referrals to what it calls "renderer processes". A renderer process is a process run by the Chromium V8 render engine that creates a web page and runs any javascript associated with it. In Chrome, every tab is essentially running a renderer process separate from all other tabs. If one tab crashes, the other ones will not (at least in theory).

As far as your app is concerned, when you create a web page and include javascript to be part of it (by using the script element for example), you can consider both the web page content (the actual HTML markup) and any javascript that is part of it as a "renderer process".

###Use a Single Page App or Multiple Pages?
Web applications can be designed in such a way that the entire application runs within a single page (such as Gmail) or it can be made up of multiple web pages where more pages can act like popups. There is no right or wrong way of doing this. However, web apps have evolved over the years and have gone from multiple pages to single pages. There are several good reasons to go with a single page app:

* **Performance**: Creating popups is resource intensive and slow. Each popup window is no different than just another tab in Chrome even if doesn't have a frame and title bar. A single web page can just add/remove stuff to the DOM and the UI updates very quickly. There really is no need to create modal dialog boxes from web pages when you can just as well create a window on top of your main window that acts like a normal modal window.

* **Communication**: If a popup window (running as a separate page) needs to communicate back with the main page, it needs to rely on a mechanism that isn't exactly native to web development. But if the popup is part of the same single page app, it can easily interact with the main app and provide feedback, especially when the popup is closed.

* **Popup Blockers**: Popup blockers, a.k.a "ad blockers" can easily prevent one page from launching another page making your app break for those who use ad blocking software.

* **Debugging**: It is much easier and manageable to debug a single page app than one with multiple pages. You also have access to all the global data in a single page app which makes debugging easier too.

* **Appearance**: Let's face it, single web pages when done properly can look great and better than an app made up of multiple pages.

##Debugging a Web Page
The Electron app entitled _Electron API Demos is used to illustrate the debugging techniques discussed here. You can download the app at:

[https://github.com/electron/electron-api-demos](https://github.com/electron/electron-api-demos)

When the app is running, you can press alt-cmd-i to bring up the developer tools (or DevTools for short), which is the same DevTools used in Chrome. You can then set breakpoints and inspect variables. Breakpoints however will only be hit **after** DevTools has already been loaded. If you set a breakpoint and then restart your app, they will not be hit because by default DevTools does not load when your app starts, although a way to do this is described below. If you don't want to restart your app, you can open up the DevTools console pane and type:

```
location.reload(true)
```

This will force the page to reload and stop at any breakpoints where they are encountered during execution. Most of the time this will probably suffice but there will be cases where you need a breakpoint to be hit without reloading the web page. If your app consists of multiple pages and one page (such as the main page) opens up another page and needs to pass data to it, you might end up losing that data if you just reload the page, depending on what sort of mechanism you use for passing data. But a more important issue is that by default, DevTools will not be opened up on the secondary page, so even if you have breakpoints set or even use a debugger statement, the code will not be suspended. As was already mentioned, breakpoints can only be hit **after** DevTools has loaded. And even then, DevTools needs a short amount of time to attach itself to the currently running DOM and javascript.

One solution is to programmatically open DevTools when the page is created and delay executing your script for a fixed duration. Open the the file _process-crash.js_ and add the openDevTools method as follows:

```javascript
const BrowserWindow = require('electron').remote.BrowserWindow
const dialog = require('electron').remote.dialog

const path = require('path')

const processCrashBtn = document.getElementById('process-crash')

processCrashBtn.addEventListener('click', function (event) {
  const crashWinPath = path.join('file://', __dirname, '../../sections/windows/process-crash.html')
  let win = new BrowserWindow({ width: 400, height: 320 });
  win.openDevTools();

  win.webContents.on('crashed', function () {
    const options = {
      type: 'info',
      title: 'Renderer Process Crashed',
      message: 'This process has crashed.',
      buttons: ['Reload', 'Close']
    }
    dialog.showMessageBox(options, function (index) {
      if (index === 0) win.reload()
      else win.close()
    })
  })

  win.on('close', function () { win = null })
  win.loadURL(crashWinPath)
  win.show()
})
```
This code is loaded in crash-hang.html which in turn is loaded in index.html. So effectively, this code is executed from the main web page (index.html). To execute this code, on the main web page, go to the item in the navigation pane labeled **_Handling window crashes and hangs_** and expand the item in the right pane labeled **_Relaunch window after the process crashes_** and then click on the **_View Demo_** button.

The popup page containing process-crash.html is loaded and the DevTools is shown. The html content in process-crash.html including any javascript attached to it is part of what is referred to as the _renderer process__.

In order to get DevTools to break on a breakpoint when process-crash.html is loaded, we need to make some modifications. Download jQuery and store it in the script folder. You can download jQuery at:

[https://code.jquery.com/jquery-3.1.1.min.js](https://code.jquery.com/jquery-3.1.1.min.js)

Create a javascript file in the script folder and name it background.js and add the following Javascript code to it:

```javascript
debugger;
console.log("This is a breakpoint in my Electron app");
```

The ```debugger;``` statement will cause the debugger to stop on this line of code when it is encountered. Remember, it will only stop **after** DevTools has already launched and had some time to attach to the web page and its Javascript. In process-crash.html, add the following code to the end of the html:

```javascript
<script>
    var $ = global.jQuery = require('../../script/jquery-3.1.1.min.js');

    $(document).ready(function () {

        setTimeout(function () {
            $.getScript("../../script/background.js", function (data, textStatus, jqxhr) {
            });
        }, 500);
    });

</script>
```
The jQuery $.get method is used to retrieve the script in background.js after a delay of 500 ms. The delay is needed to let DevTools attach itself to the html content and its Javascript. You may need to play around with the time delay. If the delay is too short, DevTools will not have enough time to attach and the debugger statement will not be caught. You can start out with one second and just decrease the time by 100 ms increments until you find the amount of time needed for DevTools to attach.

Since this is for debugging purposes, in a production app, you probably don't want the delay. There are different techniques you can use to bypass the $.get method in a production app or just leave it in but have the delay time set to zero. Keep in mind though that Javascript does not provide any sort of conditional "compiling" where flags can be set to distinguish between debug and release versions of an app. It really is up to you to roll your own code in determining what to include or exclude from your release versions. One solution is to use the Gradle which is a popular build system for building Android app, although it is heavily geared for Java based apps. Still, Gradle isn't specific to Java or Android but can be used to build any kind of project. There is even one Gradle plugin for Javascript available in Github. Essentially, Gradle can be used to swap in or out code for various kinds of builds (debug, release, testing, etc).

##Debugging the Main Process


#Debugging an Electron App
After reading the official documentation on Electron's website, I was left with a number of questions as to how Electron worked under the surface and how to effectively debug an app. This document describes some techniques on how to debug an Electron app on a Mac although the same concepts would apply for Windows and Linux.

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


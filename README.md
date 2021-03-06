Overview
========

[![Build Status](https://travis-ci.org/swisspol/GCDWebServer.svg?branch=master)](https://travis-ci.org/swisspol/GCDWebServer)

GCDWebServer is a modern and lightweight GCD based HTTP 1.1 server designed to be embedded in OS X & iOS apps. It was written from scratch with the following goals in mind:
* Elegant and easy to use architecture with only 4 core classes: server, connection, request and response (see "Understanding GCDWebServer's Architecture" below)
* Well designed API with fully documented headers for easy integration and customization
* Entirely built with an event-driven design using [Grand Central Dispatch](http://en.wikipedia.org/wiki/Grand_Central_Dispatch) for best performance and concurrency
* No dependencies on third-party source code
* Available under a friendly [New BSD License](LICENSE)

Extra built-in features:
* Minimize memory usage with disk streaming of large HTTP request or response bodies
* Parser for [web forms](http://www.w3.org/TR/html401/interact/forms.html#h-17.13.4) submitted using "application/x-www-form-urlencoded" or "multipart/form-data" encodings (including file uploads)
* [JSON](http://www.json.org/) parsing and serialization for request and response HTTP bodies
* [Chunked transfer encoding](https://en.wikipedia.org/wiki/Chunked_transfer_encoding) for request and response HTTP bodies
* [HTTP compression](https://en.wikipedia.org/wiki/HTTP_compression) with gzip for request and response HTTP bodies
* [HTTP range](https://en.wikipedia.org/wiki/Byte_serving) support for requests of local files
* [Basic](https://en.wikipedia.org/wiki/Basic_access_authentication) and [Digest Access](https://en.wikipedia.org/wiki/Digest_access_authentication) authentications for password protection
* Automatically handle transitions between foreground, background and suspended modes in iOS apps

Included extensions:
* [GCDWebUploader](GCDWebUploader/GCDWebUploader.h): subclass of ```GCDWebServer``` that implements an interface for uploading and downloading files using a web browser
* [GCDWebDAVServer](GCDWebDAVServer/GCDWebDAVServer.h): subclass of ```GCDWebServer``` that implements a class 1 [WebDAV](https://en.wikipedia.org/wiki/WebDAV) server (with partial class 2 support for OS X Finder)

What's not supported (but not really required from an embedded HTTP server):
* Keep-alive connections
* HTTPS

Requirements:
* OS X 10.7 or later (x86_64)
* iOS 5.0 or later (armv7, armv7s or arm64)

Getting Started
===============

Download or check out the [latest release](https://github.com/swisspol/GCDWebServer/releases) of GCDWebServer then add the entire "GCDWebServer" subfolder to your Xcode project. If you intend to use one of the extensions like GCDWebDAVServer or GCDWebUploader, add these subfolders as well.

Alternatively, you can install GCDWebServer using [CocoaPods](http://cocoapods.org/) by simply adding this line to your Xcode project's Podfile:
```
pod "GCDWebServer", "~> 2.0"
```
If you want to use GCDWebUploader, use this line instead:
```
pod "GCDWebServer/WebUploader", "~> 2.0"
```
Or this line for GCDWebDAVServer:
```
pod "GCDWebServer/WebDAV", "~> 2.0"
```

Hello World
===========

These code snippets show how to implement a custom HTTP server that runs on port 8080 and returns a "Hello World" HTML page to any request. Since GCDWebServer uses GCD blocks to handle requests, no subclassing or delegates are needed, which results in very clean code.

**OS X version (command line tool):**
```objectivec
#import "GCDWebServer.h"
#import "GCDWebServerDataResponse.h"

int main(int argc, const char* argv[]) {
  @autoreleasepool {
    
    // Create server
    GCDWebServer* webServer = [[GCDWebServer alloc] init];
    
    // Add a handler to respond to GET requests on any URL
    [webServer addDefaultHandlerForMethod:@"GET"
                             requestClass:[GCDWebServerRequest class]
                             processBlock:^GCDWebServerResponse *(GCDWebServerRequest* request) {
      
      return [GCDWebServerDataResponse responseWithHTML:@"<html><body><p>Hello World</p></body></html>"];
      
    }];
    
    // Use convenience method that runs server on port 8080 until SIGINT received (i.e. Ctrl-C in Terminal)
    [webServer runWithPort:8080 bonjourName:nil];
    
  }
  return 0;
}
```

**iOS version:**
```objectivec
#import "GCDWebServer.h"
#import "GCDWebServerDataResponse.h"

static GCDWebServer* _webServer = nil;  // This should really be an ivar of your application's delegate class

- (BOOL)application:(UIApplication*)application didFinishLaunchingWithOptions:(NSDictionary*)launchOptions {
  
  // Create server
  _webServer = [[GCDWebServer alloc] init];
  
  // Add a handler to respond to GET requests on any URL
  [_webServer addDefaultHandlerForMethod:@"GET"
                            requestClass:[GCDWebServerRequest class]
                            processBlock:^GCDWebServerResponse *(GCDWebServerRequest* request) {
    
    return [GCDWebServerDataResponse responseWithHTML:@"<html><body><p>Hello World</p></body></html>"];
    
  }];
  
  // Start server on port 8080
  [_webServer startWithPort:8080 bonjourName:nil];
  
  return YES;
}
```

Web Based Uploads in iOS Apps
=============================

GCDWebUploader is a subclass of ```GCDWebServer``` that provides a ready-to-use HTML 5 file uploader & downloader. This lets users upload, download, delete files and create directories from a directory inside your iOS app's sandbox using a clean user interface in their web browser.

Simply instantiate and run a ```GCDWebUploader``` instance then visit ```http://{YOUR-IOS-DEVICE-IP-ADDRESS}/``` from your web browser:

```objectivec
#import "GCDWebUploader.h"

static GCDWebUploader* _webUploader = nil;  // This should really be an ivar of your application's delegate class

- (BOOL)application:(UIApplication*)application didFinishLaunchingWithOptions:(NSDictionary*)launchOptions {
  NSString* documentsPath = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
  _webUploader = [[GCDWebUploader alloc] initWithUploadDirectory:documentsPath];
  [_webUploader start];
  NSLog(@"Visit %@ in your web browser", _webUploader.serverURL);
  return YES;
}
```

WebDAV Server in iOS Apps
=========================

GCDWebDAVServer is a subclass of ```GCDWebServer``` that provides a class 1 compliant [WebDAV](https://en.wikipedia.org/wiki/WebDAV) server. This lets users upload, download, delete files and create directories from a directory inside your iOS app's sandbox using any WebDAV client like [Transmit](https://panic.com/transmit/) (Mac), [ForkLift](http://binarynights.com/forklift/) (Mac) or [CyberDuck](http://cyberduck.io/) (Mac / Windows).

GCDWebDAVServer should also work with the [OS X Finder](http://support.apple.com/kb/PH13859) as it is partially class 2 compliant (but only when the client is the OS X WebDAV implementation).

Simply instantiate and run a ```GCDWebDAVServer``` instance then connect to ```http://{YOUR-IOS-DEVICE-IP-ADDRESS}/``` using a WebDAV client:

```objectivec
#import "GCDWebDAVServer.h"

static GCDWebDAVServer* _davServer = nil;  // This should really be an ivar of your application's delegate class

- (BOOL)application:(UIApplication*)application didFinishLaunchingWithOptions:(NSDictionary*)launchOptions {
  NSString* documentsPath = [NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES) firstObject];
  _davServer = [[GCDWebDAVServer alloc] initWithUploadDirectory:documentsPath];
  [_davServer start];
  NSLog(@"Visit %@ in your WebDAV client", _davServer.serverURL);
  return YES;
}
```

Serving a Static Website
========================

GCDWebServer includes a built-in handler that can recursively serve a directory (it also lets you control how the ["Cache-Control"](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.9) header should be set):

```objectivec
#import "GCDWebServer.h"

int main(int argc, const char* argv[]) {
  @autoreleasepool {
    
    GCDWebServer* webServer = [[GCDWebServer alloc] init];
    [webServer addGETHandlerForBasePath:@"/" directoryPath:NSHomeDirectory() indexFilename:nil cacheAge:3600 allowRangeRequests:YES];
    [webServer runWithPort:8080];
    [webServer release];  // Remove if using ARC
    
  }
  return 0;
}
```

Using GCDWebServer
==================

You start by creating an instance of the ```GCDWebServer``` class. Note that you can have multiple web servers running in the same app as long as they listen on different ports.

Then you add one or more "handlers" to the server: each handler gets a chance to handle an incoming web request and provide a response. Handlers are called in a LIFO queue, so the latest added handler overrides any previously added ones.

Finally you start the server on a given port.

Understanding GCDWebServer's Architecture
=========================================

GCDWebServer's architecture consists of only 4 core classes:
* [GCDWebServer](GCDWebServer/Core/GCDWebServer.h) manages the socket that listens for new HTTP connections and the list of handlers used by the server.
* [GCDWebServerConnection](GCDWebServer/Core/GCDWebServerConnection.h) is instantiated by ```GCDWebServer``` to handle each new HTTP connection. Each instance stays alive until the connection is closed. You cannot use this class directly, but it is exposed so you can subclass it to override some hooks.
* [GCDWebServerRequest](GCDWebServer/Core/GCDWebServerRequest.h) is created by the ```GCDWebServerConnection``` instance after HTTP headers have been received. It wraps the request and handles the HTTP body if any. GCDWebServer comes with [several subclasses](GCDWebServer/Requests) of ```GCDWebServerRequest``` to handle common cases like storing the body in memory or stream it to a file on disk.
* [GCDWebServerResponse](GCDWebServer/Core/GCDWebServerResponse.h) is created by the request handler and wraps the response HTTP headers and optional body. GCDWebServer comes with [several subclasses](GCDWebServer/Responses) of ```GCDWebServerResponse``` to handle common cases like HTML text in memory or streaming a file from disk.

Implementing Handlers
=====================

GCDWebServer relies on "handlers" to process incoming web requests and generating responses. Handlers are implemented with GCD blocks which makes it very easy to provide your owns. However, they are executed on arbitrary threads within GCD so __special attention must be paid to thread-safety and re-entrancy__.

Handlers require 2 GCD blocks:
* The ```GCDWebServerMatchBlock``` is called on every handler added to the ```GCDWebServer``` instance whenever a web request has started (i.e. HTTP headers have been received). It is passed the basic info for the web request (HTTP method, URL, headers...) and must decide if it wants to handle it or not. If yes, it must return a new ```GCDWebServerRequest``` instance (see above) created with this info. Otherwise, it simply returns nil.
* The ```GCDWebServerProcessBlock``` is called after the web request has been fully received and is passed the ```GCDWebServerRequest``` instance created at the previous step. It must return a ```GCDWebServerResponse``` instance (see above) or nil on error, which will result in a 500 HTTP status code returned to the client. It's however recommended to return an instance of [GCDWebServerErrorResponse](GCDWebServer/Responses/GCDWebServerErrorResponse.h) on error so more useful information can be returned to the client.

Note that most methods on ```GCDWebServer``` to add handlers only require the ```GCDWebServerProcessBlock``` as they already provide a built-in ```GCDWebServerMatchBlock``` e.g. to match a URL path with a Regex.

GCDWebServer & Background Mode for iOS Apps
===========================================

When doing networking operations in iOS apps, you must handle carefully [what happens when iOS puts the app in the background](https://developer.apple.com/library/ios/technotes/tn2277/_index.html). Typically you must stop any network servers while the app is in the background and restart them when the app comes back to the foreground. This can become quite complex considering servers might have ongoing connections when they need to be stopped.

Fortunately, GCDWebServer does all of this automatically for you:
- GCDWebServer begins a [background task](https://developer.apple.com/library/ios/documentation/iphone/conceptual/iphoneosprogrammingguide/ManagingYourApplicationsFlow/ManagingYourApplicationsFlow.html) whenever the first HTTP connection is opened and ends it only when the last one is closed. This prevents iOS from suspending the app after it goes in the background, which would immediately kill HTTP connections to the client.
 - While the app is in the background, as long as new HTTP connections keep being initiated, the background task will continue to exist and iOS will not suspend the app (unless under sudden and unexpected memory pressure).
 - If the app is still in the background when the last HTTP connection is closed, GCDWebServer will suspend itself and stop accepting new connections as if you had called ```-stop``` (this behavior can be disabled with the ```GCDWebServerOption_AutomaticallySuspendInBackground``` option).
- If the app goes in the background while no HTTP connections are opened, GCDWebServer will immediately suspend itself and stop accepting new connections as if you had called ```-stop``` (this behavior can be disabled with the ```GCDWebServerOption_AutomaticallySuspendInBackground``` option).
- If the app comes back to the foreground and GCDWebServer had been suspended, it will automatically resume itself and start accepting again new HTTP connections as if you had called ```-start```.

HTTP connections are often initiated in batches (or bursts), for instance when loading a web page with multiple resources. This makes it difficult to accurately detect when the *very last* HTTP connection has been closed: it's possible 2 consecutive HTTP connections part of the same batch would be separated by a small delay instead of overlapping. It would be bad for the client if GCDWebServer suspended itself right in between. The ```GCDWebServerOption_ConnectedStateCoalescingInterval``` option solves this problem elegantly by forcing GCDWebServer to wait some extra delay before performing any action after the last HTTP connection has been closed, just in case a new HTTP connection is initiated within this delay.

Advanced Example 1: Implementing HTTP Redirects
===============================================

Here's an example handler that redirects "/" to "/index.html" using the convenience method on ```GCDWebServerResponse``` (it sets the HTTP status code and "Location" header automatically):

```objectivec
[self addHandlerForMethod:@"GET"
                     path:@"/"
             requestClass:[GCDWebServerRequest class]
             processBlock:^GCDWebServerResponse *(GCDWebServerRequest* request) {
    
  return [GCDWebServerResponse responseWithRedirect:[NSURL URLWithString:@"index.html" relativeToURL:request.URL]
                                          permanent:NO];
    
}];
```

Advanced Example 2: Implementing Forms
======================================

To implement an HTTP form, you need a pair of handlers:
* The GET handler does not expect any body in the HTTP request and therefore uses the ```GCDWebServerRequest``` class. The handler generates a response containing a simple HTML form.
* The POST handler expects the form values to be in the body of the HTTP request and percent-encoded. Fortunately, GCDWebServer provides the request class ```GCDWebServerURLEncodedFormRequest``` which can automatically parse such bodies. The handler simply echoes back the value from the user submitted form.

```objectivec
[webServer addHandlerForMethod:@"GET"
                          path:@"/"
                  requestClass:[GCDWebServerRequest class]
                  processBlock:^GCDWebServerResponse *(GCDWebServerRequest* request) {
  
  NSString* html = @" \
    <html><body> \
      <form name=\"input\" action=\"/\" method=\"post\" enctype=\"application/x-www-form-urlencoded\"> \
      Value: <input type=\"text\" name=\"value\"> \
      <input type=\"submit\" value=\"Submit\"> \
      </form> \
    </body></html> \
  ";
  return [GCDWebServerDataResponse responseWithHTML:html];
  
}];

[webServer addHandlerForMethod:@"POST"
                          path:@"/"
                  requestClass:[GCDWebServerURLEncodedFormRequest class]
                  processBlock:^GCDWebServerResponse *(GCDWebServerRequest* request) {
  
  NSString* value = [[(GCDWebServerURLEncodedFormRequest*)request arguments] objectForKey:@"value"];
  NSString* html = [NSString stringWithFormat:@"<html><body><p>%@</p></body></html>", value];
  return [GCDWebServerDataResponse responseWithHTML:html];
  
}];
```

Advanced Example 3: Serving a Dynamic Website
=============================================

GCDWebServer provides an extension to the ```GCDWebServerDataResponse``` class that can return HTML content generated from a template and a set of variables (using the format ```%variable%```). It is a very basic template system and is really intended as a starting point to building more advanced template systems by subclassing ```GCDWebServerResponse```.

Assuming you have a website directory in your app containing HTML template files along with the corresponding CSS, scripts and images, it's pretty easy to turn it into a dynamic website:

```objectivec
// Get the path to the website directory
NSString* websitePath = [[NSBundle mainBundle] pathForResource:@"Website" ofType:nil];

// Add a default handler to serve static files (i.e. anything other than HTML files)
[self addGETHandlerForBasePath:@"/" directoryPath:websitePath indexFilename:nil cacheAge:3600 allowRangeRequests:YES];

// Add an override handler for all requests to "*.html" URLs to do the special HTML templatization
[self addHandlerForMethod:@"GET"
                pathRegex:@"/.*\.html"
             requestClass:[GCDWebServerRequest class]
             processBlock:^GCDWebServerResponse *(GCDWebServerRequest* request) {
    
    NSDictionary* variables = [NSDictionary dictionaryWithObjectsAndKeys:@"value", @"variable", nil];
    return [GCDWebServerDataResponse responseWithHTMLTemplate:[websitePath stringByAppendingPathComponent:request.path]
                                                    variables:variables];
    
}];

// Add an override handler to redirect "/" URL to "/index.html"
[self addHandlerForMethod:@"GET"
                     path:@"/"
             requestClass:[GCDWebServerRequest class]
             processBlock:^GCDWebServerResponse *(GCDWebServerRequest* request) {
    
    return [GCDWebServerResponse responseWithRedirect:[NSURL URLWithString:@"index.html" relativeToURL:request.URL]
                                            permanent:NO];
    
];

```

Final Example: File Downloads and Uploads From iOS App
======================================================

GCDWebServer was originally written for the [ComicFlow](http://itunes.apple.com/us/app/comicflow/id409290355?mt=8) comic reader app for iPad. It allow users to connect to their iPad with their web browser over WiFi and then upload, download and organize comic files inside the app.

ComicFlow is [entirely open-source](https://github.com/swisspol/ComicFlow) and you can see how it uses GCDWebServer in the [WebServer.h](https://github.com/swisspol/ComicFlow/blob/master/Classes/WebServer.h) and [WebServer.m](https://github.com/swisspol/ComicFlow/blob/master/Classes/WebServer.m) files.

---
layout: project
title: IHK-ResultsNotifier
date: 14 Jun 2018
screenshot:
  src: /assets/img/projects/IHK-ResultsNotifier/srcset@0,25x.jpg
  srcset:
    1920w: /assets/img/projects/IHK-ResultsNotifier/srcset@1x.jpg
    960w: /assets/img/projects/IHK-ResultsNotifier/srcset@0,5x.jpg
    480w: /assets/img/projects/IHK-ResultsNotifier/srcset@0,25x.jpg
accent_color: '#2a5f96'
accent_image:
  background: 'linear-gradient(to bottom,#2a5f96 0%,#2a5f96 30%,#5689bf 50%,#6d97c4 70%,#9dc2ea 100%)'
  overlay:    false

caption: Displays the exam results and notifies you whenever the results are available.

description: >
  Displays the exam results and notifies you whenever the results are available.

links:
  - title: Download
    url: https://github.com/giladreich/IHK-ResultsNotifier
---

# IHK Results Notifier

A simple tool to notify you whenever the results on the official website of the IHK exams has been updated.
When the new results are available, the tool will make a sound and will bring it on top of all windows, so you'll notice the new results.


# Motivation

It started as a joke when one of the classmates wrote a script on the browser to automatically refresh the page every X time, because we had to wait for the exam results from the IHK for months and we 
never knew when the results will be available.
Then I felt like getting a little bit creative and practice my C# skills by writing a multi-threaded client application to authenticate the user in order to send request to the IHK server 
every X time in order to check for new results and this is what came up from it.

## The Cookies Trick For Authenticating a User

In order to get the idea, I'll be using curl to demonstrate the authentication process. <br/><br/>

When we visit the login page(`BB_auszubildende.jsp`), the server creates for us 2 cookies that will be used later in order to check that we are not some kind of a robot and that we actually visited the website. <br/>
So before we posting the login form, we need to emit a website visit to the following pages: <br/>
```sh
curl -I "https://apps.ihk-berlin.de/tibrosBB/BB_auszubildende.jsp"

curl -I "https://apps.ihk-berlin.de/favicon.ico"
```

At this point, the server sent us 2 cookies that we'll need to include in the header of the POST request when we posting the login form(Note that the order of the cookies is important):
```sh
# Where REPLACEME_*, add the appropriate information:
curl -v -X POST "https://apps.ihk-berlin.de/tibrosBB/azubiHome.jsp" 
-H "Cookie: TSESSIONID=REPLACEME_SESSION1; TSESSIONID=REPLACEME_SESSION2" 
-d "login=REPLACEME_USER&pass=REPLACEME_PASS&anmelden="
```

IMPORTANT NOTE: If the server accepted our POST request and successfully identified us, it will send us back a new cookie as a response, otherwise, we get no cookie back. <br/>
We'll treat the new cookie from the response as: `REPLACEME_SESSION3`(I'll explain later why). <br/><br/>

Now that we're logged in, the next step is to try and access the pages that requires authentication and see if we get OK from the server.

For our purpose, we need to retrieve the exam results page, which is `azubiErgebnisse.jsp?id=1`, but here it's get a little bit tricky, as if we'll try and directly visit 
the results page, without first visiting the `azubiPruef.jsp`, the server will destroy our session and log us out automatically.
The way to access the results page, we use the following commands in the following order:
```sh
# Emit a website visit in order to let the server confirm that we visited this page before we go deep further:
curl "https://apps.ihk-berlin.de/tibrosBB/azubiPruef.jsp" 
-H "Cookie: TSESSIONID=REPLACEME_SESSION3; TSESSIONID=REPLACEME_SESSION2"

# At this point we are safe to visit this page:
curl "https://apps.ihk-berlin.de/tibrosBB/azubiErgebnisse.jsp?id=1" 
-H "Cookie: TSESSIONID=REPLACEME_SESSION3; TSESSIONID=REPLACEME_SESSION2"
```

Note that now every request we send, we always including the 2 cookies in the request-header for any type of request, because that's how the server identifies our login-session and that the user is currently authenticated/loged-on.<br/>

Also note that this can be easily done by creating a cookie jar: `curl --cookie-jar` or `curl -c`, but to have a better understanding I show the steps in the manual way.

In C# we simply bind a `CookieContainer` to an `HttpClientHandler` that will be used in `HttpClient` and it saves us all the trouble, but we still need to visit the pages in the right order to get this right :)



## Demo

### Login Interface

![Login Interface](/assets/img/projects/IHK-ResultsNotifier/pictures/login_window.png)

![Login Interface](/assets/img/projects/IHK-ResultsNotifier/pictures/login_window_typing.png)

### Main Window Interface

![Main Window Interface1](/assets/img/projects/IHK-ResultsNotifier/pictures/main_window1.png)

#### Listening for Results:

![Main Window Interface2](/assets/img/projects/IHK-ResultsNotifier/pictures/main_window2.png)

#### Found Results:

![Main Window Interface3](/assets/img/projects/IHK-ResultsNotifier/pictures/main_window3.png)

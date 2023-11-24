---
title: A Minimal(ish?) Example of Using Session Cookies in SolidStart
author: Christopher Belanger
date: '2023-11-24'
slug: a-minimal-ish-example-of-using-session-cookies-in-solidstart
categories: []
tags: []
subtitle: ''
summary: ''
authors: []
lastmod: '2023-11-24T13:28:27-05:00'
featured: no
image:
  caption: ''
  focal_point: ''
  preview_only: no
projects: []
---


This is a minimal working example showing how you can use session cookies to store persistent user data with Solid Start.

The code itself is heavily commented and is [available on GitHub](https://github.com/chris31415926535/solidstart-sessioncookie-min-example/), and I hope in conjunction with the data dump below it helps somebody out.

# What are cookies, and why?

Here's a quick run-down of the basics, but [you can read a more detailed and technical discussion cookies here.](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies)

- **HTTP is stateless**
  - When you request a web page from a server, by default the server has no idea who you are and no "memory."
  - But much of the time we want to persist data between browsing sessions or page loads: we want the server to recognize us.
  - If we only have a little bit of data that needs to be saved, one way to do this is with cookies.
- **Cookies**
  - A "cookie" is a small piece of data that's saved by a user's web browser and is associated with a particular web domain.
  - So cookies can be used to store data as a user moves between pages of a web application, or to save data between visits to a web site.
  - Everybody uses the example of online shopping carts, and that's a good one, but it could also store other site-relevant preferences or settings.
  - Cookies can have a number of attributes, including expiration dates, and are limited to storing about 4kb per web domain.
  - The browser sends to cookie along with every HTTP request, so the server can do stuff with it.
- **Session Cookies?**
  - There sometimes seems to be ambiguity about what a "session cookie" is.
  - In one sense, session cookies are used to store persistent data about a user's current browsing session.
  - In another sense, "session cookies" are cookies that have no expiration date and are deleted when the current "browsing session" ends.
  - Unfortunately there's no standard definition of "browsing session," and some browsers may retain session cookies indefinitely.
  - Also, SolidStart _encrypts_ the data in session cookies, so the data contained is not (super easily) visible to an attacker even if they get ahold of your cookie somehow.
  - _For our purposes here, a session cookie is a cookie that stores encrypted data about a user's browsing session._

# What is SolidStart, and why?

SolidStart is a meta-framework for building full-stack web applications that's based on SolidJS.

- [Read more about SolidStart here.](https://start.solidjs.com/getting-started/what-is-solidstart)
- [Read more about SolidJS here.](https://docs.solidjs.com/)

# How do we set session cookies in a SolidStart app?

To get session cookies to work in SolidStart, you have to put a few different pieces together in just the right way. And while the docs describe each of those bits individually, a few of the concepts weren't quite spelled out and I couldn't find a minimal end-to-end example. Hence, this.

## The main points (as far as I know)

- Before we can store any data, we must create a `SessionStorage` object with the function `createCookieSessionStorage()`.
  - In this example I've followed one of the SolidStart examples and called this object `storage`.
- Cookies are _stored_ in the browser, but data can only be _decrypted_ (and so can only be _used_) on the browser.
  - **So in order to read from a cookie we must use `createServerData$()` and to update a cookie we must use `createServerAction$()`.**
- To access the data inside a `server` function, you need to get the session data from the cookie that's passed through request headers.
  - **You can get the request headers by destructuring a `request` object that is optionally available to you like so: `createServerAction$(async (_, { request }) => { ... }`**
- Then you access the headers like so and get the session data: `const session = await storage.getSession(request.headers.get("Cookie"));`
- Then you can use `session.get("valueName")` and `session.set("valueName", "new value!!")` to get and set session cookie values.
- **But after changing the cookie values, you still need to do two things:** Generate a new encrypted cookie string, and update the string in the user's browser.
- Generating a new cookie string is straightforward: `const newCookie = await storage.commitSession(session);`
- Updating the string in the user's browser can be done by using one of two return types:
  - Returning a `redirect()` which navigates the user to a new page, like so: `return redirect("/new/path/", {headers: {"Set-Cookie": newCookie}});`
  - Returning a `json()` which keeps the user on the same page, like so: `return json({ newCookie }, {headers: {"Set-Cookie": newCookie}});`

### Reading cookie data

- Cookie data must be read inside `createServerData$()` functions that are themselves inside your `routeData()`.
- Then you access the data inside your component function with `useRouteData()` as usual.
  - E.g.: `const { savedCookieText, allCookieDataJson } = useRouteData<typeof routeData>();`
- Returned values are `Resource`s and so, as usual, need to be accessed by calling them as functions, e.g. `<p>Hi there {savedCookieText()}</p>`.

### Clearing cookie data

- To clear session cookies, you first create an "empty" cookie without any secret data using `storage.destroySession()`, and then return that empty cookie so the browser can update.
  1. `const destroyedCookie = await storage.destroySession(session);`
  2. `return json({ destroyedCookie }, {headers: {"Set-Cookie": destroyedCookie}});`

---
type: exercise
title: A render function and controllers
weight: 7
author: Dr Graeme Stuart
---

In this exercise we will tidy up our request handler without changing what our users see.
We will move the work of building a response into a `render` function, and then give each page a *controller*.

<!--more-->

> You should have a home page and a not-found page which share a layout from the [previous exercise][previous exercise].
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

This exercise adds nothing new for our users.
Instead, we will pay off some debt in our code, one small step at a time, checking the browser after each one.
By the end of this exercise you should be able to:

- Explain what we mean by a *feature*, *technical debt* and a *refactor*
- Write a `render` function which builds an HTML response for any view
- Give a function optional settings with an options object and a default value
- Pass a function as a value, and explain why `render` should call the view itself
- Explain what a *controller* is and where controllers fit in model-view-controller
- Pass a *context* object to our controllers

## Setup

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Open [http://localhost:8000/][localhost] and [http://localhost:8000/nope][nope] with the developer tools (**F12**) open on the **network** panel.
You should see our home page with a `200 OK` status, and our not-found page with a `404 Not Found` status.
Both tabs say **Bookshelves**.

## What's the debt? {#technical-debt}

A *feature* is something our application does for its users.
So far we have built a few: a home page, a not-found page, and a header and footer on every page.

Look at the end of **server/main.js**.

**server/main.js**
```JavaScript{hl_lines="21-22 26-27"}
import { serveDir } from "@std/http";
import { layout } from "./views/layout.js";
import { homeView } from "./views/home.js";
import { notFoundView } from "./views/notFound.js";

export function server(request) {
    const url = new URL(request.url);

    // Placeholder: this will need to change when we add middleware
    console.log(request.method, url.pathname);

    // Placeholder: this will need to change when we add a router
    if (url.pathname.startsWith("/assets/")) {
        return serveDir(request, {
            fsRoot: "public",
            urlRoot: "assets",
        });
    }
    if (url.pathname === "/") {
        // Placeholder: this will need to change when we add controllers
        const headers = new Headers({ "content-type": "text/html" });
        return new Response(layout(homeView()), { headers });
    }

    // Placeholder: this will need to change when we add controllers
    const headers = new Headers({ "content-type": "text/html" });
    return new Response(layout(notFoundView()), { headers, status: 404 });
}
```

Each route builds its own `Headers` and its own `Response` by hand.
The same two lines appear twice, with only small differences.
Every new page will need another copy.

This is an example of [*technical debt*][technical debt].
Taking a shortcut got our features working quickly, but we will pay for it later, every time we add a page or need to change how responses are built.
If we add ten pages in this way, that requires ten copies of the same simple pattern.
If we then change the pattern, we need to change it in ten places.
Like real debt, the longer we leave it the more it costs.

We pay off technical debt by [refactoring][refactoring].
A *refactor* changes how our code is organised without changing what it does.
So after each step in this exercise, both pages should look and behave just as they do now.

## A render function {#render}

Let's start by pulling the duplicated lines out into a function of their own.
Its job is to *render* a page, turning content into a complete HTML response.

Create a folder called **tools** inside the **server** folder, with a file called **render.js**.

**server/tools/render.js**
```JavaScript
import { layout } from "../views/layout.js";

// Placeholder: this will need to change when we add flash messages and sign-in
export function render(content) {
    const headers = new Headers({ "content-type": "text/html" });
    return new Response(layout(content), { headers });
}
```

`render` takes some content, wraps it in our layout and returns a `Response` with the `content-type` header.
Notice that **server/tools/render.js** now imports `layout`, rather than **server/main.js**.

> Like `layout`, `render` will need more information later on, when we add flash messages and sign-in.
> This line will change when we add them.

Now use `render` for both routes in **server/main.js**.

**server/main.js**
```JavaScript{hl_lines="2 21 25"}
import { serveDir } from "@std/http";
import { render } from "./tools/render.js";
import { homeView } from "./views/home.js";
import { notFoundView } from "./views/notFound.js";

export function server(request) {
    const url = new URL(request.url);

    // Placeholder: this will need to change when we add middleware
    console.log(request.method, url.pathname);

    // Placeholder: this will need to change when we add a router
    if (url.pathname.startsWith("/assets/")) {
        return serveDir(request, {
            fsRoot: "public",
            urlRoot: "assets",
        });
    }
    if (url.pathname === "/") {
        // Placeholder: this will need to change when we add controllers
        return render(homeView());
    }

    // Placeholder: this will need to change when we add controllers
    return render(notFoundView());
}
```

Each route now has one line.
Refresh [http://localhost:8000/][localhost] and [http://localhost:8000/nope][nope].
Both pages look exactly as they did.

But look at the **nope** row in the **network** panel.

```http
HTTP/1.1 200 OK
content-type: text/html
content-length: 1071
date: Thu, 29 Oct 2026 09:13:33 GMT
```

**Our not-found page now says `200 OK`.**
We lost the `status: 404` when we replaced the second `Response`.
The browser shows the page just the same, so the only way to notice is to check the status.

> This is why we check after every step.
> A refactor that changes behaviour, even invisibly, is a bug.

## A status option {#options}

The two routes need different statuses.
Most pages are `200 OK`, so we only want to give a status when we need a different one.

Update **server/tools/render.js** so that `render` takes an optional second argument.

**server/tools/render.js**
```JavaScript{hl_lines="4 6"}
import { layout } from "../views/layout.js";

// Placeholder: this will need to change when we add flash messages and sign-in
export function render(content, { status } = {}) {
    const headers = new Headers({ "content-type": "text/html" });
    return new Response(layout(content), { headers, status });
}
```

The second argument is an *options object*.
We have used one before, when we passed `{ headers }` to the `Response` constructor.

The curly braces in `{ status } = {}` do two things.

1. `{ status }` takes the `status` property out of the object we are given and puts it in a variable called `status`.
1. `= {}` is a [*default value*][default parameters]. If we don't pass a second argument, an empty object is used instead, so `status` is `undefined`.

When `status` is `undefined`, the `Response` constructor uses its default of `200 OK`.
So the home page doesn't need to say anything.

Now give the not-found route its status in **server/main.js**.

**server/main.js**
```JavaScript{hl_lines="25"}
import { serveDir } from "@std/http";
import { render } from "./tools/render.js";
import { homeView } from "./views/home.js";
import { notFoundView } from "./views/notFound.js";

export function server(request) {
    const url = new URL(request.url);

    // Placeholder: this will need to change when we add middleware
    console.log(request.method, url.pathname);

    // Placeholder: this will need to change when we add a router
    if (url.pathname.startsWith("/assets/")) {
        return serveDir(request, {
            fsRoot: "public",
            urlRoot: "assets",
        });
    }
    if (url.pathname === "/") {
        // Placeholder: this will need to change when we add controllers
        return render(homeView());
    }

    // Placeholder: this will need to change when we add controllers
    return render(notFoundView(), { status: 404 });
}
```

Refresh [http://localhost:8000/nope][nope] and check the **network** panel.

```http
HTTP/1.1 404 Not Found
content-type: text/html
content-length: 1071
date: Thu, 29 Oct 2026 09:13:36 GMT
```

The `404 Not Found` is back, and [http://localhost:8000/][localhost] is still `200 OK`.

> An options object makes it easy to add more options later without changing every call.
> We are about to add another one.

## A title option {#title}

Both of our tabs say **Bookshelves**.
It would be more helpful if the not-found page said so in its tab too.
The [`<title>`][title element] is in our layout, so we need to get a title from the route, through `render`, into `layout`.

Update **server/views/layout.js** so that `layout` also takes an options object with a `title`.

**server/views/layout.js**
```JavaScript{hl_lines="2 7"}
// Placeholder: this will need to change when we add flash messages, sign-in and nav highlighting
export function layout(content, { title } = {}) {
    const document = `
        <!DOCTYPE html>
        <html lang="en">
            <head>
                <meta charset="utf-8">
                <title>${title ? `${title} | ` : ""}Bookshelves</title>
                <meta name="viewport" content="width=device-width, initial-scale=1">
                <link rel="stylesheet" href="/assets/styles.css">
                <link rel="icon" href="/assets/icon.svg">
            </head>
            <body>
            <header class="secondary">
                <a href="/" class="brand"><img src="/assets/icon.svg" alt=""> Bookshelves</a>
                <nav>
                    <a href="/books">books</a>
                </nav>
            </header>
            <main>
            ${content}
            </main>
            <footer class="secondary">
                <p>
                    &copy; Bookshelves developers
                </p>
            </footer>
            </body>
        </html>
    `;
    return document;
}
```

The `<title>` uses the [conditional operator][conditional operator] (`? :`) inside our template literal.
If there is a `title`, it adds the title and a `|` before **Bookshelves**.
If there isn't, it adds an empty string, leaving just **Bookshelves**.

> The part after the `?` is another template literal, nested inside the first.
> It looks a little odd, but it works just like the outer one.

Now add a `title` option to `render` and pass it through to `layout`.

**server/tools/render.js**
```JavaScript{hl_lines="4 6"}
import { layout } from "../views/layout.js";

// Placeholder: this will need to change when we add flash messages and sign-in
export function render(content, { status, title } = {}) {
    const headers = new Headers({ "content-type": "text/html" });
    return new Response(layout(content, { title }), { headers, status });
}
```

This is [*destructuring*][destructuring].
`{ status, title }` takes two properties out of the options object at once, each into its own variable.
Then `{ title }` builds a new object for `layout`, with a `title` property holding our `title` variable.

> `{ title }` is short for `{ title: title }`.
> We did the same with `{ headers }` and `{ headers, status }`.

Finally, give the not-found route a title in **server/main.js**.

**server/main.js**
```JavaScript{hl_lines="25"}
import { serveDir } from "@std/http";
import { render } from "./tools/render.js";
import { homeView } from "./views/home.js";
import { notFoundView } from "./views/notFound.js";

export function server(request) {
    const url = new URL(request.url);

    // Placeholder: this will need to change when we add middleware
    console.log(request.method, url.pathname);

    // Placeholder: this will need to change when we add a router
    if (url.pathname.startsWith("/assets/")) {
        return serveDir(request, {
            fsRoot: "public",
            urlRoot: "assets",
        });
    }
    if (url.pathname === "/") {
        // Placeholder: this will need to change when we add controllers
        return render(homeView());
    }

    // Placeholder: this will need to change when we add controllers
    return render(notFoundView(), { status: 404, title: "Not found" });
}
```

Refresh both pages and look at the tabs.

- [http://localhost:8000/][localhost] passes no options, so it gets the default title, **Bookshelves**.
- [http://localhost:8000/nope][nope] passes a title, so its tab says **Not found | Bookshelves**.

{{<figure src="images/tab-titles.webp" alt="Two browser tabs titled Bookshelves and Not found | Bookshelves, with the not-found page open at localhost:8000/nope" caption="Each page now has its own title in the tab.">}}

Both statuses are unchanged in the **network** panel.

## Letting render run the view {#view-functions}

Look at how we call `render` now.

```JavaScript {linenos=false}
render(homeView())
```

We call the view ourselves and pass its HTML to `render`.
Every route has to remember to do this, and to do it in the same way.
It would be better if `render` called the view itself, so that the whole page is built in one place.

In JavaScript, functions are [*first-class*][first-class function] values.
We can pass a function to another function, just like a string or a number.
In fact we already do this, in **tasks/server.js**: `Deno.serve(server)` passes our `server` function to deno, which calls it for every request.

Update **server/tools/render.js** so that it takes a `view` function and calls it.

**server/tools/render.js**
```JavaScript{hl_lines="4 6"}
import { layout } from "../views/layout.js";

// Placeholder: this will need to change when we add flash messages and sign-in
export function render(view, { status, title } = {}) {
    const headers = new Headers({ "content-type": "text/html" });
    return new Response(layout(view(), { title }), { headers, status });
}
```

`view` is a function, so `view()` calls it to get the content.

Now update **server/main.js** to pass the views themselves.
**Notice that there are no brackets after `homeView` and `notFoundView`.**

**server/main.js**
```JavaScript{hl_lines="21 25"}
import { serveDir } from "@std/http";
import { render } from "./tools/render.js";
import { homeView } from "./views/home.js";
import { notFoundView } from "./views/notFound.js";

export function server(request) {
    const url = new URL(request.url);

    // Placeholder: this will need to change when we add middleware
    console.log(request.method, url.pathname);

    // Placeholder: this will need to change when we add a router
    if (url.pathname.startsWith("/assets/")) {
        return serveDir(request, {
            fsRoot: "public",
            urlRoot: "assets",
        });
    }
    if (url.pathname === "/") {
        // Placeholder: this will need to change when we add controllers
        return render(homeView);
    }

    // Placeholder: this will need to change when we add controllers
    return render(notFoundView, { status: 404, title: "Not found" });
}
```

`homeView` is the function.
`homeView()` calls the function and gives us its result, a string of HTML.

Refresh both pages.
Nothing should have changed.

### A deliberate mistake {#not-a-function}

It is very easy to type those brackets out of habit.
Let's see what happens.
Put the brackets back on `homeView` in **server/main.js**.

**server/main.js**
```JavaScript{hl_lines="21"}
import { serveDir } from "@std/http";
import { render } from "./tools/render.js";
import { homeView } from "./views/home.js";
import { notFoundView } from "./views/notFound.js";

export function server(request) {
    const url = new URL(request.url);

    // Placeholder: this will need to change when we add middleware
    console.log(request.method, url.pathname);

    // Placeholder: this will need to change when we add a router
    if (url.pathname.startsWith("/assets/")) {
        return serveDir(request, {
            fsRoot: "public",
            urlRoot: "assets",
        });
    }
    if (url.pathname === "/") {
        // Placeholder: this will need to change when we add controllers
        return render(homeView());
    }

    // Placeholder: this will need to change when we add controllers
    return render(notFoundView, { status: 404, title: "Not found" });
}
```

Refresh [http://localhost:8000/][localhost].
The browser shows **Internal Server Error**, and the **network** panel shows a `500 Internal Server Error` status.
The terminal tells us why.

```plaintext {linenos=false}
GET /
TypeError: view is not a function
    at render (file:///home/me/my-app/server/tools/render.js:6:32)
    at server (file:///home/me/my-app/server/main.js:21:16)
    at nativeMapped (ext:deno_http/00_serve.ts:1:13754)
    at nativeDispatch (ext:deno_http/00_serve.ts:1:17522)
```

> The path to your project will be different.

{{<figure src="images/internal-server-error.webp" alt="A blank page with the words Internal Server Error in small plain text" caption="Passing `homeView()` instead of `homeView` gives a 500 Internal Server Error.">}}

This time the `at ...` lines are useful.
The first two are our own code.
The error happened on line 6 of **render.js**, which was called from line 21 of **main.js**.

We passed the result of `homeView()`, which is a string.
`render` then tried to call it with `view()`, and you can't call a string.
See [TypeError: "x" is not a function][not a function].

**Take the brackets off again** and check the home page is back.

> Passing the view itself means `render` decides when to build the page.
> The route only says *which* view, and `render` does the rest.
> Building the page in one place will pay off when `render` needs to give the view more information.

## Controllers {#controllers}

Our request handler in **server/main.js** still does two jobs.
It decides which page was requested, and it knows how to build each page: which view, which status and which title.
It also knows how to serve our static files.
Let's separate them.

Create a folder called **controllers** inside the **server** folder, with a file for each page.

**server/controllers/home.js**
```JavaScript
import { render } from "../tools/render.js";
import { homeView } from "../views/home.js";

export function homeController() {
    return render(homeView);
}
```

**server/controllers/notFound.js**
```JavaScript
import { render } from "../tools/render.js";
import { notFoundView } from "../views/notFound.js";

export function notFoundController() {
    return render(notFoundView, { status: 404, title: "Not found" });
}
```

These are *controllers*.
A controller is a function which handles a request.
These two choose the view, set any options, and return the response.

Serving static files is a job for a controller too, even though there is no view.
Add a third file to the **controllers** folder.

**server/controllers/static.js**
```JavaScript
import { serveDir } from "@std/http";

export function staticController(ctx) {
    const { request } = ctx;
    return serveDir(request, {
        fsRoot: "public",
        urlRoot: "assets",
    });
}
```

This is the same `serveDir` call we had in **server/main.js**.
The difference is where the `request` comes from.

Now update **server/main.js** to import the controllers and call them.

**server/main.js**
```JavaScript{hl_lines="1-3 7 14 17 19"}
import { homeController } from "./controllers/home.js";
import { staticController } from "./controllers/static.js";
import { notFoundController } from "./controllers/notFound.js";

export function server(request) {
    const url = new URL(request.url);
    const ctx = { request };

    // Placeholder: this will need to change when we add middleware
    console.log(request.method, url.pathname);

    // Placeholder: this will need to change when we add a router
    if (url.pathname.startsWith("/assets/")) {
        return staticController(ctx);
    }
    if (url.pathname === "/") {
        return homeController(ctx);
    }
    return notFoundController(ctx);
}
```

> The controllers placeholders have done their job and are gone.
> The router placeholder is still there.
> When we add a router, it will decide which controller to call and these `if` statements will be replaced.

Our request handler now only decides which controller to call.
It doesn't import any views or `serveDir`, and it doesn't know anything about HTML.

### A context object {#ctx}

`ctx` is our *context* object.
It carries information about the request to the controller that handles it.
For now, it holds just the `request`.

The static controller takes the `request` out of `ctx`, with [destructuring][destructuring], and gives it to `serveDir`.
Our home and not-found controllers don't need anything from `ctx` yet, so they don't declare it.
**server/main.js** passes `ctx` to every controller anyway.

So a controller is a function which takes the request context and returns a `Response`.

> This is the *context pattern*.
> Rather than giving our functions a growing list of arguments, we pass one object that carries everything needed to handle the request.
> It will be a core part of our architecture, and `ctx` will grow as our app does, for example to carry the user who is signed in.

### The C in MVC {#mvc}

Controllers are the C in [model-view-controller][mvc].
In the previous exercise our request handler was doing the controllers' job, and now each page has a controller of its own.
A controller is a function, just like a view, and related functions are grouped into modules.
All our controllers live in the **controllers** folder, next to the **views** folder.

> `render` lives in a **tools** folder because it doesn't belong to any one page.
> It is a helper which any controller can use.

## Nothing changed for the user {#nothing-changed}

Let's check that our refactor really did leave our features alone.
Keep the **network** panel open.

- [http://localhost:8000/][localhost] is `200 OK`, and its tab says **Bookshelves**.
- [http://localhost:8000/nope][nope] is `404 Not Found`, and its tab says **Not found | Bookshelves**.

```http
HTTP/1.1 404 Not Found
content-type: text/html
content-length: 1083
date: Thu, 29 Oct 2026 09:13:48 GMT
```

Apart from the new tab title, our users would never know anything had changed.
But our code is in much better shape.

## Conclusion

We paid off some technical debt with a series of small refactors, checking the browser after each one.
We moved the response building into a `render` function, gave it `status` and `title` options, and let it call the view itself.
Then we moved each route's work into a controller, the C in model-view-controller, including a controller for our static files.
Each controller is given a context object, `ctx`, which holds the request.
Our request handler now only decides which controller to call.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="8-13 16 18"}
my-app/
├── .vscode/
│   └── settings.json
├── public/
│   ├── icon.svg
│   └── styles.css
├── server/
│   ├── controllers/
│   │   ├── home.js
│   │   ├── notFound.js
│   │   └── static.js
│   ├── tools/
│   │   └── render.js
│   ├── views/
│   │   ├── home.js
│   │   ├── layout.js
│   │   └── notFound.js
│   └── main.js
├── tasks/
│   └── server.js
├── deno.json
└── deno.lock
```

## Challenge: Add an About page

Try the [about page challenge][about challenge] once more, now that we have controllers.
You will need a view, a controller and one more `if` in **server/main.js**.

- Base your view and controller on the not-found ones. Your page should be `200 OK` at [http://localhost:8000/about][about], with **About | Bookshelves** in the tab.
- If you like, add a link to it in the `<nav>` in **server/views/layout.js**. Leave the link out of your count, so that it compares with the other exercises.
- When it works, count how many files you touched and how many lines you added or changed. The **source control** panel lists the files.

Compare with your counts from the previous two exercises.
Think about the number of lines, the effort involved, the quality of the code and any technical debt, and draw your own conclusions.

We will set this same challenge again after later changes to the structure of our code, so that you can compare again.

> **Remove your about page before moving on.**
> Delete the two new files and undo your changes to **server/main.js** (and **server/views/layout.js**, if you added a link), so that your code matches the starting point of the next exercise.
> If you committed before you started, the **source control** panel can discard the changes for you.

Finally, something to think about.
Every page needs another `if` in **server/main.js**, and they all look the same.
How could we describe our routes as data, rather than as code?

[previous exercise]: deno-server/views/
[localhost]: http://localhost:8000/
[nope]: http://localhost:8000/nope
[technical debt]: https://en.wikipedia.org/wiki/Technical_debt
[refactoring]: deno-server/views/#refactoring
[default parameters]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Default_parameters
[title element]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/title
[conditional operator]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Conditional_operator
[destructuring]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring
[first-class function]: https://developer.mozilla.org/en-US/docs/Glossary/First-class_Function
[not a function]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Errors/Not_a_function
[mvc]: deno-server/views/#mvc
[about challenge]: deno-server/missing-pages/#about-challenge
[about]: http://localhost:8000/about

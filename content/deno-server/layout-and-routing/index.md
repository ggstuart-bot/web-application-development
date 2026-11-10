---
type: exercise
title: Adding a second page
weight: 11
author: Dr Graeme Stuart
---

In this exercise, we will upgrade our server to serve two distinct, interlinked pages.
This will require some routing logic and will create opportunities to avoid repetition when rendering a standard page template.

<!--more-->

> Before you start, make sure everything is committed to your git repository. 
You should have a working **hello world** page from the [previous exercise]({{< ref "/http-servers/deno-serve" >}}). 

## Learning objectives

- basic routing
- dividing code into [modules]
- a layout function for well-formed HTML documents
- basic view functions
- a render helper function
- string interpolation with simple [template literals]

## Extracting the URL

If our server is going to serve different pages, it needs to understand what is being requested.
The first step is to grab the URL from the request and look at the pathname component.

> The pathname will distinguish between the root (**'/'**) and e.g. the favicon (**'/favicon.ico'**) without worrying about the host or protocol.

We can use the [URL API][url api] to do the work for us.

**server/main.js**
```JavaScript{hl_lines=[2, 3]}
export function server(request) {
    const url = new URL(request.url);
    console.log(request.method, url.pathname);
    const headers = new Headers({ "content-type": "text/html" });
    return new Response("<h1>hello world</h1>", { headers });
}
```

Make sure your server is running with the *"dev"* task.

```bash{linenos=false}
deno task dev
```

When you refresh the page in the browser, the output is reduced to the most important detail.

```bash{linenos=false}
GET /
GET /favicon.ico
```

> I also got a request for **/.well-known/appspecific/com.chrome.devtools.json** because I had the developer tools open.
> We can safely ignore this request.

## Setting up a template

If we want to serve multiple, more realistic pages from our server then we will need to generate a lot more HTML.

Create a new file **server/layout.js** containing a function that returns a fully-formed HTML document with the content for the page embedded in the `<main>` element.

**server/layout.js**
```JavaScript{hl_lines=[1, 17]}
export function layout(content) {
    return `<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>My website</title>
</head>
<body>
    <header>
        <h1>My website</h1>
        <nav>
            <a href="/">home</a>
            <a href="/about">about</a>
        </nav>
    </header>
    <main>
    ${content}
    </main>
    <footer>
        &copy; My website developers
    </footer>
</body>
</html>`;
}
```

The string returned is a complete HTML page.
Notice a few details which make our document *well-formed*.

- The [doctype] must be the very first thing in the document, so it comes immediately after the opening backtick.
- The [lang] attribute tells the browser (and screen readers) that the page is in English.
- The [meta charset][meta charset] element tells the browser how the characters are encoded.
- Every element we open is closed.

> The `${content}` placeholder is a [template literal][template literals] expression.
> Whatever string we pass in as `content` will be inserted into the `<main>` element.

Now we can create a variety of similar pages using the same function.

### Create some **views**

Our simple view functions are just functions that return HTML fragments that can be embedded into the main layout.
Create a folder called **server/views** with the following two files.

**server/views/home.js**
```JavaScript
export function homeView() {
    return `
    <section>
        <h2>Home</h2>
        <p>This is the home page</p>
        <p>See the <a href="/about">about</a> page for more information</p>
    </section>
    `;
}
```

**server/views/about.js**
```JavaScript
export function aboutView() {
    return `
    <section>
        <h2>About</h2>
        <p>This is the about page</p>
        <p>Go back to the <a href="/">home</a> page</p>
    </section>
    `;
}
```

> The two pages link to each other, both in the navigation and in the content.

## Basic routing

Finally, we can [import] our functions and add a simple pair of conditional statements to fork the request handling within **server/main.js**.

**server/main.js**
```JavaScript{hl_lines=["1-3", "10-20"]}
import { layout } from "./layout.js";
import { aboutView } from "./views/about.js";
import { homeView } from "./views/home.js";

export function server(request) {
    const url = new URL(request.url);
    console.log(request.method, url.pathname);
    const headers = new Headers({ "content-type": "text/html" });

    if (url.pathname == "/") {
        const content = homeView();
        const html = layout(content);
        return new Response(html, { headers });
    }

    if (url.pathname == "/about") {
        const content = aboutView();
        const html = layout(content);
        return new Response(html, { headers });
    }
}
```

The result should be that your site now has two distinct pages. 
Requests for the home page and for the about page are handled differently with slightly different results.
Click the links to move between them.

However, you may also notice that you are getting errors in the terminal.
```bash
TypeError: Return value from serve handler must be a response or a promise resolving to a response
```
This is because now, only the two page requests are returning a Response object. 
The other requests, e.g. for the icon, are being ignored and returning `undefined` by default.

### Add a **notFound** view

We can solve this by creating a new view function for any requests that are not explicitly handled.

**server/views/notFound.js**
```JavaScript
export function notFoundView() {
    return `
    <section>
        <h2>Not Found</h2>
        <p>The requested resource was not found.</p>
    </section>
    `;
}
```

We can add this as a fallback in **server/main.js** after our conditional statements.

**server/main.js**
```JavaScript{hl_lines=[4, "23-25"]}
import { layout } from "./layout.js";
import { aboutView } from "./views/about.js";
import { homeView } from "./views/home.js";
import { notFoundView } from "./views/notFound.js";

export function server(request) {
    const url = new URL(request.url);
    console.log(request.method, url.pathname);
    const headers = new Headers({ "content-type": "text/html" });

    if (url.pathname == "/") {
        const content = homeView();
        const html = layout(content);
        return new Response(html, { headers });
    }

    if (url.pathname == "/about") {
        const content = aboutView();
        const html = layout(content);
        return new Response(html, { headers });
    }

    const content = notFoundView();
    const html = layout(content);
    return new Response(html, { headers, status: 404 });
}
```

All the errors are now gone.
Also note that we can visit a missing url such as [http://localhost:8000/missing] and the **Not Found** page will be rendered with a **404** status code.

It is clear that our code includes a lot of repetition.

## A **render** helper function

We will reduce the repetition with a helper function that allows us to generate an HTML response with a given view function and a given layout.

Create a **server/tools** folder with a file called **render.js**.
This will be the function we call when we want to render an HTML page.

**server/tools/render.js**
```JavaScript
export function render(view, layout, { status } = {}) {
    const headers = new Headers({ "content-type": "text/html" });
    const content = view();
    const html = layout(content);
    return new Response(html, { headers, status });
}
```

It takes two arguments, both of which are functions, and an optional third argument for extra options.

> We are [destructuring] the third argument.
> We would get an error if the argument was undefined as we cannot destructure undefined.
> To handle this, we provide an empty object as a default value. 
> With this default, when no value is provided, the status variable simply becomes undefined and the default **200 OK** status is used.

This allows us to greatly simplify the code in **server/main.js**.

**server/main.js**
```JavaScript{hl_lines=[2, 12, 16, 19]}
import { layout } from "./layout.js";
import { render } from "./tools/render.js";
import { aboutView } from "./views/about.js";
import { homeView } from "./views/home.js";
import { notFoundView } from "./views/notFound.js";

export function server(request) {
    const url = new URL(request.url);
    console.log(request.method, url.pathname);

    if (url.pathname == "/") {
        return render(homeView, layout);
    }

    if (url.pathname == "/about") {
        return render(aboutView, layout);
    }

    return render(notFoundView, layout, { status: 404 });
}
```

Notice that the code reads quite well. 
We are rendering a specific view using the main layout function.
We choose a different view depending on the circumstances.

> Notice that **tasks/server.js** has not changed at all.
> It still imports our `server` function and passes it to `Deno.serve`.

### Check the HTML

View the page source in the browser (**Ctrl + U**) to see the full HTML document our server generated.

> Try pasting the source into the [W3C markup validation service][validator].
> It should report no errors.

## Conclusions

We have seen that our HTTP server receives HTTP **Request** objects and returns HTTP **Response** objects. 
We have implemented a simple form of **routing** (via a sequence of conditional statements) to choose which view to render for each request.
We implemented a fallback for requests which we don't explicitly understand.  

Our **render** helper function returns **Response** objects with the correct HTML **content-type** header and an optional status code. 
We have **view functions** and a **layout function** which handle the details of creating well-formed HTML documents.

Our project now looks like this.

```plaintext{linenos=false}
.vscode/
    settings.json
deno.json
server/
    layout.js
    main.js
    tools/
        render.js
    views/
        about.js
        home.js
        notFound.js
tasks/
    hello.js
    server.js
```

[modules]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules
[template literals]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals
[url api]: https://developer.mozilla.org/en-US/docs/Web/API/URL
[doctype]: https://developer.mozilla.org/en-US/docs/Glossary/Doctype
[lang]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Global_attributes/lang
[meta charset]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/meta#charset
[import]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import
[http://localhost:8000/missing]: http://localhost:8000/missing
[destructuring]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring
[validator]: https://validator.w3.org/#validate_by_input

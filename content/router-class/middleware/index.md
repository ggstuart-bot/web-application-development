---
type: exercise
title: One context and middleware
weight: 3
author: Dr Graeme Stuart
---

In this exercise we will give our controllers, views and layout one shared context object, and use it to highlight the current page in the nav.
Then we will replace our logging placeholder with our first *middleware*.

<!--more-->

> You should have a router from the [previous exercise][previous exercise].
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

Like the previous exercise, most of this one is a refactor.
By the end of this exercise you should be able to:

- Pass one context object through our controllers, `render`, views and layout
- Put each page's data on `ctx.data`
- Mark the current page in the nav with `aria-current`, and style it
- Explain middleware as a function that runs for every route, and write one
- Use `async` and `await` where a function needs to wait for a result

## Where we are {#debt}

Our controllers already receive our [context object][ctx], but they don't do much with it.
Instead, they give `render` an options object with a [`status`][options], a [`title`][title] and some [`data`][passing data].

That has two problems.

- Every new piece of information has needed a new option, and changes to `render`.
- The views get `{ data }`, the layout gets `{ title }`, and the controllers get `ctx`. Three different shapes for one request.

We said that `ctx` would grow as our app does.
It's time to let it carry everything a page needs.

There is one more piece of debt.
Since our very first server we have [logged each request][request logging] with a `console.log` in **server/main.js**, under a placeholder that says it will change when we add middleware.

Start the server in development mode, if it isn't already running.

```bash {linenos=false}
deno task dev
```

## One context everywhere {#ctx-data}

`render`, the layout, the views and every controller all have to change for this.
That's a lot at once, so we will make the changes one file at a time.
Some pages will break along the way, and we will fix them one by one.

### A data object {#data}

Start with **server/main.js**, which makes `ctx`.

**server/main.js**
```JavaScript{hl_lines="16 26"}
import { homeController } from "./controllers/home.js";
import { booksController } from "./controllers/books.js";
import { staticController } from "./controllers/static.js";
import { notFoundController } from "./controllers/notFound.js";
import { NotFoundError } from "./tools/errors.js";
import { Router } from "./tools/router.js";

const app = new Router();

app.get("/assets/*", staticController);
app.get("/", homeController);
app.get("/books", booksController);

export function server(request) {
    const url = new URL(request.url);
    const ctx = { request, url, data: {} };

    // Placeholder: this will need to change when we add middleware
    console.log(request.method, url.pathname);

    try {
        return app.handle(ctx);
    } catch (error) {
        if (error instanceof NotFoundError) {
            console.log("Not found:", error.message);
            return notFoundController(ctx);
        }
        throw error;
    }
}
```

Every request now gets its own `ctx` with an empty `data` object.
`ctx.data` is where the controller and any middleware collect everything this response's HTML needs, in one place that both the view and the layout read from.
The layout only uses `title` (and, later, `flash`).
The not-found controller is given `ctx` too, like every other controller.

> The middleware placeholder is still there.
> We will replace it later in this exercise.

Nothing uses `ctx.data` yet, so refresh [http://localhost:8000/books][books] to check that nothing has changed.

### render takes ctx {#render-ctx}

Update **server/tools/render.js** to take `ctx` instead of an options object.

**server/tools/render.js**
```JavaScript{hl_lines="4-5 7-9"}
import { layout } from "../views/layout.js";

// Placeholder: this will need to change when we add flash messages and sign-in
export function render(view, ctx) {
    const { status } = ctx;
    const headers = new Headers({ "content-type": "text/html" });
    const content = view(ctx);
    const html = layout(content, ctx);
    return new Response(html, { headers, status });
}
```

`render` takes the `status` out of `ctx`, then passes the whole of `ctx` to the view and to the layout.
We have also given `content` and `html` names of their own, so each line does one thing.

> The placeholder hasn't changed.
> Flash messages and sign-in will need `render` to do more.

Update **server/views/layout.js** to take its title from `ctx.data`.

**server/views/layout.js**
```JavaScript{hl_lines="2-4"}
// Placeholder: this will need to change when we add flash messages, sign-in and nav highlighting
export function layout(content, ctx) {
    const { data } = ctx;
    const title = data.title ? `${data.title} | Bookshelves` : "Bookshelves";

    const document = `
        <!DOCTYPE html>
        <html lang="en">
            <head>
                <meta charset="utf-8">
                <title>${title}</title>
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

The layout takes what it needs out of `ctx`, with [destructuring][destructuring].

> The layout placeholder hasn't changed yet.
> We will deal with nav highlighting in the next step.

This version only partly works.
Refresh [http://localhost:8000/][localhost].
It is `500 Internal Server Error`, and the terminal explains why.

```plaintext {linenos=false}
TypeError: Cannot destructure property 'status' of 'ctx' as it is undefined.
    at render (file:///…/server/tools/render.js:5:13)
    at Object.homeController [as handler] (file:///…/server/controllers/home.js:5:12)
```

The home controller still calls `render(homeView)`, so `ctx` is `undefined`.
[http://localhost:8000/nope][nope] is a `500` too.
[http://localhost:8000/books][books] still shows the books, because its old options object happens to have a `data` property, but its tab now says just **Bookshelves**.

Let's fix the controllers one at a time.

### The home page {#home-ctx}

**server/controllers/home.js**
```JavaScript{hl_lines="4-5"}
import { render } from "../tools/render.js";
import { homeView } from "../views/home.js";

export function homeController(ctx) {
    return render(homeView, ctx);
}
```

The home controller has nothing to add, but `render` needs `ctx`, so it now declares it and passes it on.

Refresh [http://localhost:8000/][localhost].
It is `200 OK` again, and its tab says **Bookshelves**.

### The books page {#books-ctx}

The books controller puts its title and its books on `ctx.data`.

**server/controllers/books.js**
```JavaScript{hl_lines="5-8"}
import { getBooks } from "../models/books.js";
import { render } from "../tools/render.js";
import { booksView } from "../views/books.js";

export function booksController(ctx) {
    ctx.data.books = getBooks();
    ctx.data.title = "Books";
    return render(booksView, ctx);
}
```

The view takes the books from `ctx`.

**server/views/books.js**
```JavaScript{hl_lines="3-4"}
import { bookCard } from "./partials/bookCard.js";

export function booksView(ctx) {
    const { data: { books } } = ctx;

    return `
        <section>
            <header>
                <h1>Books</h1>
            </header>
            <ul class="grid">
                ${books.map(bookCard).join("")}
            </ul>
        </section>
    `;
}
```

`const { data: { books } } = ctx;` takes `data` out of `ctx`, and then `books` out of `data`.
It is the same destructuring the view used before, moved inside the function.

> Our home and not-found views take no arguments, so they simply ignore `ctx`.

Refresh [http://localhost:8000/books][books].
The tab says **Books | Bookshelves** again.

### The not-found page {#not-found-ctx}

**server/controllers/notFound.js**
```JavaScript{hl_lines="4-7"}
import { render } from "../tools/render.js";
import { notFoundView } from "../views/notFound.js";

export function notFoundController(ctx) {
    ctx.status = 404;
    ctx.data.title = "Not found";
    return render(notFoundView, ctx);
}
```

The not-found controller sets `ctx.status` to `404`, and puts its title on `ctx.data`.

Refresh [http://localhost:8000/nope][nope].
It is `404 Not Found` again, and its tab says **Not found | Bookshelves**.

### One shape for everything {#one-shape}

Nothing has changed for our users.
But now a controller really is a function which takes the request context and returns a `Response`, and the views and the layout take the same context.
When a page needs something new, its controller puts it on `ctx`, and `render` doesn't have to change.

## The current page {#nav-link}

Now that the layout has `ctx`, it can find out which page it is wrapping.
Our router put the `url` on `ctx`, so the layout can compare each nav link with `url.pathname`.

Update **server/views/layout.js**.

**server/views/layout.js**
```JavaScript{hl_lines="1 3 20 37-40"}
// Placeholder: this will need to change when we add flash messages and sign-in
export function layout(content, ctx) {
    const { data, url } = ctx;
    const title = data.title ? `${data.title} | Bookshelves` : "Bookshelves";

    const document = `
        <!DOCTYPE html>
        <html lang="en">
            <head>
                <meta charset="utf-8">
                <title>${title}</title>
                <meta name="viewport" content="width=device-width, initial-scale=1">
                <link rel="stylesheet" href="/assets/styles.css">
                <link rel="icon" href="/assets/icon.svg">
            </head>
            <body>
            <header class="secondary">
                <a href="/" class="brand"><img src="/assets/icon.svg" alt=""> Bookshelves</a>
                <nav>
                    ${navLink("/books", "books", url.pathname)}
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

// A header link, marked as the current page when it matches the path exactly
function navLink(href, content, path) {
    return `<a href="${href}" aria-current="${href === path ? "page" : "false"}">${content}</a>`;
}
```

`navLink` builds one link in the header.
If the link's `href` is the path of the current page, it gets [`aria-current="page"`][aria-current].
That tells screen readers, and our stylesheet, which link is the current page.
The [conditional operator][conditional operator] picks `"page"` or `"false"`.

`navLink` isn't exported, because only the layout uses it.

> The placeholder no longer mentions nav highlighting, because that's done.
> Flash messages and sign-in are still to come.

Open [http://localhost:8000/books][books], and look at the nav in the **elements** panel.

```html
<nav>
    <a href="/books" aria-current="page">books</a>
</nav>
```

On [http://localhost:8000/][localhost] the same link has `aria-current="false"`.
The page looks the same, because nothing in our stylesheet uses the attribute yet.

## Styling the current link {#nav-styles}

Add a rule to **public/styles.css**, after the `a` rule.

**public/styles.css**
```css {linenostart="26"}
/* The current page's nav link is filled with the accent */
nav a[aria-current="page"] {
    background: var(--highlight);
    color: contrast-color(var(--highlight));
}
```

Then add a `nav a` rule to the end of the file.

**public/styles.css**
```css {linenostart="136"}
/* Nav links are outlined pills, underlined on hover */
nav a {
    padding: 0.25em 0.75em;
    border: 1px solid;
    border-radius: 1em;
    background: none;
    color: inherit;
    text-decoration: none;
    &:hover {
        text-decoration: underline var(--highlight) 2px;
    }
}
```

- `nav a` gives every link in the nav an outline with round ends, like a pill. The `&:hover` is nested inside it, like the `& > header` in our `body` rule.
- [`[aria-current="page"]`][attribute selectors] is an *attribute selector*. It only matches the link for the current page, and fills it with our accent colour.
- [`contrast-color()`][contrast-color] picks black or white, whichever is easier to read on the accent.

> The attribute selector makes the first rule more specific than `nav a`, so it wins even though it comes first in the file.

Refresh [http://localhost:8000/books][books].
The **books** link in the header is filled with the accent colour.
On [http://localhost:8000/][localhost] it is an outlined pill.

## Middleware {#middleware}

Look at the logging in **server/main.js**.
It isn't routing, and it isn't the job of any one controller.
It's something we want to happen for *every* request, and it would be more useful if it could log the response too.

A *middleware* is a function that runs for every route, wrapped around the route's handler.
It can do something before the handler, and something after it.
To make that possible, our router has to learn to run a *chain* of functions, rather than just the handler.

### A chain of handlers {#chain}

Update **server/tools/router.js**.

**server/tools/router.js**
```JavaScript{hl_lines="3 7 15-17 33-34 36-40"}
import { NotFoundError } from "./errors.js";

// Placeholder: this will need to change when we add URL parameters
export class Router {
    constructor() {
        this.routes = [];
        this.middleware = [];
    }

    register(method, pathname, handler) {
        const pattern = new URLPattern({ pathname });
        this.routes.push({ method, pattern, handler });
    }

    use(fn) {
        this.middleware.push(fn);
    }

    get(...args) {
        this.register("GET", ...args);
    }

    handle(ctx) {
        const { url, request } = ctx;
        const route = this.routes.find((route) => {
            return route.method === request.method && route.pattern.test(url);
        });
        if (!route) {
            throw new NotFoundError(
                `${request.method} ${url.pathname} doesn't match any known route`,
            );
        }
        return this.chain(ctx, [...this.middleware, route.handler]);
    }

    chain(ctx, handlers) {
        const [fn, ...remaining] = handlers;
        if (!remaining.length) return fn(ctx);
        const next = (ctx) => this.chain(ctx, remaining);
        return fn(ctx, next);
    }
}
```

> The placeholder no longer mentions middleware.
> Our router will still need to change when we add URL parameters.

The constructor gives the router a second array, `this.middleware`, and the new `use` method adds a function to it.

`handle` no longer calls the route's handler directly.
It makes a list of functions, with our middleware first and the route's handler last, and passes it to `chain`.
`[...this.middleware, route.handler]` uses [spread syntax][spread syntax] again, this time to copy the middleware into a new array.

`chain` calls the first function in the list, and gives it a way to call the rest.

- `const [fn, ...remaining] = handlers;` takes the first function out of the list, and collects the others in `remaining`. This is [array destructuring][array destructuring], with a rest element like the one in `get`.
- If nothing remains, `fn` is the route's handler, so we call it with `ctx`. (`!remaining.length` is `true` when `remaining` is empty.)
- Otherwise we make `next`, an [arrow function][arrow functions] that runs `chain` on the rest of the list, and call `fn(ctx, next)`.

So every function in the chain, apart from the last, gets a `next` function that runs the rest of the chain.
Handing `next` over is just like handing a function to [`map`][map]: we pass a function, and the other code decides when to call it.

> Because `next` is an arrow function, `this` inside it is still our router.

We haven't registered any middleware, so the chain is just the handler.
Refresh [http://localhost:8000/books][books] to check that nothing has changed.

### Our first middleware {#with-logs}

Create **logging.js** in a new **server/middleware** folder.

**server/middleware/logging.js**
```JavaScript
// Placeholder: this will need to change when we add flash messages and sessions
export async function withLogs(ctx, next) {
    const { url, request } = ctx;
    console.log("\n============================");
    console.log(`${request.method} ${url.pathname}`);
    const response = await next(ctx);
    console.log("RESPONSE", response.status);
    console.log("============================");

    return response;
}
```

> The placeholder says `withLogs` will change.
> When we add flash messages and sessions, it will log those too.

`withLogs` takes `ctx`, like a controller, and a second argument, `next`.
It logs the request, calls `next` to get the response from the rest of the chain, logs its status, and returns the response unchanged.

### async and await {#async}

The response from `next` might not be ready straight away.
Our static controller, for example, returns a [promise][promise] of a response, because reading a file takes time.
We said in the [static files exercise][serve dir] that we would look at promises properly later; this is the first time we need to.

- A promise stands for a result that will arrive later.
- [`await`][await] pauses our function until that result has arrived. The server carries on handling other requests in the meantime.
- Only an [`async`][async function] function can use `await`, and an `async` function always returns a promise. `Deno.serve` waits for it.

That's all we need for now.

### Using the middleware {#use}

Update **server/main.js** to register `withLogs`.

**server/main.js**
```JavaScript{hl_lines="7 11"}
import { homeController } from "./controllers/home.js";
import { booksController } from "./controllers/books.js";
import { staticController } from "./controllers/static.js";
import { notFoundController } from "./controllers/notFound.js";
import { NotFoundError } from "./tools/errors.js";
import { Router } from "./tools/router.js";
import { withLogs } from "./middleware/logging.js";

const app = new Router();

app.use(withLogs);

app.get("/assets/*", staticController);
app.get("/", homeController);
app.get("/books", booksController);

export function server(request) {
    const url = new URL(request.url);
    const ctx = { request, url, data: {} };

    // Placeholder: this will need to change when we add middleware
    console.log(request.method, url.pathname);

    try {
        return app.handle(ctx);
    } catch (error) {
        if (error instanceof NotFoundError) {
            console.log("Not found:", error.message);
            return notFoundController(ctx);
        }
        throw error;
    }
}
```

`app.use(withLogs)` adds our middleware before the routes.

Open [http://localhost:8000/books][books] and look at the terminal.

```plaintext {linenos=false}
GET /books

============================
GET /books
RESPONSE 200
============================
```

Every request is logged twice: once by the old line in **server/main.js**, and once by `withLogs`, which also logs the status.

### Removing the placeholder {#remove-placeholder}

The old line has done its job.
Remove it, and its placeholder, from **server/main.js**.

**server/main.js**
```JavaScript
import { homeController } from "./controllers/home.js";
import { booksController } from "./controllers/books.js";
import { staticController } from "./controllers/static.js";
import { notFoundController } from "./controllers/notFound.js";
import { NotFoundError } from "./tools/errors.js";
import { Router } from "./tools/router.js";
import { withLogs } from "./middleware/logging.js";

const app = new Router();

app.use(withLogs);

app.get("/assets/*", staticController);
app.get("/", homeController);
app.get("/books", booksController);

export function server(request) {
    const url = new URL(request.url);
    const ctx = { request, url, data: {} };

    try {
        return app.handle(ctx);
    } catch (error) {
        if (error instanceof NotFoundError) {
            console.log("Not found:", error.message);
            return notFoundController(ctx);
        }
        throw error;
    }
}
```

The request handler now only builds `ctx` and hands it to the router.

Refresh [http://localhost:8000/books][books] and look at the terminal.

```plaintext {linenos=false}

============================
GET /books
RESPONSE 200
============================

============================
GET /assets/styles.css
[2026-11-10 12:07:35] [GET] /assets/styles.css 200
RESPONSE 200
============================

============================
GET /assets/icon.svg
[2026-11-10 12:07:35] [GET] /assets/icon.svg 200
RESPONSE 200
============================
```

Every request is logged with its status, including our static files, because the middleware runs for every route.

Now open [http://localhost:8000/nope][nope].

```plaintext {linenos=false}
Not found: GET /nope doesn't match any known route
```

There is no block for `/nope`.
No route matches, so the router throws before the chain, and our middleware, ever runs.

## Awaiting the router {#await-router}

`withLogs` is `async`, so `app.handle` now returns a promise.
That causes a problem we can't see yet, because none of our controllers throws an error.

Let's make one throw, just for a moment.
Add `import { NotFoundError } from "../tools/errors.js";` to the top of **server/controllers/books.js**, and `throw new NotFoundError("Testing");` as the first line inside `booksController`.

Refresh [http://localhost:8000/books][books].
It is `500 Internal Server Error`, and the terminal shows the error.

```plaintext {linenos=false}

============================
GET /books
NotFoundError: Testing
    at booksController (file:///…/server/controllers/books.js:7:11)
    …
```

Our `catch` should have turned that into a `404`.
But the error happened inside `withLogs`, after an `await`, so it arrived inside the promise *after* our `try` block had already finished.
Notice that `withLogs` never logged a `RESPONSE`, either.

Update **server/main.js** to await the router.

**server/main.js**
```JavaScript{hl_lines="17 22"}
import { homeController } from "./controllers/home.js";
import { booksController } from "./controllers/books.js";
import { staticController } from "./controllers/static.js";
import { notFoundController } from "./controllers/notFound.js";
import { NotFoundError } from "./tools/errors.js";
import { Router } from "./tools/router.js";
import { withLogs } from "./middleware/logging.js";

const app = new Router();

app.use(withLogs);

app.get("/assets/*", staticController);
app.get("/", homeController);
app.get("/books", booksController);

export async function server(request) {
    const url = new URL(request.url);
    const ctx = { request, url, data: {} };

    try {
        return await app.handle(ctx);
    } catch (error) {
        if (error instanceof NotFoundError) {
            console.log("Not found:", error.message);
            return notFoundController(ctx);
        }
        throw error;
    }
}
```

`server` is now `async`, and `await app.handle(ctx)` waits for the response.
If the promise fails with an error, `await` throws it right there, inside our `try` block, where `catch` can see it.
`Deno.serve` is happy, because an `async` function returns a promise of a `Response`.

Refresh [http://localhost:8000/books][books].
It is `404 Not Found`, with our not-found page.

```plaintext {linenos=false}

============================
GET /books
Not found: Testing
```

Now remove the two lines you added to **server/controllers/books.js**, and check that [http://localhost:8000/books][books] shows the books again.

> Later, controllers will throw a `NotFoundError` of their own, for example when someone asks for a book that doesn't exist.
> Our request handler is ready for them.

## Conclusion

We paid off two more pieces of technical debt.
Our controllers, `render`, views and layout now share one context object, and each page's data goes on `ctx.data`.
The layout used it straight away, to mark the current page in the nav.
We replaced the logging placeholder with `withLogs`, our first middleware, which the router runs around every route's handler.
Because `withLogs` is `async`, we made our request handler `async` too, so it still catches errors from our controllers.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="17-18"}
my-app/
├── .vscode/
│   └── settings.json
├── db/
│   ├── app.db
│   ├── BOOKS-ATTRIBUTION.md
│   └── books.json
├── public/
│   ├── icon.svg
│   └── styles.css
├── server/
│   ├── controllers/
│   │   ├── books.js
│   │   ├── home.js
│   │   ├── notFound.js
│   │   └── static.js
│   ├── middleware/
│   │   └── logging.js
│   ├── models/
│   │   ├── books.js
│   │   └── db.js
│   ├── tools/
│   │   ├── errors.js
│   │   ├── render.js
│   │   └── router.js
│   ├── views/
│   │   ├── partials/
│   │   │   └── bookCard.js
│   │   ├── books.js
│   │   ├── home.js
│   │   ├── layout.js
│   │   └── notFound.js
│   └── main.js
├── tasks/
│   ├── db-create.js
│   ├── db-seed.js
│   └── server.js
├── .gitignore
├── deno.json
└── deno.lock
```

Our framework now has a router, a context object and middleware.
Next, we will start accepting input from our users.

## Challenge: Experiment with middleware

- Add an about page with a view, a controller and a route, and a second `navLink` for it in the layout. Does the highlight move when you click between the pages?
- Write a second middleware in **server/middleware/logging.js** that logs how long each request took, using `Date.now()` before and after `await next(ctx)`. Register it after `withLogs`. In what order do the log lines appear?
- Change `--hue` in **public/styles.css** to a few different values. When does `contrast-color()` pick black text, and when white?

> Undo your changes, so that your code matches the version above before moving on.

[previous exercise]: router-class/router/
[ctx]: views-and-controllers/controllers/#ctx
[options]: views-and-controllers/controllers/#options
[title]: views-and-controllers/controllers/#title
[passing data]: sqlite-and-models/books-page/#passing-data
[request logging]: http-servers/deno-serve/#request-logging
[destructuring]: javascript-fundamentals/books-data/#destructuring
[localhost]: http://localhost:8000/
[books]: http://localhost:8000/books
[nope]: http://localhost:8000/nope
[aria-current]: https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Attributes/aria-current
[conditional operator]: views-and-controllers/controllers/#title
[attribute selectors]: https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Selectors/Attribute_selectors
[contrast-color]: https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Values/color_value/contrast-color
[promise]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise
[await]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await
[async function]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function
[spread syntax]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax
[array destructuring]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring#array_destructuring
[arrow functions]: javascript-fundamentals/book-search/#arrow-functions
[map]: javascript-fundamentals/books-data/#map
[serve dir]: http-servers/static-files/#serve-dir

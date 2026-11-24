---
type: exercise
title: Middleware
weight: 4
author: Dr Graeme Stuart
---

In this exercise we will replace our logging placeholder with our first *middleware*.
Then we will use a second middleware to handle errors in one place, so that every response, even a `404`, is logged.

<!--more-->

> You should have one context object shared by every controller, view and the layout, from the [previous exercise][previous exercise].
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

Like the previous exercises in this unit, most of this one is a refactor.
By the end of this exercise you should be able to:

- Explain middleware as a function that runs for every route, and write one
- Use `async` and `await` where a function needs to wait for a result
- Handle errors in one place, with a middleware
- Show a server error page when something unexpected goes wrong, without losing the original error

## Where we are {#debt}

There is one more piece of debt in this unit.
Since our very first server we have [logged each request][request logging] with a `console.log` in **server/main.js**, under a placeholder that says it will change when we add middleware.

**server/main.js** also catches the `NotFoundError` from our router, and shows the not-found page.
We will move that job into a middleware too.

Start the server in development mode, if it isn't already running.

```bash {linenos=false}
deno task dev
```

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
```JavaScript{hl_lines="3 7 15-17 27-28 30-34"}
import { NotFoundError } from "./errors.js";

// Placeholder: this will need to change when we add URL parameters
export class Router {
    constructor() {
        this.routes = [];
        this.middleware = [];
    }

    register(pathname, handler) {
        const pattern = new URLPattern({ pathname });
        this.routes.push({ pattern, handler });
    }

    use(fn) {
        this.middleware.push(fn);
    }

    handle(ctx) {
        const { url } = ctx;
        const route = this.routes.find((route) => {
            return route.pattern.test(url);
        });
        if (!route) {
            throw new NotFoundError(`${url.pathname} doesn't match any known route`);
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
`[...this.middleware, route.handler]` makes that list, as a new array.
The `...` is [*spread syntax*][spread syntax]: it spreads the items of `this.middleware` out, one by one, into the new array, and `route.handler` goes on the end.
`this.middleware` itself doesn't change.

`chain` calls the first function in the list, and gives it a way to call the rest.

- `const [fn, ...remaining] = handlers;` is [array destructuring][array destructuring]. It works like the [object destructuring][destructuring] we already know, but by position: `fn` gets the first function. The `...remaining` is a [*rest element*][rest element], the opposite of spread: it collects all the other functions into a new array called `remaining`.
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

#### async and await {#async}

The response from `next` might not be ready straight away.
Our static controller, for example, returns a [promise][promise] of a response, because reading a file takes time.
We said in the [static files exercise][serve dir] that we would look at promises properly later; this is the first time we need to.

- A promise stands for a result that will arrive later.
- [`await`][await] pauses our function until that result has arrived. The server carries on handling other requests in the meantime.
- Only an [`async`][async function] function can use `await`, and an `async` function always returns a promise. `Deno.serve` waits for it.

That's all we need for now.

Nothing uses `withLogs` yet, so the website is unchanged.

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

app.register("/assets/*", staticController);
app.register("/", homeController);
app.register("/books", booksController);

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

app.register("/assets/*", staticController);
app.register("/", homeController);
app.register("/books", booksController);

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
Not found: /nope doesn't match any known route
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
    at booksController (file:///…/server/controllers/books.js:9:11)
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

app.register("/assets/*", staticController);
app.register("/", homeController);
app.register("/books", booksController);

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

## 404s in the log {#dispatch}

We saw that [http://localhost:8000/nope][nope] has no block in the terminal, so our log doesn't tell us that it was a `404`.
The router looks for the route *before* it runs the chain, so when there isn't one, our middleware never runs.

We can fix that by making the route lookup the last link of the chain.
Update **server/tools/router.js**.

**server/tools/router.js**
```JavaScript{hl_lines="20-23 31"}
import { NotFoundError } from "./errors.js";

// Placeholder: this will need to change when we add URL parameters
export class Router {
    constructor() {
        this.routes = [];
        this.middleware = [];
    }

    register(pathname, handler) {
        const pattern = new URLPattern({ pathname });
        this.routes.push({ pattern, handler });
    }

    use(fn) {
        this.middleware.push(fn);
    }

    handle(ctx) {
        return this.chain(ctx, [...this.middleware, (ctx) => this.dispatch(ctx)]);
    }

    dispatch(ctx) {
        const { url } = ctx;
        const route = this.routes.find((route) => {
            return route.pattern.test(url);
        });
        if (!route) {
            throw new NotFoundError(`${url.pathname} doesn't match any known route`);
        }
        return route.handler(ctx);
    }

    chain(ctx, handlers) {
        const [fn, ...remaining] = handlers;
        if (!remaining.length) return fn(ctx);
        const next = (ctx) => this.chain(ctx, remaining);
        return fn(ctx, next);
    }
}
```

The route lookup has moved out of `handle` into a new `dispatch` method.
`dispatch` finds the route, throws a `NotFoundError` if there isn't one, and calls the route's handler.

`handle` now passes `chain` our middleware, followed by an [arrow function][arrow functions] that calls `dispatch`.
So `dispatch` is the last link of the chain, and every request goes through our middleware first, even one that matches no route.

> We need the arrow function because `dispatch` uses `this`.
> As with `next`, `this` inside an arrow function is still our router.

Refresh [http://localhost:8000/nope][nope].
It is still `404 Not Found`, and now `withLogs` logs the request.

```plaintext {linenos=false}

============================
GET /nope
Not found: /nope doesn't match any known route
```

The `NotFoundError` travels back through `withLogs`, and on through the promise from `app.handle`.
Because `server` [awaits the router][await router], our `catch` still sees it.

But there is no `RESPONSE` line.
The error passes straight through `withLogs`, so it never gets as far as logging the status.

## Catching errors in middleware {#with-errors}

Our `catch` is in **server/main.js**, outside the chain, so a `404` never comes back through `withLogs`.
If we catch the error *inside* the chain instead, with a middleware, `withLogs` will see the not-found page like any other response.

Create **errors.js** in the **server/middleware** folder.

**server/middleware/errors.js**
```JavaScript
import { NotFoundError } from "../tools/errors.js";
import { notFoundController } from "../controllers/notFound.js";

export async function withErrors(ctx, next) {
    try {
        return await next(ctx);
    } catch (error) {
        ctx.error = error;
        if (error instanceof NotFoundError) return notFoundController(ctx);
        throw error;
    }
}
```

`withErrors` does the same job as the [`try...catch`][catch] in **server/main.js**, but as a middleware.

- It awaits the rest of the chain inside a `try` block. As when we [awaited the router][await router], the `await` matters: without it, an error from a controller would arrive after the `try` block had finished.
- If anything throws, it puts the error on `ctx.error`, so that other middleware can find out what went wrong.
- A `NotFoundError` is answered with our not-found page.
- Anything else is still a real bug, so we `throw` it again.

Now update **server/middleware/logging.js** to log the error.

**server/middleware/logging.js**
```JavaScript{hl_lines="7"}
// Placeholder: this will need to change when we add flash messages and sessions
export async function withLogs(ctx, next) {
    const { url, request } = ctx;
    console.log("\n============================");
    console.log(`${request.method} ${url.pathname}`);
    const response = await next(ctx);
    if (ctx.error) console.log("[error]", ctx.error.message);
    console.log("RESPONSE", response.status);
    console.log("============================");

    return response;
}
```

If there is an error on `ctx`, `withLogs` logs its message as an `[error]` line, before the status.
All of a request's logging now stays in one place.

Finally, update **server/main.js** to use `withErrors`, and remove its `try` block.

**server/main.js**
```JavaScript{hl_lines="6 11 17 20"}
import { homeController } from "./controllers/home.js";
import { booksController } from "./controllers/books.js";
import { staticController } from "./controllers/static.js";
import { Router } from "./tools/router.js";
import { withLogs } from "./middleware/logging.js";
import { withErrors } from "./middleware/errors.js";

const app = new Router();

app.use(withLogs);
app.use(withErrors);

app.register("/assets/*", staticController);
app.register("/", homeController);
app.register("/books", booksController);

export function server(request) {
    const url = new URL(request.url);
    const ctx = { request, url, data: {} };
    return app.handle(ctx);
}
```

We register `withErrors` straight after `withLogs`.
The middleware runs in the order we register it, so `withLogs` wraps `withErrors`, and sees every response that `withErrors` gives back.

**server/main.js** no longer needs `notFoundController` or `NotFoundError`.
`server` just builds `ctx` and returns `app.handle(ctx)`.
It doesn't need to be `async` any more, because it doesn't `await` anything.

Refresh [http://localhost:8000/nope][nope].
It is `404 Not Found`, and the terminal shows the whole story.

```plaintext {linenos=false}

============================
GET /nope
[error] /nope doesn't match any known route
RESPONSE 404
============================
```

## Error pages {#error-pages}

Our not-found page is one kind of error page, and we will want others.
Let's give the view and the controller names for errors in general.

Rename **server/views/notFound.js** to **error.js**, and update it.

**server/views/error.js**
```JavaScript
export function errorView(ctx) {
    const { data: { title, message } } = ctx;
    return `
        <section>
            <header>
                <h1>${title}</h1>
                <p>${message}</p>
            </header>
        </section>
    `;
}
```

`notFoundView` is now `errorView`.
Instead of always saying *Not found*, it takes the `title` and the `message` from `ctx.data`.

Rename **server/controllers/notFound.js** to **errors.js**, and update it.

**server/controllers/errors.js**
```JavaScript{hl_lines="2 7-8"}
import { render } from "../tools/render.js";
import { errorView } from "../views/error.js";

export function notFoundController(ctx) {
    ctx.status = 404;
    ctx.data.title = "Not found";
    ctx.data.message = "The requested resource was not found.";
    return render(errorView, ctx);
}
```

`notFoundController` keeps its name, and now puts the message on `ctx.data` as well as the title.

Our middleware has to import it from its new home.

**server/middleware/errors.js**
```JavaScript{hl_lines="2"}
import { NotFoundError } from "../tools/errors.js";
import { notFoundController } from "../controllers/errors.js";

export async function withErrors(ctx, next) {
    try {
        return await next(ctx);
    } catch (error) {
        ctx.error = error;
        if (error instanceof NotFoundError) return notFoundController(ctx);
        throw error;
    }
}
```

Refresh [http://localhost:8000/nope][nope].
The not-found page looks exactly as it did.

{{<figure src="images/not-found.webp" alt="The not-found page at /nope, with the heading Not found and the message The requested resource was not found, in our layout. The network panel shows the nope document with a 404 status" caption="The not-found page now comes from errorView, with a 404 status.">}}

## Unexpected errors {#unexpected}

`withErrors` still throws anything that isn't a `NotFoundError`.
Let's see what that looks like, just for a moment.
Add `throw new Error("Testing");` as the first line inside `homeController`, in **server/controllers/home.js**.

Refresh [http://localhost:8000/][localhost].
It is `500 Internal Server Error`, and all we get is a plain *Internal Server Error* from deno, with none of our layout.
The terminal shows the error, but `withLogs` never logs a `RESPONSE`.

Leave the `throw` in place for now, and add a second error class to **server/tools/errors.js**.

**server/tools/errors.js**
```JavaScript{hl_lines="3"}
export class NotFoundError extends Error {}

export class UnexpectedError extends Error {}
```

Add a `serverErrorController` to **server/controllers/errors.js**.

**server/controllers/errors.js**
```JavaScript{hl_lines="11-16"}
import { render } from "../tools/render.js";
import { errorView } from "../views/error.js";

export function notFoundController(ctx) {
    ctx.status = 404;
    ctx.data.title = "Not found";
    ctx.data.message = "The requested resource was not found.";
    return render(errorView, ctx);
}

export function serverErrorController(ctx) {
    ctx.status = 500;
    ctx.data.title = "Server error";
    ctx.data.message = "Something went wrong on the server.";
    return render(errorView, ctx);
}
```

It sets the status to `500`, with a title and a message, and renders the same `errorView`.
So a server error gets a proper page in our layout.

Update **server/middleware/errors.js** to use it.

**server/middleware/errors.js**
```JavaScript{hl_lines="1-2 10-11"}
import { NotFoundError, UnexpectedError } from "../tools/errors.js";
import { notFoundController, serverErrorController } from "../controllers/errors.js";

export async function withErrors(ctx, next) {
    try {
        return await next(ctx);
    } catch (error) {
        ctx.error = error;
        if (error instanceof NotFoundError) return notFoundController(ctx);
        ctx.error = new UnexpectedError("Unexpected error", { cause: error });
        return serverErrorController(ctx);
    }
}
```

Any error that isn't a `NotFoundError` is now wrapped in a new `UnexpectedError` before it goes on `ctx.error`, and answered with our server error page.

The [`Error` constructor][error cause] takes an options object as its second argument.
`{ cause: error }` keeps the original error inside the new one, so nothing about it is lost.

> More specific kinds of error can be handled with more conditions, before this fallback.

Finally, update **server/middleware/logging.js**.

**server/middleware/logging.js**
```JavaScript{hl_lines="1 9-10"}
import { UnexpectedError } from "../tools/errors.js";

// Placeholder: this will need to change when we add flash messages and sessions
export async function withLogs(ctx, next) {
    const { url, request } = ctx;
    console.log("\n============================");
    console.log(`${request.method} ${url.pathname}`);
    const response = await next(ctx);
    if (ctx.error instanceof UnexpectedError) console.log("[error]", ctx.error);
    else if (ctx.error) console.log("[error]", ctx.error.message);
    console.log("RESPONSE", response.status);
    console.log("============================");

    return response;
}
```

A `NotFoundError`'s message says all there is to know, so we still log just the message.
An `UnexpectedError` is a bug, so we log the whole error.
The [`else`][if else] means the second line only runs when the first condition is `false`.

Refresh [http://localhost:8000/][localhost].
It is still `500 Internal Server Error`, but now we see our server error page, in our layout.

{{<figure src="images/server-error.webp" alt="The home page address showing the heading Server error and the message Something went wrong on the server, in our layout. The network panel shows the localhost document with a 500 status" caption="An unexpected error now gets our server error page, with a 500 status.">}}

The terminal shows the `UnexpectedError` with its stack, and then the original error, with its stack, under `[cause]`.

```plaintext {linenos=false}

============================
GET /
[error] UnexpectedError: Unexpected error
    at withErrors (file:///…/server/middleware/errors.js:10:21)
    at Router.chain (file:///…/server/tools/router.js:38:16)
    ... 2 lines matching cause stack trace ...
    at Router.chain (file:///…/server/tools/router.js:38:16)
    at Router.handle (file:///…/server/tools/router.js:20:21)
    at server (file:///…/server/main.js:20:16)
    at nativeMapped (ext:deno_http/00_serve.ts:1:13754)
    at nativeDispatch (ext:deno_http/00_serve.ts:1:17522) {
  [cause]: Error: Testing
      at Object.homeController [as handler] (file:///…/server/controllers/home.js:5:11)
      at Router.dispatch (file:///…/server/tools/router.js:31:22)
      at file:///…/server/tools/router.js:20:67
      at Router.chain (file:///…/server/tools/router.js:36:39)
      at next (file:///…/server/tools/router.js:37:36)
      at withErrors (file:///…/server/middleware/errors.js:6:22)
      at Router.chain (file:///…/server/tools/router.js:38:16)
      at next (file:///…/server/tools/router.js:37:36)
      at withLogs (file:///…/server/middleware/logging.js:8:28)
      at Router.chain (file:///…/server/tools/router.js:38:16)
}
RESPONSE 500
============================
```

The `[cause]` tells us exactly where the problem is: line 5 of **server/controllers/home.js**.

Now remove the line you added to **server/controllers/home.js**, and check that [http://localhost:8000/][localhost] is `200 OK` again.

## Conclusion

We paid off the last piece of technical debt in this unit.
We replaced the logging placeholder with `withLogs`, our first middleware, which the router runs around every route's handler.
Because `withLogs` is `async`, we made our request handler `async` too, so it could still catch errors from our controllers.

Then we made the route lookup the last link of the chain, so every request goes through our middleware, and moved our error handling into a second middleware, `withErrors`.
Not-found pages and server errors are now answered in one place, and logged with the rest of the request.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="15 18-20 30"}
my-app/
├── .vscode/
│   └── settings.json
├── db/
│   ├── app.db
│   ├── BOOKS-ATTRIBUTION.md
│   └── books.json
├── public/
│   ├── default-cover.svg
│   ├── icon.svg
│   └── styles.css
├── server/
│   ├── controllers/
│   │   ├── books.js
│   │   ├── errors.js
│   │   ├── home.js
│   │   └── static.js
│   ├── middleware/
│   │   ├── errors.js
│   │   └── logging.js
│   ├── models/
│   │   ├── books.js
│   │   └── db.js
│   ├── tools/
│   │   ├── errors.js
│   │   ├── render.js
│   │   └── router.js
│   ├── views/
│   │   ├── books.js
│   │   ├── error.js
│   │   ├── home.js
│   │   └── layout.js
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

- Write a second middleware in **server/middleware/logging.js** that logs how long each request took, using `Date.now()` before and after `await next(ctx)`. Register it after `withLogs`. In what order do the log lines appear?
- Register `withErrors` *before* `withLogs`. What does the terminal show for [http://localhost:8000/nope][nope] now, and why?

> Undo your changes, so that your code matches the version above before moving on.

[previous exercise]: router-class/one-context/
[request logging]: http-servers/deno-serve/#request-logging
[localhost]: http://localhost:8000/
[books]: http://localhost:8000/books
[nope]: http://localhost:8000/nope
[destructuring]: javascript-fundamentals/books-data/#destructuring
[promise]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise
[await]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/await
[async function]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function
[spread syntax]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax
[rest element]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring#rest_property
[array destructuring]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring#array_destructuring
[arrow functions]: javascript-fundamentals/book-search/#arrow-functions
[map]: javascript-fundamentals/books-data/#map
[serve dir]: http-servers/static-files/#serve-dir
[catch]: router-class/router/#catch
[await router]: router-class/middleware/#await-router
[error cause]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/cause
[if else]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/if...else

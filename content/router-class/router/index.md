---
type: exercise
title: Routes and a router
weight: 2
author: Dr Graeme Stuart
---

In this exercise we will replace the growing chain of `if` statements in our request handler with a *router*.
Along the way we will write our first class.

<!--more-->

> You should have a books page showing 20 books from the [previous exercise][previous exercise].
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

This exercise is a *refactor*.
Our users won't see any difference, but our code will be much easier to grow.
By the end of this exercise you should be able to:

- Explain why a growing `if` chain is technical debt, and how a router pays it off
- Write a simple class with a constructor, a property and methods, and make an object from it with `new`
- Match a path against a pattern with `URLPattern`
- Route requests on the HTTP method as well as the path
- Throw an error, catch it with `try...catch`, and check what kind of error it is with `instanceof`

## Where we are {#debt}

We have three pages: home, books and not found.
Every time we added a page, we added another `if` to **server/main.js**.
When we built the books page we noticed that this is [technical debt][if chain], and the router placeholder has been reminding us to pay it off ever since.

This is our request handler as we left it.

**server/main.js**
```JavaScript
import { homeController } from "./controllers/home.js";
import { booksController } from "./controllers/books.js";
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
    if (url.pathname === "/books") {
        return booksController(ctx);
    }
    return notFoundController(ctx);
}
```

> The two placeholders mark the debt we will pay off in this unit.
> The router placeholder goes in this exercise, and the middleware placeholder in the next.

Look at what it does.

- Every page needs another `if`, and they all look almost the same.
- The handler has two jobs: it sets up the request (`url`, `ctx` and the log line), and it decides which controller to call.

This unit is about paying off [technical debt][technical debt].
In this exercise we will pay off the `if` chain.
In the next one we will tidy up how our controllers, views and layout share information.

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Open [http://localhost:8000/books][books] with the developer tools (**F12**) open on the **network** panel.
Keep it open; after each step we will check that nothing has changed.

## A Router class {#class}

Instead of a chain of `if` statements, we want a list of *routes*.
Each route says which method and path it handles, and which controller handles it.
Routing a request then means looking it up in that list.

We will keep the list, and the code that uses it, together in a *class*.

Create **router.js** in the **server/tools** folder.

**server/tools/router.js**
```JavaScript
// Placeholder: this will need to change when we add middleware and URL parameters
export class Router {
    constructor() {
        this.routes = [];
    }

    register(method, pathname, handler) {
        const pattern = new URLPattern({ pathname });
        this.routes.push({ method, pattern, handler });
    }
}
```

> The placeholder says our router will change.
> When we add middleware and URL parameters, it will need to do more.

### Classes {#classes}

A [class][classes] is a template for making objects.
Every object made from the same class has the same *properties* (values) and *methods* (functions), so they all behave in the same way.

- The [`constructor`][constructor] runs once, when a new object is made. Ours gives the new object an empty `routes` array.
- Inside a class, [`this`][this] means the object itself. `this.routes` is that object's own list of routes.
- `register` is a [method][methods], a function that belongs to the object. It adds one route to `this.routes`.

Each route is a small object with three properties: the method, a pattern for the path, and the handler.
[`push`][array push] adds it to the end of the array.

[`URLPattern`][urlpattern] is built into deno, and into modern browsers.
It describes the paths a route should match.
`{ pathname }` is shorthand for `{ pathname: pathname }`.
The `*` in a pattern such as `/assets/*` matches anything, so one route can cover every file in **public**.

Now update **server/main.js** to make a router and register our routes.

**server/main.js**
```JavaScript{hl_lines="5 7 9-11"}
import { homeController } from "./controllers/home.js";
import { booksController } from "./controllers/books.js";
import { staticController } from "./controllers/static.js";
import { notFoundController } from "./controllers/notFound.js";
import { Router } from "./tools/router.js";

const app = new Router();

app.register("GET", "/assets/*", staticController);
app.register("GET", "/", homeController);
app.register("GET", "/books", booksController);

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
    if (url.pathname === "/books") {
        return booksController(ctx);
    }
    return notFoundController(ctx);
}
```

[`new`][new] makes a new object from a class, and runs its constructor.
We call our router `app`, because its routes describe our whole application.

Notice that we pass each controller to `register` without calling it, just as we [passed views to `render`][first-class function].
The router will call them later.

> The routes are registered once, when **server/main.js** is first loaded, not every time a request arrives.

Nothing uses the routes yet, so the `if` statements are still doing the work.
Refresh [http://localhost:8000/books][books] to check that nothing has changed.

## Handling a request {#handle}

Now our router needs to find the right route for each request, and call its handler.

Add a `handle` method to the class.

**server/tools/router.js**
```JavaScript{hl_lines="12-21"}
// Placeholder: this will need to change when we add middleware and URL parameters
export class Router {
    constructor() {
        this.routes = [];
    }

    register(method, pathname, handler) {
        const pattern = new URLPattern({ pathname });
        this.routes.push({ method, pattern, handler });
    }

    handle(ctx) {
        const { url, request } = ctx;
        const route = this.routes.find((route) => {
            return route.method === request.method && route.pattern.test(url);
        });
        if (!route) {
            return;
        }
        return route.handler(ctx);
    }
}
```

`handle` takes our [context object][ctx], and takes the `url` and the `request` out of it.

[`find`][array find] is like the [`filter`][filter] we used in the book search, but it gives us the first matching item rather than an array of them.
The [arrow function][arrow functions] we pass to it checks two things.

- `route.method === request.method` checks the HTTP method; every page we visit is a `GET` request.
- [`route.pattern.test(url)`][urlpattern test] is `true` when the URL matches the route's pattern.

The [`&&`][logical and] between them means a route only matches when *both* are `true`.

If no route matches, `find` gives us `undefined`.
[`!`][logical not] means *not*, so `!route` is `true` when there is no route.
Our `return;` then returns `undefined` too, which tells the caller we couldn't help.
Otherwise we call the route's handler with `ctx`, and return the `Response` it gives us.

Now update **server/main.js** to let the router decide which controller to call.

**server/main.js**
```JavaScript{hl_lines="15 20"}
import { homeController } from "./controllers/home.js";
import { booksController } from "./controllers/books.js";
import { staticController } from "./controllers/static.js";
import { notFoundController } from "./controllers/notFound.js";
import { Router } from "./tools/router.js";

const app = new Router();

app.register("GET", "/assets/*", staticController);
app.register("GET", "/", homeController);
app.register("GET", "/books", booksController);

export function server(request) {
    const url = new URL(request.url);
    const ctx = { request, url };

    // Placeholder: this will need to change when we add middleware
    console.log(request.method, url.pathname);

    return app.handle(ctx) ?? notFoundController();
}
```

The `if` statements have gone, and so has the router placeholder.
`handle` needs the `url`, so we add it to `ctx`.
This is the context pattern doing its job: `ctx` grows, and nothing else has to change.

If the router returns `undefined`, the [`??`][nullish coalescing] falls back to our not-found controller.

> The middleware placeholder is still there.
> We will replace it in the next exercise.

Check our pages.

- [http://localhost:8000/][localhost] is `200 OK`, with its title.
- [http://localhost:8000/books][books] is `200 OK`, with the books and their covers, so our static files work too.
- [http://localhost:8000/nope][nope] is `404 Not Found`.

### Why is this better? {#better}

Adding a page is now one line in the list of routes.
The request handler doesn't change when we add a page, and each line says exactly which method and path a controller handles.
The routing logic lives in one place, in a class we can improve later.

## A get method {#get}

Every route we register says `"GET"`.
Most routes in a web application are `GET` routes, so let's make them shorter to write.

Add a `get` method to the class.

**server/tools/router.js**
```JavaScript{hl_lines="12-14"}
// Placeholder: this will need to change when we add middleware and URL parameters
export class Router {
    constructor() {
        this.routes = [];
    }

    register(method, pathname, handler) {
        const pattern = new URLPattern({ pathname });
        this.routes.push({ method, pattern, handler });
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
            return;
        }
        return route.handler(ctx);
    }
}
```

The `...args` in the parameter list is a [*rest parameter*][rest parameters].
It collects all the arguments into an array called `args`, however many there are.
The `...args` in the call to `register` is [*spread syntax*][spread syntax].
It spreads the array back out into separate arguments.

So `app.get("/", homeController)` calls `this.register("GET", "/", homeController)`.

Update **server/main.js** to use it.

**server/main.js**
```JavaScript{hl_lines="9-11"}
import { homeController } from "./controllers/home.js";
import { booksController } from "./controllers/books.js";
import { staticController } from "./controllers/static.js";
import { notFoundController } from "./controllers/notFound.js";
import { Router } from "./tools/router.js";

const app = new Router();

app.get("/assets/*", staticController);
app.get("/", homeController);
app.get("/books", booksController);

export function server(request) {
    const url = new URL(request.url);
    const ctx = { request, url };

    // Placeholder: this will need to change when we add middleware
    console.log(request.method, url.pathname);

    return app.handle(ctx) ?? notFoundController();
}
```

Each route is now shorter, and reads almost like a sentence: get `/books` with the books controller.

Refresh [http://localhost:8000/books][books] to check that nothing has changed.

## Not found is an error {#not-found-error}

When no route matches, `handle` returns `undefined`.
That works, but it is easy to forget the `??` in **server/main.js**, and `undefined` doesn't tell anyone *why* nothing came back.

A request for a page that doesn't exist is an *exceptional* situation, and JavaScript has a way to report those: errors.

### Throwing an error {#throw}

Create **errors.js** in the **server/tools** folder.

**server/tools/errors.js**
```JavaScript
export class NotFoundError extends Error {}
```

`NotFoundError` is a new kind of [`Error`][error].
[`extends`][extends] means it is an `Error` in every way, with nothing added.
Its name tells us what went wrong.

Update **server/tools/router.js** to throw a `NotFoundError` when no route matches.

**server/tools/router.js**
```JavaScript{hl_lines="1 24-26"}
import { NotFoundError } from "./errors.js";

// Placeholder: this will need to change when we add middleware and URL parameters
export class Router {
    constructor() {
        this.routes = [];
    }

    register(method, pathname, handler) {
        const pattern = new URLPattern({ pathname });
        this.routes.push({ method, pattern, handler });
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
        return route.handler(ctx);
    }
}
```

[`throw`][throw] stops the function straight away, and sends the error back to whoever called it, and on to *their* caller, until something catches it.
Our message, built with a template literal, says which request failed.

Nothing catches it yet.
Refresh [http://localhost:8000/nope][nope].

```http
HTTP/1.1 500 Internal Server Error
content-length: 21
```

The error reached `Deno.serve`, which answered with a `500 Internal Server Error`, and printed the error in the terminal.

```plaintext {linenos=false}
GET /nope
NotFoundError: GET /nope doesn't match any known route
    at Router.handle (file:///…/server/tools/router.js:24:19)
    at server (file:///…/server/main.js:20:16)
```

We have seen a `500` caused by our own code before, when our [handler returned `undefined`][implicit undefined].
This one is more helpful, because the message says exactly what happened.

### Catching the error {#catch}

Update **server/main.js** to catch the error.

**server/main.js**
```JavaScript{hl_lines="5 21-29"}
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
    const ctx = { request, url };

    // Placeholder: this will need to change when we add middleware
    console.log(request.method, url.pathname);

    try {
        return app.handle(ctx);
    } catch (error) {
        if (error instanceof NotFoundError) {
            console.log("Not found:", error.message);
            return notFoundController();
        }
        throw error;
    }
}
```

[`try...catch`][try catch] runs the code in the `try` block.
If anything in there throws an error, the `catch` block runs instead, with the error.

[`instanceof`][instanceof] checks what kind of error we caught.

- If it is a `NotFoundError`, we log its message and return our not-found page.
- Anything else is a real bug, so we `throw` it again. `Deno.serve` will still give a `500`, and we will still see the error in the terminal.

Refresh [http://localhost:8000/nope][nope].
It is `404 Not Found` again, with our not-found page.
The terminal now says why.

```plaintext {linenos=false}
GET /nope
Not found: GET /nope doesn't match any known route
GET /assets/styles.css
[2026-11-10 12:07:34] [GET] /assets/styles.css 200
GET /assets/icon.svg
[2026-11-10 12:07:34] [GET] /assets/icon.svg 200
```

The router reports *what* went wrong, and **server/main.js** decides what to do about it.

## Conclusion

We paid off the technical debt of our `if` chain.
Our routes are now a list, kept in a `Router` object, our first class.
The router matches the method as well as the path, with `URLPattern`, and a `get` method makes routes short to write.
When nothing matches, the router throws a `NotFoundError`, and **server/main.js** catches it and shows the not-found page.

Our users wouldn't notice any of this.
But adding a page is now one line in **server/main.js**.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="21 23"}
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

In the next exercise we will give every controller, view and the layout the same context object, and replace our logging placeholder with our first *middleware*.

## Challenge: Try out the router

- Try the [about page challenge][about challenge] again. How many lines of **server/main.js** does it take now?
- Visit a few pages and watch the terminal. Which method does the browser use for the page itself, and for its stylesheet and images? Check your answer in the **network** panel.
- Register `app.get("/*", homeController);` *before* the other routes. What happens to [http://localhost:8000/books][books], and why does the order of the routes matter?

> Undo your changes, so that your code matches the starting point of the next exercise.

[previous exercise]: sqlite-and-models/books-page/
[if chain]: sqlite-and-models/books-page/#if-chain
[technical debt]: views-and-controllers/controllers/#technical-debt
[books]: http://localhost:8000/books
[classes]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Using_classes
[constructor]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/constructor
[this]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this
[methods]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Method_definitions
[array push]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/push
[urlpattern]: https://developer.mozilla.org/en-US/docs/Web/API/URLPattern
[new]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new
[first-class function]: views-and-controllers/controllers/#view-functions
[ctx]: views-and-controllers/controllers/#ctx
[array find]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/find
[filter]: javascript-fundamentals/book-search/#filter
[arrow functions]: javascript-fundamentals/book-search/#arrow-functions
[urlpattern test]: https://developer.mozilla.org/en-US/docs/Web/API/URLPattern/test
[logical and]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Logical_AND
[logical not]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Logical_NOT
[nullish coalescing]: javascript-fundamentals/books-data/#missing-year
[localhost]: http://localhost:8000/
[nope]: http://localhost:8000/nope
[rest parameters]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/rest_parameters
[spread syntax]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax
[error]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error
[extends]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/extends
[throw]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/throw
[implicit undefined]: http-servers/missing-pages/#implicit-undefined
[try catch]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/try...catch
[instanceof]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/instanceof
[about challenge]: http-servers/missing-pages/#about-challenge

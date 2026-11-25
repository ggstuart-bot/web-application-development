---
type: exercise
title: A form that posts, then redirect
weight: 1
author: Dr Graeme Stuart
---

In this exercise we will add a form that makes a new shelf.
We will send it with a `POST` request, answer it with a redirect, and teach our router to tell a `GET` from a `POST`.

<!--more-->

> You should have book pages, shelf pages and shelf cards with their covers, from the previous unit.
> Your database should hold the seeded books and the three seeded shelves, **classics**, **fantasy** and **to read**. If you have been experimenting, run `deno task db:reset` to get back to the seed data.
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

So far, every request our server has handled has only read data.
Even our [search form][search form] only asks for a page, with the search in the query string.
In this exercise we will add a form that *changes* the database.
By the end of this exercise you should be able to:

- Write a form that sends its data in the body of a `POST` request
- Read a form's data on the server with `request.formData()`
- Explain why refreshing the page after a `POST` can repeat it
- Answer a `POST` with a `303 See Other` redirect, the *Post/Redirect/Get* pattern
- Match routes by their method as well as their path
- Give each controller one job

## A new shelf form {#new-shelf-form}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Open [http://localhost:8000/shelves][shelves].
We can see the three seeded shelves, but there's no way to make a new one.

Update **server/views/shelves.js** to add a form in an [`<aside>`][aside], after the section.

**server/views/shelves.js**
```JavaScript {hl_lines="16-23"}
import { html } from "../tools/html.js";
import { DEFAULT_COVER } from "./partials/bookCard.js";
import { cardGrid } from "./partials/cardGrid.js";

export function shelvesView(ctx) {
    const { data: { shelves } } = ctx;

    return html`
        <section>
            <header>
                <h1>Shelves</h1>
            </header>
            <h2>All shelves</h2>
            ${cardGrid(shelves.map(shelfCard), "No shelves yet")}
        </section>
        <aside>
            <h2>New shelf</h2>
            <form class="secondary" method="POST" action="/shelves">
                <label for="title">Title</label>
                <input id="title" name="title">
                <button>create</button>
            </form>
        </aside>
    `;
}

// A shelf as a card in a .grid list, with a fanned row of its first few covers.
// Shelf lists always sit under an <h2>, so the title is an <h3>
function shelfCard(shelf) {
    return html`
        <li class="card shelf">
            ${shelf.thumbnails.length
                ? html`<div class="covers">${shelf.thumbnails.map(coverImage)}</div>`
                : html`<p class="covers">No books yet</p>`}
            <div>
                <div class="row">
                    <h3><a href="/shelves/${shelf.id}">${shelf.title}</a></h3>
                    <p>${shelf.bookCount} ${shelf.bookCount === 1 ? "book" : "books"}</p>
                </div>
            </div>
        </li>
    `;
}

function coverImage(book) {
    return html`<img src="${
        book.thumbnailUrl ?? DEFAULT_COVER
    }" alt="" width="64" height="96" loading="lazy">`;
}
```

- An `<aside>` holds content that sits beside the main content of the page, here a form for making a new shelf.
- The form has the `secondary` class, the same dark panel style as our header and footer.
- The [`<label>`][label] is joined to its [`<input>`][input] by `for="title"` and `id="title"`. Clicking the label puts the cursor in the box.
- The input's `name` is `title`. When the form is sent, the value is sent with that name.

The important part is the form's [`method`][form method] and `action`.
Our search form uses `method="GET"`, so the browser puts its values in the query string and visits the URL, just like following a link.
This form uses `method="POST"`.
The browser sends a `POST` request to `/shelves`, the same address as the page, with the title in the *body* of the request rather than in the URL.

`GET` is for asking for something.
`POST` is for sending something that the server should act on, such as a new shelf.

Refresh [http://localhost:8000/shelves][shelves], with the developer tools (**F12**) open on the **network** panel.
The form appears under the list of shelves.
Type **poetry** into the box and press **create**.

The network panel shows a `POST` request to `/shelves`.
Click on it, and look at the **payload** (or **request**) tab.
The browser sent `title=poetry` in the body of the request.
The address bar still shows `http://localhost:8000/shelves`, with no query string.

The page comes back unchanged, and there is no **poetry** shelf.
Our router only looks at the path, so the `POST` went to `shelvesController`, just like a `GET`, and it ignored the title.

```plaintext {linenos=false}

============================
POST /shelves
RESPONSE 200
============================
```

## An aside beside the shelves {#aside-styles}

The form is hard to find under the list, and it looks like a plain dark box.
Add these rules to **public/styles.css**, after the `search > form` rule.

**public/styles.css**
```css {linenostart="200", hl_lines="1-20"}
/* A secondary form is padded with rounded corners */
form.secondary {
    padding: 1.5rem;
    border-radius: 0.5rem;
}

/* On wide screens main becomes a two-column grid, so an aside sits beside the section */
@media (width > 50rem) {
    main {
        display: grid;
        grid-template-columns: 1fr auto;
        /* A gap and a line separate the aside from the section */
        > aside {
            margin-inline-start: 1rem;
            padding-inline-start: 1rem;
            border-inline-start: 1px solid;
        }
    }
}

```

- `form.secondary` only matches forms with the `secondary` class. The `.secondary` rule already makes them dark with light text, so we just add some padding and rounded corners to make a panel.
- [`@media (width > 50rem)`][media range] only applies its rules when the window is wider than `50rem`. On a wide screen, `main` becomes a grid with two columns.
- `1fr auto` gives the first column all the spare space, and makes the second only as wide as its content needs. The section of shelves takes the first column and the aside takes the second.
- `> aside` is nested, so it matches an `<aside>` that is a child of `main`. A margin, some padding and a border on its starting edge separate it from the shelves.

On a narrow screen none of this applies.
`main` stays a single column, and the aside drops below the shelves.

Refresh [http://localhost:8000/shelves][shelves].
The form is now a panel beside the shelves.
Make the window narrower and watch the form drop below them.

{{<figure src="images/new-shelf-form.webp" alt="The shelves page, with three shelf cards in a grid and a dark New shelf panel beside them, holding a Title box and a create button" caption="The new shelf form sits beside the shelves on a wide screen.">}}

## Creating a shelf {#create-shelf}

Now we can read the form.
Update `shelvesController` in **server/controllers/shelves.js**.

**server/controllers/shelves.js**
```JavaScript {hl_lines="1 8-13"}
import { createShelf, getShelfById, getShelves } from "../models/shelves.js";
import { addThumbnails, getBooksByShelfId } from "../models/shelfBooks.js";
import { render } from "../tools/render.js";
import { NotFoundError } from "../tools/errors.js";
import { shelvesView } from "../views/shelves.js";
import { shelfView } from "../views/shelf.js";

export async function shelvesController(ctx) {
    const { request } = ctx;
    if (request.method === "POST") {
        const formData = await request.formData();
        createShelf({ title: formData.get("title") });
    }
    ctx.data.shelves = addThumbnails(getShelves());
    ctx.data.title = "Shelves";
    return render(shelvesView, ctx);
}

export function shelfController(ctx) {
    const { params: { shelfId } } = ctx;
    const shelf = getShelfById(Number(shelfId));
    if (!shelf) {
        throw new NotFoundError(`Shelf ${shelfId} not found`);
    }
    ctx.data.shelf = shelf;
    ctx.data.title = shelf.title;
    ctx.data.books = getBooksByShelfId(shelf.id);
    return render(shelfView, ctx);
}
```

- `request.method` is `"GET"` or `"POST"`. When it's a `POST`, we read the form.
- [`request.formData()`][form data] reads the body of the request as a [`FormData`][form data object] object. Its [`get`][form data get] method gives us the value sent with a name, here the `title`.
- We pass the title to `createShelf`, which our seed task already uses to make the seeded shelves.
- Either way, the controller carries on and shows the shelves page.

The body of a request arrives over the network, and may not all be here yet.
`request.formData()` returns a promise, so we `await` it, and the controller becomes `async`, just like [`withLogs`][async].

Go back to [http://localhost:8000/shelves][shelves] and create a shelf called **poetry**.
This time a **poetry** card appears in the list, with **No books yet**.

Now refresh the page.
The browser warns you that it will send the form again.
If you agree, you get a second **poetry** shelf.
Refresh again, and you get a third.

The page we are looking at is the response to a `POST`.
Refreshing a page repeats the request that fetched it, so every refresh repeats the `POST`, and makes another shelf.
The back button can do the same thing.

Reset the database to get rid of the copies.

```bash {linenos=false}
deno task db:reset
```

## Post/Redirect/Get {#post-redirect-get}

The answer is to never show a page in response to a `POST` that changed something.
Instead, we tell the browser to go somewhere else.

Create a new file, **server/tools/redirect.js**.

**server/tools/redirect.js**
```JavaScript {hl_lines="1-4"}
export function redirect(location) {
    const headers = new Headers({ location });
    return new Response(null, { headers, status: 303 });
}
```

`redirect` builds a response with no body.

- The status is [`303 See Other`][303]. It tells the browser to go and fetch another page with a `GET`.
- The [`location`][location] header says which page.

Every form we add from now on will need a redirect, so it gets its own module in **server/tools/**.

Update **server/controllers/shelves.js** to redirect to the new shelf.

**server/controllers/shelves.js**
```JavaScript {hl_lines="3 13-14"}
import { createShelf, getShelfById, getShelves } from "../models/shelves.js";
import { addThumbnails, getBooksByShelfId } from "../models/shelfBooks.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { NotFoundError } from "../tools/errors.js";
import { shelvesView } from "../views/shelves.js";
import { shelfView } from "../views/shelf.js";

export async function shelvesController(ctx) {
    const { request } = ctx;
    if (request.method === "POST") {
        const formData = await request.formData();
        const shelf = createShelf({ title: formData.get("title") });
        return redirect(`/shelves/${shelf.id}`);
    }
    ctx.data.shelves = addThumbnails(getShelves());
    ctx.data.title = "Shelves";
    return render(shelvesView, ctx);
}

export function shelfController(ctx) {
    const { params: { shelfId } } = ctx;
    const shelf = getShelfById(Number(shelfId));
    if (!shelf) {
        throw new NotFoundError(`Shelf ${shelfId} not found`);
    }
    ctx.data.shelf = shelf;
    ctx.data.title = shelf.title;
    ctx.data.books = getBooksByShelfId(shelf.id);
    return render(shelfView, ctx);
}
```

- `createShelf` gives back the new row, with its `id` and `title`.
- We `return` a redirect to the new shelf's page, so the controller never gets as far as showing the shelves page for a `POST`.

Go to [http://localhost:8000/shelves][shelves], with the **network** panel open, and create a shelf called **poetry**.
You land on the new shelf's page at [http://localhost:8000/shelves/4][shelf 4].

> Tick **preserve log** in the network panel, so the list isn't cleared when the browser moves to a new page.

The network panel shows two requests.
The `POST` to `/shelves` got a `303` response, with a `location` header of `/shelves/4`.
The browser followed it with a `GET` to `/shelves/4`, which got a `200`.

{{<figure src="images/post-redirect-get.webp" alt="The new poetry shelf page, with the network panel below listing a POST to shelves with status 303, then a GET to shelves/4 with status 200" caption="The POST gets a 303, and the browser follows it with a GET.">}}

```plaintext {linenos=false}

============================
POST /shelves
RESPONSE 303
============================

============================
GET /shelves/4
RESPONSE 200
============================
```

> You will see the requests for the stylesheet and icon in the terminal too.

Now refresh the page.
There is no warning, and no second shelf.
The page we are looking at was fetched with a `GET`, so refreshing it only repeats the `GET`.

This is called [Post/Redirect/Get][prg]: a `POST` changes something, the server answers with a redirect, and the browser fetches the result with a `GET`.
We will use it for every form that changes data.

Try a shelf called `<em>poetry</em>`.
The shelf page shows the tags as text, because every view uses our [`html` helper][html].

## Matching the method {#method}

Our router still ignores the method.
`shelvesController` has to check it for itself, and any `POST` to any page goes to that page's controller.

Update **server/tools/router.js** so that each route has a method as well as a pathname.

**server/tools/router.js**
```JavaScript {hl_lines="9 11 23 25 28-30"}
import { NotFoundError } from "./errors.js";

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

    handle(ctx) {
        return this.chain(ctx, [...this.middleware, (ctx) => this.dispatch(ctx)]);
    }

    dispatch(ctx) {
        const { url, request } = ctx;
        const route = this.routes.find((route) => {
            return route.method === request.method && route.pattern.test(url);
        });
        if (!route) {
            throw new NotFoundError(
                `${request.method} ${url.pathname} doesn't match any known route`,
            );
        }
        ctx.params = route.pattern.exec(url).pathname.groups;
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

- `register` takes the method first, and keeps it with the route.
- In `dispatch`, a route only matches when its method *and* its pattern both match the request.
- If nothing matches, the `NotFoundError` message names the method as well as the path.

Every call to `register` now needs a method.
Update **server/main.js**.

**server/main.js**
```JavaScript {hl_lines="14-20"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import { shelfController, shelvesController } from "./controllers/shelves.js";
import { staticController } from "./controllers/static.js";
import { Router } from "./tools/router.js";
import { withLogs } from "./middleware/logging.js";
import { withErrors } from "./middleware/errors.js";

const app = new Router();

app.use(withLogs);
app.use(withErrors);

app.register("GET", "/assets/*", staticController);
app.register("GET", "/", homeController);
app.register("GET", "/books", booksController);
app.register("GET", "/books/:bookId", bookController);
app.register("GET", "/shelves", shelvesController);
app.register("POST", "/shelves", shelvesController);
app.register("GET", "/shelves/:shelfId", shelfController);

export function server(request) {
    const url = new URL(request.url);
    const ctx = { request, url, data: {} };
    return app.handle(ctx);
}
```

Every page is registered for `"GET"`.
The form still needs to reach `shelvesController`, so `/shelves` is registered for `"POST"` as well, with the same controller for now.

Create another shelf from [http://localhost:8000/shelves][shelves].
It works just as before.

### Changing the form {#wrong-method}

What happens when a `POST` arrives at a page that only has a `GET` route?
We can't type a `POST` into the address bar, but we can change our form in the browser's developer tools.

1. Open [http://localhost:8000/shelves][shelves], and open the developer tools (**F12**) on the **elements** panel.
2. Find the new shelf form: `<form class="secondary" method="POST" action="/shelves">`.
3. Double-click the `action` value, change it to `/books`, and press **Enter**.
4. Type a title and press **create**.

This time you get our **Not found** page.

```plaintext {linenos=false}

============================
POST /books
[error] POST /books doesn't match any known route
RESPONSE 404
============================
```

Before we changed the router, the same `POST` would have shown the books page.
Now a request with the wrong method is not found, just like a request for a path we don't know.

> Changes in the elements panel only last until the page is reloaded. Refresh the page to get the real form back.

## A controller for each job {#create-shelf-controller}

Now that the router can tell a `GET` from a `POST`, the two requests don't need to share a controller.
Update **server/controllers/shelves.js**.

**server/controllers/shelves.js**
```JavaScript {hl_lines="9 15-21"}
import { createShelf, getShelfById, getShelves } from "../models/shelves.js";
import { addThumbnails, getBooksByShelfId } from "../models/shelfBooks.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { NotFoundError } from "../tools/errors.js";
import { shelvesView } from "../views/shelves.js";
import { shelfView } from "../views/shelf.js";

export function shelvesController(ctx) {
    ctx.data.shelves = addThumbnails(getShelves());
    ctx.data.title = "Shelves";
    return render(shelvesView, ctx);
}

export async function createShelfController(ctx) {
    const { request } = ctx;
    const formData = await request.formData();
    const shelf = createShelf({ title: formData.get("title") });
    return redirect(`/shelves/${shelf.id}`);
}

export function shelfController(ctx) {
    const { params: { shelfId } } = ctx;
    const shelf = getShelfById(Number(shelfId));
    if (!shelf) {
        throw new NotFoundError(`Shelf ${shelfId} not found`);
    }
    ctx.data.shelf = shelf;
    ctx.data.title = shelf.title;
    ctx.data.books = getBooksByShelfId(shelf.id);
    return render(shelfView, ctx);
}
```

- `shelvesController` goes back to only listing the shelves. It no longer checks the method, and it doesn't need to be `async`.
- The new `createShelfController` handles the `POST`. It reads the form, creates the shelf and redirects to it.

Update **server/main.js** to send the `POST` to the new controller.

**server/main.js**
```JavaScript {hl_lines="3-7 23"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import {
    createShelfController,
    shelfController,
    shelvesController,
} from "./controllers/shelves.js";
import { staticController } from "./controllers/static.js";
import { Router } from "./tools/router.js";
import { withLogs } from "./middleware/logging.js";
import { withErrors } from "./middleware/errors.js";

const app = new Router();

app.use(withLogs);
app.use(withErrors);

app.register("GET", "/assets/*", staticController);
app.register("GET", "/", homeController);
app.register("GET", "/books", booksController);
app.register("GET", "/books/:bookId", bookController);
app.register("GET", "/shelves", shelvesController);
app.register("POST", "/shelves", createShelfController);
app.register("GET", "/shelves/:shelfId", shelfController);

export function server(request) {
    const url = new URL(request.url);
    const ctx = { request, url, data: {} };
    return app.handle(ctx);
}
```

Each controller does one job, and **server/main.js** shows which controller handles which request.

Create a shelf from [http://localhost:8000/shelves][shelves] again, and check that you still land on its page.

## get and post {#get-post}

Writing `"GET"` on every route is repetitive, and a typo like `"GTE"` would quietly give us a route that never matches.
Add two shortcuts to the `Router` class in **server/tools/router.js**.

**server/tools/router.js**
```JavaScript {hl_lines="18-24"}
import { NotFoundError } from "./errors.js";

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
    post(...args) {
        this.register("POST", ...args);
    }

    handle(ctx) {
        return this.chain(ctx, [...this.middleware, (ctx) => this.dispatch(ctx)]);
    }

    dispatch(ctx) {
        const { url, request } = ctx;
        const route = this.routes.find((route) => {
            return route.method === request.method && route.pattern.test(url);
        });
        if (!route) {
            throw new NotFoundError(
                `${request.method} ${url.pathname} doesn't match any known route`,
            );
        }
        ctx.params = route.pattern.exec(url).pathname.groups;
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

`get` and `post` take any arguments and pass them on to `register`, with the method filled in.
`...args` is a [rest parameter][rest parameters]: it collects all the arguments into an array.
`...args` in the call to `register` spreads them back out again, just as we [spread the middleware][chain] into the list of handlers.

Update **server/main.js** to use them.

**server/main.js**
```JavaScript {hl_lines="18-24"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import {
    createShelfController,
    shelfController,
    shelvesController,
} from "./controllers/shelves.js";
import { staticController } from "./controllers/static.js";
import { Router } from "./tools/router.js";
import { withLogs } from "./middleware/logging.js";
import { withErrors } from "./middleware/errors.js";

const app = new Router();

app.use(withLogs);
app.use(withErrors);

app.get("/assets/*", staticController);
app.get("/", homeController);
app.get("/books", booksController);
app.get("/books/:bookId", bookController);
app.get("/shelves", shelvesController);
app.post("/shelves", createShelfController);
app.get("/shelves/:shelfId", shelfController);

export function server(request) {
    const url = new URL(request.url);
    const ctx = { request, url, data: {} };
    return app.handle(ctx);
}
```

Now each route reads like the request it handles: `app.get("/books", ...)` and `app.post("/shelves", ...)`.

Check that every page still works, and that you can still create a shelf.

## Conclusion

Our application can now change its data from the browser.
The new shelf form sends a `POST` request, and `createShelfController` reads it with `request.formData()`, creates the shelf and redirects to it with a `303 See Other`.
Because the browser fetches the new page with a `GET`, refreshing it never repeats the `POST`.
Our router matches the method as well as the path, and `get` and `post` keep **server/main.js** easy to read.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="31"}
my-app/
├── .vscode/
│   └── settings.json
├── db/
│   ├── app.db
│   ├── BOOKS-ATTRIBUTION.md
│   ├── books.json
│   └── seed.json
├── public/
│   ├── default-cover.svg
│   ├── icon.svg
│   └── styles.css
├── server/
│   ├── controllers/
│   │   ├── books.js
│   │   ├── errors.js
│   │   ├── home.js
│   │   ├── shelves.js
│   │   └── static.js
│   ├── middleware/
│   │   ├── errors.js
│   │   └── logging.js
│   ├── models/
│   │   ├── books.js
│   │   ├── db.js
│   │   ├── shelfBooks.js
│   │   └── shelves.js
│   ├── tools/
│   │   ├── errors.js
│   │   ├── html.js
│   │   ├── redirect.js
│   │   ├── render.js
│   │   └── router.js
│   ├── views/
│   │   ├── partials/
│   │   │   ├── bookCard.js
│   │   │   ├── cardGrid.js
│   │   │   └── searchForm.js
│   │   ├── book.js
│   │   ├── books.js
│   │   ├── error.js
│   │   ├── home.js
│   │   ├── layout.js
│   │   ├── shelf.js
│   │   └── shelves.js
│   └── main.js
├── tasks/
│   ├── db-create.js
│   ├── db-seed.js
│   └── server.js
├── .gitignore
├── deno.json
└── deno.lock
```

## Challenge: Only a GET

These are questions to investigate, not changes to keep.

- Create a shelf with an empty title. What happens? Where should we stop it? We will come back to this when we validate our users' input.
- With the **network** panel open, create a shelf and click the `POST` request. Find the `location` header in the response. What would happen if it pointed at a shelf that doesn't exist, such as `/shelves/999`?
- Change the status in **server/tools/redirect.js** to `200`. Create a shelf. What do you see, and why doesn't the browser go to the new shelf?
- Look up [`302 Found`][302] and `303 See Other`. Why is `303` the clearer choice after a `POST`?
- Use the developer tools to change the new shelf form's `method` to `GET`, then create a shelf. Where do you end up, and what is in the address bar?

<details>
<summary>A form sent with GET</summary>

The browser puts the title in the query string and visits `/shelves?title=poetry`.
That's a `GET` to `/shelves`, so `shelvesController` shows the list of shelves, and the title is ignored.
No shelf is created, because only the `POST` route creates shelves.
This is why the method matters: a link, a bookmark or a refresh only ever sends a `GET`, so none of them can create a shelf by accident.

</details>

> If you tried any changes, undo them, and run `deno task db:reset`, so that your code and data match the starting point of the next exercise.

[search form]: user-input-and-security/search/#form
[shelves]: http://localhost:8000/shelves
[shelf 4]: http://localhost:8000/shelves/4
[aside]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/aside
[label]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/label
[input]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/input
[form method]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/form#method
[media range]: https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Media_queries/Using#syntax_improvements_in_level_4
[form data]: https://developer.mozilla.org/en-US/docs/Web/API/Request/formData
[form data object]: https://developer.mozilla.org/en-US/docs/Web/API/FormData
[form data get]: https://developer.mozilla.org/en-US/docs/Web/API/FormData/get
[async]: router-class/middleware/#async
[303]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/303
[302]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/302
[location]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Location
[prg]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Redirections#temporary_redirections
[html]: user-input-and-security/escaping/#html
[rest parameters]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/rest_parameters
[chain]: router-class/middleware/#chain

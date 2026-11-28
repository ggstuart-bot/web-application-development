---
type: exercise
title: Route middleware, then delete a shelf
weight: 2
author: Dr Graeme Stuart
---

In this exercise we will let a route run more than one handler.
We will move the shelf and book lookups out of our controllers and into middleware, and then use them to add a button that deletes a shelf.

<!--more-->

> You should have the add and remove buttons, and `controllers/shelfBooks.js`, from the [previous exercise][previous exercise].
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

Four of our controllers start the same way: load a shelf or a book from the address, and throw a `NotFoundError` if it isn't there.
Our router already runs a chain of [middleware][chain] for every request.
In this exercise we will give each route a chain of its own.
By the end of this exercise you should be able to:

- Register a route with a chain of handlers, ending with its controller
- Write *route middleware* that loads the thing a route is about, and puts it on `ctx.data`
- Explain why a controller can rely on what its middleware loaded
- Delete a row, and use `RETURNING` to get back what was deleted
- Explain what `ON DELETE CASCADE` does to the rows that point at a deleted row

## A chain of handlers {#handler-chain}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

At the moment, each route has a single handler, its controller.
Update **server/tools/router.js** so that a route can have as many as we like.

**server/tools/router.js**
```JavaScript {hl_lines="9 11 40"}
import { NotFoundError } from "./errors.js";

export class Router {
    constructor() {
        this.routes = [];
        this.middleware = [];
    }

    register(method, pathname, ...handlers) {
        const pattern = new URLPattern({ pathname });
        this.routes.push({ method, pattern, handlers });
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
        return this.chain(ctx, route.handlers);
    }

    chain(ctx, handlers) {
        const [fn, ...remaining] = handlers;
        if (!remaining.length) return fn(ctx);
        const next = (ctx) => this.chain(ctx, remaining);
        return fn(ctx, next);
    }
}
```

- `register` takes any number of handlers after the pathname. `...handlers` is a rest parameter, just like the `...args` in our [get and post shortcuts][get post], so `handlers` is an array.
- The route keeps the whole list.
- When `dispatch` finds the route, it runs the list with `chain`, the same method that runs our middleware.

`chain` calls the first handler with `ctx` and a `next` function.
Each handler can do its job, then call `next(ctx)` to pass `ctx` on to the one after it.
The last handler in the list is the controller, and it's called with just `ctx`.

Every route still has one handler, so nothing changes in the browser.
Check that [http://localhost:8000/shelves/1][shelf 1] and [http://localhost:8000/books/1][book 1] still work.

## withShelf {#with-shelf}

`shelfController`, `addBookController` and `removeBookController` all start by loading the shelf.
Create a new file, **server/middleware/resources.js**.

**server/middleware/resources.js**
```JavaScript {hl_lines="1-12"}
import { getShelfById } from "../models/shelves.js";
import { NotFoundError } from "../tools/errors.js";

export function withShelf(ctx, next) {
    const { params: { shelfId } } = ctx;
    const shelf = getShelfById(Number(shelfId));
    if (!shelf) {
        throw new NotFoundError(`Shelf ${shelfId} not found`);
    }
    ctx.data.shelf = shelf;
    return next(ctx);
}
```

`withShelf` is the lookup from our controllers, turned into a middleware.

- It loads the shelf from `:shelfId`, and throws a `NotFoundError` if it isn't there.
- Otherwise it puts the shelf on `ctx.data.shelf` and calls `next(ctx)`.

A handler after `withShelf` only ever runs if the shelf exists, so it can use `ctx.data.shelf` straight away.

Update **server/main.js** to put `withShelf` before the three controllers that need it.

**server/main.js**
```JavaScript {hl_lines="13 26-28"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import { addBookController, removeBookController } from "./controllers/shelfBooks.js";
import {
    createShelfController,
    shelfController,
    shelvesController,
} from "./controllers/shelves.js";
import { staticController } from "./controllers/static.js";
import { Router } from "./tools/router.js";
import { withLogs } from "./middleware/logging.js";
import { withErrors } from "./middleware/errors.js";
import { withShelf } from "./middleware/resources.js";

const app = new Router();

app.use(withLogs);
app.use(withErrors);

app.get("/assets/*", staticController);
app.get("/", homeController);
app.get("/books", booksController);
app.get("/books/:bookId", bookController);
app.get("/shelves", shelvesController);
app.post("/shelves", createShelfController);
app.get("/shelves/:shelfId", withShelf, shelfController);
app.post("/shelves/:shelfId/books/:bookId/add", withShelf, addBookController);
app.post("/shelves/:shelfId/books/:bookId/delete", withShelf, removeBookController);

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

The route reads from left to right: load the shelf, then run the controller.

Now remove the lookups from the controllers.
Update `shelfController` in **server/controllers/shelves.js**.

**server/controllers/shelves.js**
```JavaScript {hl_lines="1 23"}
import { createShelf, getShelves } from "../models/shelves.js";
import { addThumbnails, getBooksByShelfId } from "../models/shelfBooks.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
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
    ctx.flash = `Shelf '${shelf.title}' created`;
    return redirect(`/shelves/${shelf.id}`, ctx);
}

export function shelfController(ctx) {
    const { data: { shelf } } = ctx;
    ctx.data.title = shelf.title;
    ctx.data.books = getBooksByShelfId(shelf.id);
    return render(shelfView, ctx);
}
```

`shelfController` takes the shelf from `ctx.data`, where `withShelf` put it.
It no longer needs `getShelfById` or `NotFoundError`, so their imports go.

Update **server/controllers/shelfBooks.js** in the same way.

**server/controllers/shelfBooks.js**
```JavaScript {hl_lines="7 20"}
import { getBookById } from "../models/books.js";
import { addBookToShelf, removeBookFromShelf } from "../models/shelfBooks.js";
import { NotFoundError } from "../tools/errors.js";
import { redirect } from "../tools/redirect.js";

export function addBookController(ctx) {
    const { params: { bookId }, data: { shelf } } = ctx;
    const book = getBookById(Number(bookId));
    if (!book) {
        throw new NotFoundError(`Book ${bookId} not found`);
    }
    const changes = addBookToShelf(shelf.id, book.id);
    ctx.flash = changes
        ? `Book '${book.title}' added to '${shelf.title}'`
        : `Book '${book.title}' is already on '${shelf.title}'`;
    return redirect(`/books/${book.id}`, ctx);
}

export function removeBookController(ctx) {
    const { params: { bookId }, data: { shelf } } = ctx;
    const book = getBookById(Number(bookId));
    if (!book) {
        throw new NotFoundError(`Book ${bookId} not found`);
    }
    removeBookFromShelf(shelf.id, book.id);
    ctx.flash = `Book '${book.title}' removed`;
    return redirect(`/shelves/${shelf.id}`, ctx);
}
```

Both controllers now take `bookId` from `ctx.params` and `shelf` from `ctx.data`, in one destructuring assignment.

Open [http://localhost:8000/shelves/1][shelf 1], add a book to a shelf, and remove one.
Everything works just as before.
[http://localhost:8000/shelves/999][shelf 999] still gets the not found page, but now it comes from `withShelf`.

```plaintext {linenos=false}

============================
GET /shelves/999
[error] Shelf 999 not found
RESPONSE 404
============================
```

## withBook {#with-book}

The book lookups can go the same way.
Add `withBook` to **server/middleware/resources.js**.

**server/middleware/resources.js**
```JavaScript {hl_lines="2 14-23"}
import { getShelfById } from "../models/shelves.js";
import { getBookById } from "../models/books.js";
import { NotFoundError } from "../tools/errors.js";

export function withShelf(ctx, next) {
    const { params: { shelfId } } = ctx;
    const shelf = getShelfById(Number(shelfId));
    if (!shelf) {
        throw new NotFoundError(`Shelf ${shelfId} not found`);
    }
    ctx.data.shelf = shelf;
    return next(ctx);
}

export function withBook(ctx, next) {
    const { params: { bookId } } = ctx;
    const book = getBookById(Number(bookId));
    if (!book) {
        throw new NotFoundError(`Book ${bookId} not found`);
    }
    ctx.data.book = book;
    return next(ctx);
}
```

`withBook` does for `:bookId` what `withShelf` does for `:shelfId`, and puts the book on `ctx.data.book`.

Update **server/main.js**.

**server/main.js**
```JavaScript {hl_lines="13 23 27-28"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import { addBookController, removeBookController } from "./controllers/shelfBooks.js";
import {
    createShelfController,
    shelfController,
    shelvesController,
} from "./controllers/shelves.js";
import { staticController } from "./controllers/static.js";
import { Router } from "./tools/router.js";
import { withLogs } from "./middleware/logging.js";
import { withErrors } from "./middleware/errors.js";
import { withBook, withShelf } from "./middleware/resources.js";

const app = new Router();

app.use(withLogs);
app.use(withErrors);

app.get("/assets/*", staticController);
app.get("/", homeController);
app.get("/books", booksController);
app.get("/books/:bookId", withBook, bookController);
app.get("/shelves", shelvesController);
app.post("/shelves", createShelfController);
app.get("/shelves/:shelfId", withShelf, shelfController);
app.post("/shelves/:shelfId/books/:bookId/add", withShelf, withBook, addBookController);
app.post("/shelves/:shelfId/books/:bookId/delete", withShelf, withBook, removeBookController);

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

- The book page gets `withBook` before `bookController`.
- Adding and removing get both: `withShelf`, then `withBook`, then the controller.

Update `bookController` in **server/controllers/books.js**.

**server/controllers/books.js**
```JavaScript {hl_lines="1 29"}
import { countPages, searchBooks } from "../models/books.js";
import { getShelves } from "../models/shelves.js";
import { render } from "../tools/render.js";
import { NotFoundError } from "../tools/errors.js";
import { booksView } from "../views/books.js";
import { bookView } from "../views/book.js";

const PAGE_SIZE = 16;

export function booksController(ctx) {
    const { url } = ctx;
    const q = url.searchParams.get("q") ?? "";
    const pageParam = url.searchParams.get("page");
    const page = pageParam === null ? 1 : Number(pageParam);
    const pageCount = Math.max(1, countPages(q, PAGE_SIZE));
    if (!Number.isSafeInteger(page) || page < 1 || page > pageCount) {
        throw new NotFoundError(`Page ${pageParam} not found`);
    }
    ctx.data.q = q;
    ctx.data.page = page;
    ctx.data.books = searchBooks(q, page, PAGE_SIZE);
    ctx.data.pageCount = pageCount;
    ctx.data.title = q ? `Books matching "${q}"` : "Books";
    return render(booksView, ctx);
}

export function bookController(ctx) {
    ctx.data.shelves = getShelves();
    ctx.data.title = ctx.data.book.title;
    return render(bookView, ctx);
}
```

`bookController` only has to load the shelves and set the title.
`getBookById` has gone from its imports, but `NotFoundError` stays, because `booksController` still throws one for a page that doesn't exist.

Finally, update **server/controllers/shelfBooks.js**.

**server/controllers/shelfBooks.js**
```JavaScript {hl_lines="5 14"}
import { addBookToShelf, removeBookFromShelf } from "../models/shelfBooks.js";
import { redirect } from "../tools/redirect.js";

export function addBookController(ctx) {
    const { data: { shelf, book } } = ctx;
    const changes = addBookToShelf(shelf.id, book.id);
    ctx.flash = changes
        ? `Book '${book.title}' added to '${shelf.title}'`
        : `Book '${book.title}' is already on '${shelf.title}'`;
    return redirect(`/books/${book.id}`, ctx);
}

export function removeBookController(ctx) {
    const { data: { shelf, book } } = ctx;
    removeBookFromShelf(shelf.id, book.id);
    ctx.flash = `Book '${book.title}' removed`;
    return redirect(`/shelves/${shelf.id}`, ctx);
}
```

By the time these controllers run, both the shelf and the book are known to exist.
All that's left in each one is the change itself, the flash message and the redirect.

Check that the book page, adding and removing all still work.
Then try [http://localhost:8000/books/99999][book 99999], which gets the not found page from `withBook`.

## A delete button {#delete-button}

Now we can add a new feature with very little code.
A shelf should have a button that deletes it.

Update **server/views/shelf.js** to add an aside, like the one on the book page.

**server/views/shelf.js**
```JavaScript {hl_lines="20-24"}
import { html } from "../tools/html.js";
import { bookCard } from "./partials/bookCard.js";
import { cardGrid } from "./partials/cardGrid.js";
import { postButton } from "./partials/postButton.js";

export function shelfView(ctx) {
    const { data: { shelf, books } } = ctx;

    return html`
        <section>
            <header>
                <h1>${shelf.title}</h1>
            </header>
            ${cardGrid(
                books.map((book) => bookCard(book, removeForm(shelf, book))),
                html`No books yet, <a href="/books">find some books</a> to add`,
            )}
            <p><a href="/shelves">Back to all shelves</a></p>
        </section>
        <aside>
            <h2>Delete shelf</h2>
            <p>The books stay in the catalogue</p>
            ${postButton(`/shelves/${shelf.id}/delete`, "delete shelf")}
        </aside>
    `;
}

function removeForm(shelf, book) {
    return postButton(`/shelves/${shelf.id}/books/${book.id}/delete`, "remove");
}
```

- The button is our third `postButton`, and it posts to `/shelves/:shelfId/delete`.
- The note above it says that the books stay in the catalogue. Deleting a shelf only deletes the shelf and its links to books, never the books themselves.

Open [http://localhost:8000/shelves/3][shelf 3].
The cards are **The Kite Runner**, **The Book Thief** and **Life of Pi**, with the **Delete shelf** aside beside them.

{{<figure src="images/delete-button.webp" alt="The to read shelf, with three book cards on the left: The Kite Runner, The Book Thief and Life of Pi, and a light Delete shelf aside on the right, separated by a vertical line, with the note The books stay in the catalogue above an outlined delete shelf button" caption="The delete button sits in an aside beside the books.">}}

Press **delete shelf**.
There's no route yet, so we get the **Not found** page.

## A stub behind withShelf {#delete-stub}

As before, we start with a stub.
Add `deleteShelfController` to the end of **server/controllers/shelves.js**.

**server/controllers/shelves.js**
```JavaScript {hl_lines="28-33"}
import { createShelf, getShelves } from "../models/shelves.js";
import { addThumbnails, getBooksByShelfId } from "../models/shelfBooks.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
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
    ctx.flash = `Shelf '${shelf.title}' created`;
    return redirect(`/shelves/${shelf.id}`, ctx);
}

export function shelfController(ctx) {
    const { data: { shelf } } = ctx;
    ctx.data.title = shelf.title;
    ctx.data.books = getBooksByShelfId(shelf.id);
    return render(shelfView, ctx);
}

export function deleteShelfController(ctx) {
    const { data: { shelf } } = ctx;
    ctx.flash = `Ignored request to delete shelf '${shelf.title}'`;
    return redirect(`/shelves/${shelf.id}`, ctx);
}
```

The stub takes the shelf from `ctx.data`, flashes a message with its title, and redirects back to the shelf.

Add the route to **server/main.js**, with `withShelf` first.

**server/main.js**
```JavaScript {hl_lines="6 28"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import { addBookController, removeBookController } from "./controllers/shelfBooks.js";
import {
    createShelfController,
    deleteShelfController,
    shelfController,
    shelvesController,
} from "./controllers/shelves.js";
import { staticController } from "./controllers/static.js";
import { Router } from "./tools/router.js";
import { withLogs } from "./middleware/logging.js";
import { withErrors } from "./middleware/errors.js";
import { withBook, withShelf } from "./middleware/resources.js";

const app = new Router();

app.use(withLogs);
app.use(withErrors);

app.get("/assets/*", staticController);
app.get("/", homeController);
app.get("/books", booksController);
app.get("/books/:bookId", withBook, bookController);
app.get("/shelves", shelvesController);
app.post("/shelves", createShelfController);
app.get("/shelves/:shelfId", withShelf, shelfController);
app.post("/shelves/:shelfId/delete", withShelf, deleteShelfController);
app.post("/shelves/:shelfId/books/:bookId/add", withShelf, withBook, addBookController);
app.post("/shelves/:shelfId/books/:bookId/delete", withShelf, withBook, removeBookController);

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

Press **delete shelf** on [http://localhost:8000/shelves/3][shelf 3].
You come back to the shelf, with the message **Ignored request to delete shelf 'to read'**.

Our first stubs could only name the ids they were sent.
This one can name the shelf, because `withShelf` has already loaded it.

### Changing the address {#delete-stub-address}

Try the developer tools trick from the [previous exercise][stub address] on this button.

1. On [http://localhost:8000/shelves/3][shelf 3], open the developer tools (**F12**) on the **elements** panel.
2. Find the delete button's form: `<form method="POST" action="/shelves/3/delete">`.
3. Change the `action` to `/shelves/999/delete`, and press the button.

The add stub believed any id we sent it, but this time we get the **Not found** page.

```plaintext {linenos=false}

============================
POST /shelves/999/delete
[error] Shelf 999 not found
RESPONSE 404
============================
```

The stub never ran.
`withShelf` comes first, and it threw a `NotFoundError` before the controller had a chance.
Every controller behind `withShelf` gets that check for free.

> Refresh the page to get the real button back.

## Deleting the shelf {#delete-shelf}

Deleting needs a new function in our model.
Add `deleteShelfById` to the end of **server/models/shelves.js**.

**server/models/shelves.js**
```JavaScript {linenostart="25", hl_lines="1-5"}

// Placeholder: this will need to change when shelves have owners
export function deleteShelfById(id) {
    return db.prepare(`DELETE FROM shelves WHERE id=:id RETURNING id, title`).get({ id });
}
```

- `DELETE FROM shelves WHERE id=:id` deletes the shelf's row.
- [`RETURNING`][returning] hands back the row we deleted, just as `createShelf` hands back the row it made. We use `get`, because there's at most one.

> Like the other functions in this file, it has a placeholder.
> Deleting a shelf will need to change when shelves have owners.

Now update `deleteShelfController` in **server/controllers/shelves.js**.

**server/controllers/shelves.js**
```JavaScript {hl_lines="1 30-33"}
import { createShelf, deleteShelfById, getShelves } from "../models/shelves.js";
import { addThumbnails, getBooksByShelfId } from "../models/shelfBooks.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
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
    ctx.flash = `Shelf '${shelf.title}' created`;
    return redirect(`/shelves/${shelf.id}`, ctx);
}

export function shelfController(ctx) {
    const { data: { shelf } } = ctx;
    ctx.data.title = shelf.title;
    ctx.data.books = getBooksByShelfId(shelf.id);
    return render(shelfView, ctx);
}

export function deleteShelfController(ctx) {
    const { data: { shelf: { id } } } = ctx;
    const shelf = deleteShelfById(id);
    ctx.flash = `Shelf '${shelf.title}' deleted`;
    return redirect("/shelves", ctx);
}
```

- `{ data: { shelf: { id } } }` reaches two levels into `ctx`, and takes just the shelf's `id`.
- `deleteShelfById` gives us back the deleted shelf, so the flash message can still name it.
- The redirect goes to the shelves page, because the shelf page we came from no longer exists.

What happens to the shelf's books?
The `shelfBooks` table's `shelfId` column was created with `ON DELETE CASCADE`.
When a shelf is deleted, SQLite deletes every row in `shelfBooks` that points at it too.
The books themselves are in the `books` table, and nothing there points at shelves, so they stay.

Press **delete shelf** on [http://localhost:8000/shelves/3][shelf 3].
You land on [http://localhost:8000/shelves][shelves] with the message **Shelf 'to read' deleted**, and the **to read** card has gone.

```plaintext {linenos=false}

============================
POST /shelves/3/delete
[flash sent] Shelf 'to read' deleted
RESPONSE 303
============================

============================
GET /shelves
[flash received] Shelf 'to read' deleted
RESPONSE 200
============================
```

Visit [http://localhost:8000/shelves/3][shelf 3] again, and you get the not found page.
Search for [The Book Thief][book thief], which was on the **to read** shelf.
It's still in the catalogue.

## Conclusion

Our routes can now run a chain of handlers, and two route middleware, `withShelf` and `withBook`, load the shelf and book from the address before the controller runs.
A controller behind them can rely on what they loaded, so our controllers only do their own job.
A new feature behind `withShelf`, like deleting a shelf, gets the not found check for free, even as a stub.
`deleteShelfById` uses `RETURNING` so we can still name the deleted shelf, and `ON DELETE CASCADE` cleans up its links to books.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="24"}
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
│   │   ├── shelfBooks.js
│   │   ├── shelves.js
│   │   └── static.js
│   ├── middleware/
│   │   ├── errors.js
│   │   ├── logging.js
│   │   └── resources.js
│   ├── models/
│   │   ├── books.js
│   │   ├── db.js
│   │   ├── shelfBooks.js
│   │   └── shelves.js
│   ├── tools/
│   │   ├── errors.js
│   │   ├── flash.js
│   │   ├── html.js
│   │   ├── redirect.js
│   │   ├── render.js
│   │   └── router.js
│   ├── views/
│   │   ├── partials/
│   │   │   ├── bookCard.js
│   │   │   ├── cardGrid.js
│   │   │   ├── postButton.js
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

## Challenge: Order in the chain

These are questions to investigate, not changes to keep.

- Swap `withShelf` and `withBook` on the add route. Send an add request for shelf 999 and book 99999, using the developer tools. Which message does the terminal show before and after the swap, and why?
- Put `withShelf` *after* `deleteShelfController` in its route. What happens when you press **delete shelf**? Which handler does `chain` run first?
- Delete every shelf, then open a book's page. What does the **Add to a shelf** panel say?
- The `shelfBooks` table has `ON DELETE CASCADE` on `bookId` as well. What would happen to the shelves if we deleted a book?

<details>
<summary>A middleware in the wrong place</summary>

`chain` runs the handlers in the order we list them, so `deleteShelfController` runs first.
It is given a `next` function, but it never calls it, so `withShelf` never runs.
Nothing has put a shelf on `ctx.data`, so reaching for `ctx.data.shelf.id` throws a `TypeError`, and `withErrors` turns it into the **Server error** page.
A middleware always has to come before the handlers that rely on it.

</details>

> If you tried any changes, undo them, and run `deno task db:reset`, so that your code and data match the starting point of the next unit.

[previous exercise]: resource-middleware/adding-and-removing-books/
[stub address]: resource-middleware/adding-and-removing-books/#stub-address
[chain]: router-class/middleware/#chain
[get post]: post-redirect-flash/form-that-posts/#get-post
[returning]: https://www.sqlite.org/lang_returning.html
[shelf 1]: http://localhost:8000/shelves/1
[shelf 3]: http://localhost:8000/shelves/3
[shelf 999]: http://localhost:8000/shelves/999
[shelves]: http://localhost:8000/shelves
[book 1]: http://localhost:8000/books/1
[book 99999]: http://localhost:8000/books/99999
[book thief]: http://localhost:8000/books?q=book+thief

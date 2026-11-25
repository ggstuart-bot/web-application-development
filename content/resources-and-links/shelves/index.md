---
type: exercise
title: Shelves
weight: 2
author: Dr Graeme Stuart
---

In this exercise we will add shelves to our application.
We will give them a table in the database, seed three of them, list them on a shelves page, and give each one a page of its own.

<!--more-->

> You should have a page for every book from the [previous exercise][previous exercise].
> If you have been experimenting, run `deno task db:reset` to get back to the seed data.
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

Our books have pages of their own now.
But a book app isn't much use until we can collect books together, so in this exercise we will add *shelves*.
For now a shelf is just a title.
In the next exercise we will put books on them.
By the end of this exercise you should be able to:

- Add a second table to our database
- Seed data of our own from a JSON file
- Build a list page and a page for each item, using the same steps as the book page
- Decide when something drawn many times deserves a function of its own, and where that function should live
- Extract a partial in two steps, so that nothing changes until it is shared

## A shelves link {#shelves-link}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Shelves will live at **/shelves**, so they need a link in the site header.
Update **server/views/layout.js**.

**server/views/layout.js**
```JavaScript{hl_lines="23"}
import { html } from "../tools/html.js";

// Placeholder: this will need to change when we add flash messages and sign-in
export function layout(content, ctx) {
    const { data, url } = ctx;
    const title = data.title ? `${data.title} | Bookshelves` : "Bookshelves";

    return html`
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
                        ${navLink("/shelves", "shelves", url.pathname)}
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
}

// A header link, marked as the current page when it matches the path exactly
function navLink(href, content, path) {
    return html`<a href="${href}" aria-current="${
        href === path ? "page" : "false"
    }">${content}</a>`;
}
```

[`navLink`][nav link] already marks the link to the current page, so the new link is just one line.

> The layout's placeholder hasn't changed.
> It is still waiting for flash messages and sign-in.

Refresh any page, and the header has a **shelves** link.
Click it, and we get our not found page, because there is no shelves page yet.

## A stub shelves page {#shelves-stub}

Just like the book page, the shelves page starts as a stub.

Create **server/views/shelves.js**.

**server/views/shelves.js**
```JavaScript{hl_lines="1-11"}
import { html } from "../tools/html.js";

export function shelvesView() {
    return html`
        <section>
            <header>
                <h1>Shelves</h1>
            </header>
        </section>
    `;
}
```

`shelvesView` takes no `ctx`, because it doesn't need anything from it yet.

Create **server/controllers/shelves.js**, a new controller module for everything to do with shelves.

**server/controllers/shelves.js**
```JavaScript{hl_lines="1-7"}
import { render } from "../tools/render.js";
import { shelvesView } from "../views/shelves.js";

export function shelvesController(ctx) {
    ctx.data.title = "Shelves";
    return render(shelvesView, ctx);
}
```

Register the route in **server/main.js**.

**server/main.js**
```JavaScript{hl_lines="3 18"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import { shelvesController } from "./controllers/shelves.js";
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
app.register("/books/:bookId", bookController);
app.register("/shelves", shelvesController);

export function server(request) {
    const url = new URL(request.url);
    const ctx = { request, url, data: {} };
    return app.handle(ctx);
}
```

Click the **shelves** link again.
This time we get a page of our own, with the heading **Shelves**, and the **shelves** link is filled in because it is the current page.

## A shelves table {#shelves-table}

Shelves need somewhere to live in the database.
Update **tasks/db-create.js** to create a `shelves` table.

**tasks/db-create.js**
```JavaScript{hl_lines="3 6-13"}
import { db } from "../server/models/db.js";

db.exec("DROP TABLE IF EXISTS shelves");
db.exec("DROP TABLE IF EXISTS books");

// Placeholder: this will need to change when shelves have owners
db.exec(`
    CREATE TABLE shelves (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT NOT NULL
    )
`);

db.exec(`
    CREATE TABLE books (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT NOT NULL,
        author TEXT NOT NULL,
        year INTEGER,
        thumbnailUrl TEXT,
        coverUrl TEXT
    )
`);
```

Each shelf has an `id`, chosen by the database just like a book's, and a `title`, which it must have.
The table is dropped along with `books`, so `db:create` starts both tables empty.

> The placeholder tells us that this table will need to change when shelves have owners.
> We don't have users yet.

Reset the database.

```bash {linenos=false}
deno task db:reset
```

The `shelves` table is there, but it is empty.

## Seeding the shelves {#seed-shelves}

Our books come from **db/books.json**, which is a catalogue of real books.
Shelves are different: they are data of our own, to start the application with.
They will go in a file of their own.

Create **db/seed.json**.

**db/seed.json**
```json{hl_lines="1-7"}
{
    "shelves": [
        { "title": "classics" },
        { "title": "fantasy" },
        { "title": "to read" }
    ]
}
```

There are three shelves, each with just a title for now.

We need a model function that creates a shelf.
Create **server/models/shelves.js**.

**server/models/shelves.js**
```JavaScript{hl_lines="1-10"}
import { db } from "./db.js";

// Placeholder: this will need to change when shelves have owners
export function createShelf({ title }) {
    return db.prepare(`
        INSERT INTO shelves(title)
        VALUES (:title)
        RETURNING id, title
    `).get({ title });
}
```

`createShelf` is just like [`createBook`][create book], with a single named parameter.
`RETURNING` gives us back the new shelf, with the `id` the database chose.

> The placeholder says that `createShelf` will need to change when shelves have owners.

Update **tasks/db-seed.js** to create the shelves after the books.

**tasks/db-seed.js**
```JavaScript{hl_lines="1 3 9-13"}
import seed from "../db/seed.json" with { type: "json" };
import books from "../db/books.json" with { type: "json" };
import { createShelf } from "../server/models/shelves.js";
import { createBook } from "../server/models/books.js";

for (const book of books) {
    createBook(book);
}

// Temporary: these shelves will belong to users once there are users
for (const shelf of seed.shelves) {
    createShelf({ title: shelf.title });
}
```

We import **db/seed.json** in the same way as **db/books.json**, and call `createShelf` for each shelf in its `shelves` list.

> The `Temporary:` comment reminds us that these shelves belong to nobody.
> Once there are users, each shelf will belong to one of them, and the seed data will change.

Reset the database again.

```bash {linenos=false}
deno task db:reset
```

Nothing shows the shelves yet, but the `shelves` table now holds three rows.

## Listing the shelves {#list-shelves}

Add `getShelves` to **server/models/shelves.js**.

**server/models/shelves.js**
```JavaScript{hl_lines="11-15"}
import { db } from "./db.js";

// Placeholder: this will need to change when shelves have owners
export function createShelf({ title }) {
    return db.prepare(`
        INSERT INTO shelves(title)
        VALUES (:title)
        RETURNING id, title
    `).get({ title });
}

// Placeholder: this will need to change when shelves have owners
export function getShelves() {
    return db.prepare(`SELECT id, title FROM shelves`).all();
}
```

It finds every shelf, just as `getBooks` once found every book.

> This placeholder says the same: once shelves have owners, we won't simply want every shelf.

Update **server/controllers/shelves.js** to load the shelves for the view.

**server/controllers/shelves.js**
```JavaScript{hl_lines="1 6"}
import { getShelves } from "../models/shelves.js";
import { render } from "../tools/render.js";
import { shelvesView } from "../views/shelves.js";

export function shelvesController(ctx) {
    ctx.data.shelves = getShelves();
    ctx.data.title = "Shelves";
    return render(shelvesView, ctx);
}
```

Update **server/views/shelves.js** to show them.

**server/views/shelves.js**
```JavaScript{hl_lines="3-5 11-12 16-26"}
import { html } from "../tools/html.js";

export function shelvesView(ctx) {
    const { data: { shelves } } = ctx;

    return html`
        <section>
            <header>
                <h1>Shelves</h1>
            </header>
            <h2>All shelves</h2>
            <ul class="grid">${shelves.map(shelfCard)}</ul>
        </section>
    `;
}

// Shelf lists always sit under an <h2>, so the title is an <h3>
function shelfCard(shelf) {
    return html`
        <li class="card shelf">
            <div>
                <h3>${shelf.title}</h3>
            </div>
        </li>
    `;
}
```

- The view needs `ctx` now, so it takes it, and destructures `shelves` from `ctx.data`.
- The shelves go in a `<ul class="grid">`, as cards, just like the books.
- They sit under an **All shelves** heading, so each card's title is an `<h3>`. The comment above `shelfCard` says so.

Every shelf is drawn as a card, so drawing a shelf gets a name of its own: `shelfCard`.
Only this page uses it, so it is a function in **shelves.js**, not a partial.
It isn't exported, because nothing else needs it.

Visit [http://localhost:8000/shelves][shelves].
There are three cards: **classics**, **fantasy** and **to read**.

## Links to shelf pages {#shelf-links}

Each shelf will have its own page, just like each book.
Update **server/views/shelves.js** so that each card's title links to its shelf's page.

**server/views/shelves.js**
```JavaScript{hl_lines="22"}
import { html } from "../tools/html.js";

export function shelvesView(ctx) {
    const { data: { shelves } } = ctx;

    return html`
        <section>
            <header>
                <h1>Shelves</h1>
            </header>
            <h2>All shelves</h2>
            <ul class="grid">${shelves.map(shelfCard)}</ul>
        </section>
    `;
}

// Shelf lists always sit under an <h2>, so the title is an <h3>
function shelfCard(shelf) {
    return html`
        <li class="card shelf">
            <div>
                <h3><a href="/shelves/${shelf.id}">${shelf.title}</a></h3>
            </div>
        </li>
    `;
}
```

Click on **classics**.
The address bar shows [http://localhost:8000/shelves/1][shelf 1], and we get the not found page again, until the route exists.

## A stub shelf page {#shelf-stub}

The shelf page starts as a stub too, just like the [first version of the book page][book stub].

Create **server/views/shelf.js**.

**server/views/shelf.js**
```JavaScript{hl_lines="1-13"}
import { html } from "../tools/html.js";

export function shelfView(ctx) {
    const { params: { shelfId } } = ctx;

    return html`
        <section>
            <header>
                <h1>Shelf ${shelfId}</h1>
            </header>
        </section>
    `;
}
```

Add a `shelfController` to **server/controllers/shelves.js**.

**server/controllers/shelves.js**
```JavaScript{hl_lines="4 11-14"}
import { getShelves } from "../models/shelves.js";
import { render } from "../tools/render.js";
import { shelvesView } from "../views/shelves.js";
import { shelfView } from "../views/shelf.js";

export function shelvesController(ctx) {
    ctx.data.shelves = getShelves();
    ctx.data.title = "Shelves";
    return render(shelvesView, ctx);
}

export function shelfController(ctx) {
    return render(shelfView, ctx);
}
```

Register the route in **server/main.js**.

**server/main.js**
```JavaScript{hl_lines="3 19"}
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

app.register("/assets/*", staticController);
app.register("/", homeController);
app.register("/books", booksController);
app.register("/books/:bookId", bookController);
app.register("/shelves", shelvesController);
app.register("/shelves/:shelfId", shelfController);

export function server(request) {
    const url = new URL(request.url);
    const ctx = { request, url, data: {} };
    return app.handle(ctx);
}
```

The route uses a [route parameter][route params], `:shelfId`, so our router puts it on `ctx.params`.

Click on **classics** again.
The page says **Shelf 1**.

## Loading the shelf {#get-shelf-by-id}

Add `getShelfById` to **server/models/shelves.js**.

**server/models/shelves.js**
```JavaScript{hl_lines="12-16"}
import { db } from "./db.js";

// Placeholder: this will need to change when shelves have owners
export function createShelf({ title }) {
    return db.prepare(`
        INSERT INTO shelves(title)
        VALUES (:title)
        RETURNING id, title
    `).get({ title });
}

// Placeholder: this will need to change when shelves have owners
export function getShelfById(id) {
    return db.prepare(`SELECT id, title FROM shelves WHERE id=:id`).get({ id });
}

// Placeholder: this will need to change when shelves have owners
export function getShelves() {
    return db.prepare(`SELECT id, title FROM shelves`).all();
}
```

It works just like [`getBookById`][get book by id], returning the shelf or `undefined`.

> This placeholder says that `getShelfById` will need to change when shelves have owners, too.

Update **server/controllers/shelves.js** to load the shelf.

**server/controllers/shelves.js**
```JavaScript{hl_lines="1 3 14-20"}
import { getShelfById, getShelves } from "../models/shelves.js";
import { render } from "../tools/render.js";
import { NotFoundError } from "../tools/errors.js";
import { shelvesView } from "../views/shelves.js";
import { shelfView } from "../views/shelf.js";

export function shelvesController(ctx) {
    ctx.data.shelves = getShelves();
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
    return render(shelfView, ctx);
}
```

This is the same shape as `bookController`.
We turn the id into a number, throw a `NotFoundError` if there is no such shelf, and put the shelf and its title on `ctx.data`.

Update **server/views/shelf.js** to show the shelf's title, with a link back.

**server/views/shelf.js**
```JavaScript{hl_lines="4 9 11"}
import { html } from "../tools/html.js";

export function shelfView(ctx) {
    const { data: { shelf } } = ctx;

    return html`
        <section>
            <header>
                <h1>${shelf.title}</h1>
            </header>
            <p><a href="/shelves">Back to all shelves</a></p>
        </section>
    `;
}
```

Visit [http://localhost:8000/shelves/1][shelf 1].
The heading is **classics**, with a link back to all shelves.
The shelf has no books yet, so that's all there is.

Visit [http://localhost:8000/shelves/999][shelf 999], and we get the not found page.

## A card grid partial {#card-grid}

The books page shows a grid of cards, or a message when there aren't any.
The shelves page always shows a grid, even if it is empty.
It needs the same choice as the books page, so we will share it.

We will do this in two steps.
First we move the code into a *partial*, and use it where it came from, so we can check that nothing has changed.
Only then do we use it somewhere new.

Create **server/views/partials/cardGrid.js**.

**server/views/partials/cardGrid.js**
```JavaScript{hl_lines="1-5"}
import { html } from "../../tools/html.js";

export function cardGrid(cards, emptyMessage) {
    return cards.length ? html`<ul class="grid">${cards}</ul>` : html`<p>${emptyMessage}</p>`;
}
```

`cardGrid` takes the cards, and the message to show when there are none.
It is the same conditional as the old `results` constant in the books view, with the books taken out.

Update **server/views/books.js** to use it.

**server/views/books.js**
```JavaScript{hl_lines="2 17-20"}
import { html } from "../tools/html.js";
import { cardGrid } from "./partials/cardGrid.js";
import { searchForm } from "./partials/searchForm.js";

export const DEFAULT_COVER = "/assets/default-cover.svg";

export function booksView(ctx) {
    const { data: { title, books, q, page, pageCount } } = ctx;

    return html`
        <section>
            <header>
                <h1>${title}</h1>
                ${q ? html`<p><a href="/books">show all books</a></p>` : ""}
            </header>
            ${searchForm(q)}
            ${cardGrid(
                books.map(bookCard),
                q ? `No books match "${q}"` : "No books found",
            )}
            ${pagination(q, page, pageCount)}
        </section>
    `;
}

function bookCard(book) {
    const cover = book.coverUrl ?? DEFAULT_COVER;
    return html`
        <li class="card book">
            <img class="cover" src="${cover}" alt="" width="200" height="300" loading="lazy">
            <div>
                <h2><a href="/books/${book.id}">${book.title}</a></h2>
                <p>${book.author}${book.year ? ` (${book.year})` : ""}</p>
            </div>
        </li>
    `;
}

// Previous, the current page and next. Nothing at all if there's only one page
function pagination(q, page, pageCount) {
    if (pageCount <= 1) return "";
    return html`<nav aria-label="Pagination">
            ${page > 1 ? html`<a href="${pageUrl(q, page - 1)}" rel="prev">previous</a>` : ""}
            <a href="${pageUrl(q, page)}" aria-current="page">page ${page} of ${pageCount}</a>
            ${page < pageCount ? html`<a href="${pageUrl(q, page + 1)}" rel="next">next</a>` : ""}
        </nav>`;
}

// Only keep q in the link if there's a search
function pageUrl(q, page) {
    const params = q ? { q, page } : { page };
    return `/books?${new URLSearchParams(params)}`;
}
```

The `results` constant has gone.
We pass `cardGrid` the cards, and the right message for a search or for no books at all.

Visit [http://localhost:8000/books][books] and search for [**zebra**][q zebra].
Both look just as they did.

## cardGrid on the shelves page {#card-grid-shelves}

Now we can use the partial somewhere new.
Update **server/views/shelves.js**.

**server/views/shelves.js**
```JavaScript{hl_lines="2 13"}
import { html } from "../tools/html.js";
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
    `;
}

// Shelf lists always sit under an <h2>, so the title is an <h3>
function shelfCard(shelf) {
    return html`
        <li class="card shelf">
            <div>
                <h3><a href="/shelves/${shelf.id}">${shelf.title}</a></h3>
            </div>
        </li>
    `;
}
```

With no shelves, the page now says **No shelves yet**, rather than showing an empty grid.
We have three shelves, so [http://localhost:8000/shelves][shelves] looks just the same.
You can see the message in the challenge.

## Shelf card titles {#h3}

Look closely at [http://localhost:8000/shelves][shelves].
The shelf titles are bolder and larger than the book titles on the books page, and they have extra space around them.
That's because they are `<h3>` elements, and our styles only set up `<h1>` and `<h2>`.

Add `h3` to the headings rule near the top of **public/styles.css**.

**public/styles.css**
```css {linenostart="15", hl_lines="3-4"}
/* Headings are a little heavier than the light body text */
h1,
h2,
h3 {
    font-weight: 400;
}
```

Then add it to the headings inside the `.card` rule.

**public/styles.css**
```css {linenostart="105", hl_lines="19"}
/* A card is a bordered box with rounded corners. Each kind of card sets its own grid */
.card {
    display: grid;
    border: 1px solid;
    border-radius: 0.5rem;
    overflow: clip; /* the cover is cut to the rounded corners */
    overflow-wrap: anywhere; /* a very long word breaks, rather than squashing the cover */

    /* The text is spread from the top of the card to the bottom */
    > div:last-child {
        display: grid;
        align-content: space-between;
        gap: 0.5rem;
        padding: 0.5rem;
    }

    /* Headings and paragraphs in a card have no margins and stay at the text size */
    h2,
    h3,
    p {
        margin: 0;
        font-size: 1em;
    }
}
```

- `h3` gets the same light weight as the other headings.
- Inside a card, an `h3` has no margins and stays at the text size, just like an `h2`.

Refresh [http://localhost:8000/shelves][shelves].
A shelf card's title now sits just like a book card's.

{{<figure src="images/shelves-list.webp" alt="The Shelves page, with an All shelves heading and three plain cards titled classics, fantasy and to read, each a link" caption="The shelves page lists every shelf.">}}

## Conclusion

Our application has shelves now.
They have a table in the database, seeded from **db/seed.json**, a page that lists them all, and a page for each one.
We built both pages in the same steps as the book page: a stub, then the data, then the markup and styles.

`shelfCard` is a function in the shelves view, because only that page draws shelf cards.
`cardGrid` is a partial, because two pages use it.
We extracted it in two steps, checking that the books page didn't change before using it on the shelves page.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="8 18 26 34 41 42"}
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
│   │   └── shelves.js
│   ├── tools/
│   │   ├── errors.js
│   │   ├── html.js
│   │   ├── render.js
│   │   └── router.js
│   ├── views/
│   │   ├── partials/
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

Our shelves are empty, though.
In the next exercise we will put books on them.

## Challenge: Empty shelves

These are questions to investigate, not changes to keep.

- Empty the `shelves` list in **db/seed.json** (leave `"shelves": []`), and reset the database. What does [http://localhost:8000/shelves][shelves] show now?
- With the shelves back, visit [http://localhost:8000/shelves/4][shelf 4]. What do you get, and what does the terminal say?
- Add a shelf to **db/seed.json** with the title `<em>poetry</em>`, and reset the database. How does its card look, and why?
- `shelvesView` didn't take `ctx` at first. What would have gone wrong if the stub had tried to destructure `shelves` from `ctx.data` before the controller loaded them?

<details>
<summary>What to expect</summary>

With no shelves, `cardGrid` gets an empty array, so it shows **No shelves yet**.

There are only three shelves, so `getShelfById(4)` returns `undefined`, and `shelfController` throws a `NotFoundError` with the message **Shelf 4 not found**.

The title is escaped by our `html` helper, so the card shows the text **&lt;em&gt;poetry&lt;/em&gt;**, tags and all, rather than italic text.

`ctx.data.shelves` would have been `undefined`, so `shelves.map` would throw a `TypeError`, and we would get the server error page.

</details>

> If you tried any changes, undo them, and run `deno task db:reset`, so that your code and data match the starting point of the next exercise.

[previous exercise]: resources-and-links/book-pages/
[books]: http://localhost:8000/books
[q zebra]: http://localhost:8000/books?q=zebra
[shelves]: http://localhost:8000/shelves
[shelf 1]: http://localhost:8000/shelves/1
[shelf 4]: http://localhost:8000/shelves/4
[shelf 999]: http://localhost:8000/shelves/999
[nav link]: router-class/one-context/#nav-link
[create book]: sqlite-and-models/database/#models
[book stub]: resources-and-links/book-pages/#book-stub
[route params]: resources-and-links/book-pages/#route-params
[get book by id]: resources-and-links/book-pages/#get-book-by-id

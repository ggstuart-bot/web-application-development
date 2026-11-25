---
type: exercise
title: Books on shelves
weight: 3
author: Dr Graeme Stuart
---

In this exercise we will put books on our shelves.
We will link the two tables together with a third, show each shelf's books on its page, and give each shelf card a book count and a strip of covers.

<!--more-->

> You should have the shelves page and a page for each shelf from the [previous exercise][previous exercise].
> If you have been experimenting, run `deno task db:reset` to get back to the seed data.
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

We have books and we have shelves, but they have nothing to do with each other yet.
A shelf holds many books, and a book can sit on many shelves, so we need a way to connect them.
By the end of this exercise you should be able to:

- Explain a *many-to-many* relationship, and store one in a *join table*
- Use foreign keys, `ON DELETE CASCADE` and a primary key made of two columns
- Join two tables in a query, and count rows with a subquery
- Look up seed data by name, and stop with a clear error when a name is wrong
- Move a function and a constant it needs into a partial, without changing the page that uses it

## Many books, many shelves {#shelf-books-table}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Where should we record which books are on which shelf?

- A book can't hold a `shelfId`, because a book can be on many shelves.
- A shelf can't hold a `bookId`, because a shelf holds many books.

This is a *many-to-many* relationship, and the usual answer is a third table, a *join table*, with one row for each book on each shelf.
If **The Hobbit** is on two shelves, there are two rows, one for each shelf.

Update **tasks/db-create.js** to create a `shelfBooks` table.

**tasks/db-create.js**
```JavaScript{hl_lines="3 25-32"}
import { db } from "../server/models/db.js";

db.exec("DROP TABLE IF EXISTS shelfBooks");
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

db.exec(`
    CREATE TABLE shelfBooks (
        shelfId INTEGER NOT NULL REFERENCES shelves(id) ON DELETE CASCADE,
        bookId INTEGER NOT NULL REFERENCES books(id) ON DELETE CASCADE,
        PRIMARY KEY (shelfId, bookId)
    )
`);
```

- `shelfId` and `bookId` are the only columns. Each row says "this book is on this shelf".
- [`REFERENCES`][foreign keys] makes each column a *foreign key*. The database won't store a `shelfId` that isn't the `id` of a real shelf, or a `bookId` that isn't the `id` of a real book.
- `ON DELETE CASCADE` means that when a shelf or a book is deleted, its rows in `shelfBooks` go too. A shelf's rows never point at a book that has gone.
- `PRIMARY KEY (shelfId, bookId)` makes the *pair* the primary key. No two rows can have the same pair, so a book can't be on the same shelf twice.

`shelfBooks` is dropped first, because it refers to the other two tables.
A table that refers to another should go before the table it refers to.

> The placeholder on the `shelves` table is still there, because shelves still have no owners.

> In plain SQLite, foreign key checks are switched off unless you ask for them.
> The SQLite library we use switches them on for us.

Reset the database.

```bash {linenos=false}
deno task db:reset
```

## Books on the seeded shelves {#seed-shelf-books}

Our seed shelves should start with some books on them.
Update **db/seed.json** so each shelf lists the titles of its books.

**db/seed.json**
```json{hl_lines="3-28"}
{
    "shelves": [
        {
            "title": "classics",
            "books": [
                "Pride and Prejudice",
                "Jane Eyre",
                "The Great Gatsby",
                "To Kill a Mockingbird"
            ]
        },
        {
            "title": "fantasy",
            "books": [
                "The Hobbit",
                "A Game of Thrones",
                "The Name of the Wind",
                "The Way of Kings"
            ]
        },
        {
            "title": "to read",
            "books": [
                "The Book Thief",
                "The Kite Runner",
                "Life of Pi"
            ]
        }
    ]
}
```

We list books by title, because the titles are easy to read and to check.
The seeder will have to turn each title into a `bookId`.

We need a model function that puts a book on a shelf.
Create **server/models/shelfBooks.js**, a model for our new table.

**server/models/shelfBooks.js**
```JavaScript{hl_lines="1-7"}
import { db } from "./db.js";

export function addBookToShelf(shelfId, bookId) {
    return db.prepare(
        `INSERT OR IGNORE INTO shelfBooks(shelfId, bookId) VALUES (:shelfId, :bookId)`,
    ).run({ shelfId, bookId });
}
```

- `INSERT OR IGNORE` adds the row, or does nothing if it would break a rule. Here the rule is the primary key, so putting a book on a shelf it is already on does nothing, rather than throwing an error.
- We don't need anything back, so we call `.run()`, rather than `.get()` or `.all()`. `.run()` is for statements that don't return rows.

Now update **tasks/db-seed.js**.

**tasks/db-seed.js**
```JavaScript{hl_lines="5 7 9-10 15-22"}
import seed from "../db/seed.json" with { type: "json" };
import books from "../db/books.json" with { type: "json" };
import { createShelf } from "../server/models/shelves.js";
import { createBook } from "../server/models/books.js";
import { addBookToShelf } from "../server/models/shelfBooks.js";

const bookIds = {};
for (const book of books) {
    const { id } = createBook(book);
    bookIds[book.title] = id;
}

// Temporary: these shelves will belong to users once there are users
for (const shelf of seed.shelves) {
    const { id: shelfId } = createShelf({ title: shelf.title });
    for (const title of shelf.books) {
        const bookId = bookIds[title];
        if (!bookId) {
            throw new Error(`Book '${title}' not found in books.json`);
        }
        addBookToShelf(shelfId, bookId);
    }
}
```

- `bookIds` is an object that remembers each book's `id` by its title, as we create the books. `createBook` returns the new book, so we take its `id` out with [destructuring][destructuring].
- `{ id: shelfId }` is destructuring with a new name: it takes the shelf's `id` and calls it `shelfId`, so it isn't confused with a book's `id`.
- For each title on the shelf, we look up the book's `id`. If there isn't one, we throw an error that names the title. Otherwise, we put the book on the shelf.

> The `Temporary:` comment is still there.
> These shelves still belong to nobody.

Reset the database.

```bash {linenos=false}
deno task db:reset
```

There are now 11 rows in `shelfBooks`.

The error is there for typing mistakes.
Change **Jane Eyre** to **Jane Air** in **db/seed.json**, and reset the database again.

```plaintext {linenos=false}
error: Uncaught (in promise) Error: Book 'Jane Air' not found in books.json
            throw new Error(`Book '${title}' not found in books.json`);
                  ^
```

Without the check, `bookIds["Jane Air"]` would be `undefined`, and `addBookToShelf` would fail with an error that doesn't say which title was wrong.
The seeding stops part way through, so fix the title and reset the database once more.

## A book card partial {#book-card}

The shelf page is going to show its books as cards, just like the books page.
So `bookCard` needs to move out of **server/views/books.js** into a partial, and `DEFAULT_COVER` goes with it.
We will use the same two steps as [cardGrid][card grid]: first move it and use it where it came from, then use it somewhere new.

Create **server/views/partials/bookCard.js**.

**server/views/partials/bookCard.js**
```JavaScript{hl_lines="1-17"}
import { html } from "../../tools/html.js";

export const DEFAULT_COVER = "/assets/default-cover.svg";

// A book as a card in a .grid list
export function bookCard(book) {
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
```

The function is just as it was, with a comment to say what it is for.
It is exported now, because the books page needs it from here.

Update **server/views/books.js** to import it.

**server/views/books.js**
```JavaScript{hl_lines="2"}
import { html } from "../tools/html.js";
import { bookCard } from "./partials/bookCard.js";
import { cardGrid } from "./partials/cardGrid.js";
import { searchForm } from "./partials/searchForm.js";

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

`DEFAULT_COVER` and `bookCard` have both gone from the books view.

The book page imported `DEFAULT_COVER` from **books.js**, so update **server/views/book.js** to import it from the partial instead.

**server/views/book.js**
```JavaScript{hl_lines="2"}
import { html } from "../tools/html.js";
import { DEFAULT_COVER } from "./partials/bookCard.js";

export function bookView(ctx) {
    const { data: { book } } = ctx;

    return html`
        <section>
            <header>
                <h1>${book.title}</h1>
                <p>by ${book.author}</p>
            </header>
            <div class="row">
                <img class="cover" src="${book.coverUrl ?? DEFAULT_COVER}"
                    alt="Cover of '${book.title}'">
                <div>
                    ${book.year ? html`<p>Published ${book.year}</p>` : ""}
                    <p><a href="/books">Back to all books</a></p>
                </div>
            </div>
        </section>
    `;
}
```

Visit [http://localhost:8000/books][books] and [http://localhost:8000/books/36][book 36].
Both look just as they did.

## A shelf's books {#shelf-books}

Add `getBooksByShelfId` to **server/models/shelfBooks.js**.

**server/models/shelfBooks.js**
```JavaScript{hl_lines="3-10"}
import { db } from "./db.js";

export function getBooksByShelfId(shelfId) {
    return db.prepare(`
        SELECT books.id, title, author, year, thumbnailUrl, coverUrl
        FROM shelfBooks JOIN books ON books.id = shelfBooks.bookId
        WHERE shelfId=:shelfId
    `).all({ shelfId });
}

export function addBookToShelf(shelfId, bookId) {
    return db.prepare(
        `INSERT OR IGNORE INTO shelfBooks(shelfId, bookId) VALUES (:shelfId, :bookId)`,
    ).run({ shelfId, bookId });
}
```

The books we want are in the `books` table, but which books are on the shelf is in `shelfBooks`.
A [`JOIN`][join] puts the two tables together.

- `FROM shelfBooks JOIN books ON books.id = shelfBooks.bookId` pairs each row of `shelfBooks` with the book it refers to.
- `WHERE shelfId=:shelfId` keeps only the rows for our shelf.
- We select the book's columns, so each result is a full book, just like the ones `searchBooks` returns. `books.id` says which table the `id` comes from. Only `books` has an `id` column, but with two tables it is clearer to say.

Update **server/controllers/shelves.js** to load the shelf's books.

**server/controllers/shelves.js**
```JavaScript{hl_lines="2 22"}
import { getShelfById, getShelves } from "../models/shelves.js";
import { getBooksByShelfId } from "../models/shelfBooks.js";
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
    ctx.data.books = getBooksByShelfId(shelf.id);
    return render(shelfView, ctx);
}
```

Update **server/views/shelf.js** to show them.

**server/views/shelf.js**
```JavaScript{hl_lines="2-3 6 13-16"}
import { html } from "../tools/html.js";
import { bookCard } from "./partials/bookCard.js";
import { cardGrid } from "./partials/cardGrid.js";

export function shelfView(ctx) {
    const { data: { shelf, books } } = ctx;

    return html`
        <section>
            <header>
                <h1>${shelf.title}</h1>
            </header>
            ${cardGrid(
                books.map(bookCard),
                html`No books yet, <a href="/books">find some books</a> to add`,
            )}
            <p><a href="/shelves">Back to all shelves</a></p>
        </section>
    `;
}
```

- The books are shown with `bookCard` in a `cardGrid`, exactly as on the books page.
- An empty shelf gets a message with a link to the books page. The message has a link in it, so it is made with the `html` helper, and `cardGrid` puts it in the page without escaping it, just like [any other HTML we nest][nesting].

Visit [http://localhost:8000/shelves/1][shelf 1].

{{<figure src="images/shelf-books.webp" alt="The classics shelf page, with four book cards: To Kill a Mockingbird, The Great Gatsby, Pride and Prejudice and Jane Eyre, and a Back to all shelves link below" caption="A shelf's page shows its books.">}}

The books are not in the order we listed them in **db/seed.json**.
Our query has no `ORDER BY`, so the database returns the rows in whatever order is quickest for it.
Here that happens to be the order of the books' ids.
We have seen that [paging needs a stable order][stable order], but a shelf's books all fit on one page, so for now any order will do.

Each card links to its book's page, so the book pages and the shelf pages are linked together now.

## A book count {#book-count}

The shelves page should say how many books each shelf holds.
We could run a query for each shelf, but SQL can count them in the same query that finds the shelves.

Update `getShelves` in **server/models/shelves.js**.

**server/models/shelves.js**
```JavaScript{hl_lines="19-23"}
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
    return db.prepare(`
        SELECT id, title,
            (SELECT count(*) FROM shelfBooks WHERE shelfId = shelves.id) AS bookCount
        FROM shelves
    `).all();
}
```

The second line of the `SELECT` is a *subquery*, a query in brackets inside another query.

- For each shelf, it counts the rows in `shelfBooks` for that shelf, with [`count(*)`][count].
- `shelves.id` means the `id` of the shelf that the outer query is working on, so the subquery runs once for each shelf.
- `AS bookCount` names the result, so each shelf object gets a `bookCount` property.

> All three placeholders are still there.
> `getShelves` still finds every shelf, until shelves have owners.

Update `shelfCard` in **server/views/shelves.js** to show the count.

**server/views/shelves.js**
```JavaScript{hl_lines="23-26"}
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
                <div class="row">
                    <h3><a href="/shelves/${shelf.id}">${shelf.title}</a></h3>
                    <p>${shelf.bookCount} ${shelf.bookCount === 1 ? "book" : "books"}</p>
                </div>
            </div>
        </li>
    `;
}
```

- The title and the count sit together in the same `.row` that we wrote for the [book page][row].
- We say **book** for exactly one, and **books** otherwise, so we get **1 book**, **4 books** and **0 books**.

Visit [http://localhost:8000/shelves][shelves].
**classics** and **fantasy** have 4 books each, and **to read** has 3.

## Covers on shelf cards {#thumbnails}

A shelf card would be much easier to recognise with a few of its covers.
We will show up to five small thumbnails on each card.

Add two functions to **server/models/shelfBooks.js**.

**server/models/shelfBooks.js**
```JavaScript{hl_lines="16-34"}
import { db } from "./db.js";

export function getBooksByShelfId(shelfId) {
    return db.prepare(`
        SELECT books.id, title, author, year, thumbnailUrl, coverUrl
        FROM shelfBooks JOIN books ON books.id = shelfBooks.bookId
        WHERE shelfId=:shelfId
    `).all({ shelfId });
}

export function addBookToShelf(shelfId, bookId) {
    return db.prepare(
        `INSERT OR IGNORE INTO shelfBooks(shelfId, bookId) VALUES (:shelfId, :bookId)`,
    ).run({ shelfId, bookId });
}

// Up to 5 small covers to show on a shelf card
function getThumbnailsByShelfId(shelfId) {
    return db.prepare(`
        SELECT books.id, thumbnailUrl
        FROM shelfBooks JOIN books ON books.id = shelfBooks.bookId
        WHERE shelfId=:shelfId
        LIMIT 5
    `).all({ shelfId });
}

// Adds each shelf's thumbnails to the shelves it's given
// This runs one small query per shelf, which is fine for a handful of shelves
export function addThumbnails(shelves) {
    for (const shelf of shelves) {
        shelf.thumbnails = getThumbnailsByShelfId(shelf.id);
    }
    return shelves;
}
```

- `getThumbnailsByShelfId` uses the same `JOIN` as `getBooksByShelfId`, but only selects the `id` and the small `thumbnailUrl`, and stops after 5 rows with `LIMIT 5`.
- `addThumbnails` takes the list of shelves, gives each one a `thumbnails` property, and returns the list.
- Only `addThumbnails` calls `getThumbnailsByShelfId`, so `getThumbnailsByShelfId` isn't exported.

This runs one small query for each shelf, as the comment says.
With three shelves that is three quick queries, which is fine.
With thousands of shelves on one page it would start to add up, and we would look for a way to get all the thumbnails at once.

Update **server/controllers/shelves.js** to add the thumbnails to the shelves.

**server/controllers/shelves.js**
```JavaScript{hl_lines="2 9"}
import { getShelfById, getShelves } from "../models/shelves.js";
import { addThumbnails, getBooksByShelfId } from "../models/shelfBooks.js";
import { render } from "../tools/render.js";
import { NotFoundError } from "../tools/errors.js";
import { shelvesView } from "../views/shelves.js";
import { shelfView } from "../views/shelf.js";

export function shelvesController(ctx) {
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

Update **server/views/shelves.js** to show them.

**server/views/shelves.js**
```JavaScript{hl_lines="2 19 24-26 36-41"}
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

- A shelf with thumbnails gets a `<div class="covers">` with an image for each one. An empty shelf gets a paragraph saying **No books yet** in the same place.
- A cover is drawn once for each book, so drawing one gets a name: `coverImage`. It sits beside `shelfCard`, because only the shelves page uses it.
- `coverImage` uses `DEFAULT_COVER` from the book card partial for a book with no thumbnail.
- The covers are decoration, because the shelf's title says what the card is, so their `alt` is empty.

Visit [http://localhost:8000/shelves][shelves].
The covers are there, along the top of each card.
But they are pushed to the left, nothing separates them from the text, and the title and the book count are bunched together.

{{<figure src="images/shelf-covers-unstyled.webp" alt="The shelves page, where each card has its small covers in a row at the top left, with the title and book count squeezed together underneath" caption="The covers need some styles.">}}

## Styling the shelf cards {#shelf-card-styles}

Add two rules to the end of **public/styles.css**.

**public/styles.css**
```css {linenostart="200"}
/* A shelf card: a strip of covers on top, the text fills the rest */
.card.shelf {
    grid-template-rows: auto 1fr;
    /* The row's items are pushed to opposite ends */
    .row {
        justify-content: space-between;
    }
}

/* A centred strip of covers with a line underneath */
.covers {
    display: flex;
    justify-content: center;
    gap: 2px;
    padding: 0.5rem;
    border-block-end: 1px solid;
    /* min-inline-size: 0 lets the covers shrink to fit the card */
    img {
        min-inline-size: 0;
        object-fit: cover;
    }
}
```

- A shelf card has two rows: the strip of covers on top, sized to fit them (`auto`), and the text below, filling the rest (`1fr`).
- Inside a shelf card, `justify-content: space-between` pushes the title and the book count to opposite ends of their `.row`.
- `.covers` lays the covers out in one centred row, with a small gap, some padding, and a line underneath with [`border-block-end`][border block end].
- An image in a flex row won't shrink below its own width unless we let it. `min-inline-size: 0` lets the covers shrink to fit a narrow card, and `object-fit: cover` stops them looking squashed when they do.

Notice that `.covers` styles both the `<div>` of covers and the **No books yet** paragraph, so an empty shelf has a line in the same place as the others.

Refresh [http://localhost:8000/shelves][shelves].

{{<figure src="images/shelf-cards.webp" alt="The shelves page with three cards. Each has a centred strip of small covers with a line underneath, then the shelf's title on the left and its book count on the right" caption="Each shelf card shows its covers, title and book count.">}}

Make the window narrow, and the covers shrink to fit their card.

## Conclusion

Books and shelves are linked now.
`shelfBooks` is a join table with one row for each book on each shelf, and its foreign keys and primary key keep the data honest.
The seeder puts books on the seed shelves by title, and stops with a clear error if a title is wrong.

A shelf's page shows its books with the `bookCard` partial, using a `JOIN` to find them.
The shelves page counts each shelf's books with a subquery, and shows up to five covers on each card.

Over the whole unit, books and shelves became *resources*, each with its own address and its own page, and links take us from one to another.
Everything so far has been read-only: we can look at shelves, but we can't change them.
In the next unit we will add forms, so that we can make a new shelf, rename it, delete it, and put books on it.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="26 35"}
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

## Challenge: Explore the join

These are questions to investigate, not changes to keep.

- Add **The Hobbit** to the **classics** shelf in **db/seed.json**, twice, and reset the database. How many books does **classics** have now, and why?
- Add a fourth shelf called **poetry** with an empty `books` list, and reset the database. What do its card and its page show?
- Add a book called **The Awakening** to a shelf, and reset the database. Search for [**the awakening**][q awakening]. Which book ends up on the shelf, and why?
- Add `ORDER BY title` to `getBooksByShelfId`. What changes on [http://localhost:8000/shelves/1][shelf 1]?
- How many queries does the shelves page run with three shelves? How many would it run with 100?

<details>
<summary>What to expect</summary>

**The Hobbit** is listed twice, but `INSERT OR IGNORE` ignores the second one, because the pair is already in `shelfBooks`.
**classics** has 5 books.

**poetry** has **No books yet** on its card, **0 books** as its count, and a link to find some books on its page.

There are two books called **The Awakening**.
`bookIds` remembers one `id` for each title, so the second book overwrites the first, and the second one goes on the shelf.
Looking books up by title is fine for our own small seed file, but it isn't a good way to find a book in general.

With `ORDER BY title`, the classics shelf shows **Jane Eyre**, **Pride and Prejudice**, **The Great Gatsby** and then **To Kill a Mockingbird**.

The shelves page runs one query for the shelves and their counts, then one for each shelf's thumbnails: 4 queries with three shelves, and 101 with 100.

</details>

> If you tried any changes, undo them, and run `deno task db:reset`, so that your code and data match the starting point of the next unit.

[previous exercise]: resources-and-links/shelves/
[books]: http://localhost:8000/books
[book 36]: http://localhost:8000/books/36
[shelves]: http://localhost:8000/shelves
[shelf 1]: http://localhost:8000/shelves/1
[q awakening]: http://localhost:8000/books?q=the+awakening
[foreign keys]: https://www.sqlite.org/foreignkeys.html
[destructuring]: javascript-fundamentals/books-data/#destructuring
[card grid]: resources-and-links/shelves/#card-grid
[join]: https://www.sqlite.org/lang_select.html#from_clause
[nesting]: user-input-and-security/escaping/#nesting
[stable order]: user-input-and-security/pagination/#one-page
[count]: https://www.sqlite.org/lang_aggfunc.html#count
[row]: resources-and-links/book-pages/#row
[border block end]: https://developer.mozilla.org/en-US/docs/Web/CSS/border-block-end

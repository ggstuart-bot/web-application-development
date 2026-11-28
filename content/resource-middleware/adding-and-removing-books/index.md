---
type: exercise
title: Adding and removing books
weight: 1
author: Dr Graeme Stuart
---

In this exercise we will put books on shelves from the browser.
A book's page will get an add button for each shelf, and each book on a shelf's page will get a remove button.

<!--more-->

> You should have the new shelf form and flash messages from the [previous unit][previous unit].
> Your database should hold the seeded books and the three seeded shelves, **classics**, **fantasy** and **to read**. If you have been experimenting, run `deno task db:reset` to get back to the seed data.
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

Our shelves only hold the books we seeded.
Now that we can [send a `POST` and redirect][prg], we can let users choose the books themselves.
By the end of this exercise you should be able to:

- Send a `POST` from a form with nothing but a button, with everything it needs in its address
- Build a feature in steps: a link that gets the not found page, a stub controller, then the real thing
- Answer every `POST` with a redirect and a flash message, even one we ignore
- Throw a `NotFoundError` when an id in the address doesn't exist
- Use the number of rows a query changed to choose a message
- Extract a partial, and give it an optional argument

## Add buttons on the book page {#add-buttons}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Open [http://localhost:8000/books/1][book 1], **The Hunger Games**.
We can see the book, but there's nothing we can do with it.

The book page needs to know about the shelves.
Update `bookController` in **server/controllers/books.js**.

**server/controllers/books.js**
```JavaScript {hl_lines="2 34"}
import { countPages, getBookById, searchBooks } from "../models/books.js";
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
    const { params: { bookId } } = ctx;
    const book = getBookById(Number(bookId));
    if (!book) {
        throw new NotFoundError(`Book ${bookId} not found`);
    }
    ctx.data.book = book;
    ctx.data.shelves = getShelves();
    ctx.data.title = book.title;
    return render(bookView, ctx);
}
```

`getShelves` is the same function the shelves page uses, so `ctx.data.shelves` holds every shelf.

Now update **server/views/book.js** to add an aside with a button for each shelf.

**server/views/book.js**
```JavaScript {hl_lines="5 22-27 30-41"}
import { html } from "../tools/html.js";
import { DEFAULT_COVER } from "./partials/bookCard.js";

export function bookView(ctx) {
    const { data: { book, shelves } } = ctx;

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
        <aside>
            <h2>Add to a shelf</h2>
            <div class="grid">
            ${addToShelfForms(book, shelves)}
            </div>
        </aside>
    `;
}

function addToShelfForms(book, shelves) {
    return shelves.length
        ? shelves.map((shelf) =>
            html`
                <form method="POST" action="/shelves/${shelf.id}/books/${book.id}/add">
                    <button>add to '${shelf.title}'</button>
                </form>
            `
        )
        : html`<p>You don't have any shelves yet</p>`;
}
```

- The aside sits beside the book's details. Our stylesheet already puts an aside in a second column on a wide screen, with a line between it and the section, just as it does for the [new shelf form][aside styles]. This aside isn't a `secondary` form, so it stays light and the buttons stay outlined.
- `addToShelfForms` makes one small form for each shelf. With no shelves, it says so instead.
- Each form has `method="POST"` and a single button, but no inputs at all.

The address says everything the server needs to know.
`/shelves/3/books/1/add` means "add book 1 to shelf 3".
The shelf's id and the book's id are both in the path, so there's nothing for the user to type.

Refresh [http://localhost:8000/books/1][book 1].
The aside has a button for each shelf: **add to 'classics'**, **add to 'fantasy'** and **add to 'to read'**.

{{<figure src="images/add-buttons.webp" alt="The Hunger Games book page, with the cover and details on the left and a light Add to a shelf aside on the right, separated by a vertical line, with outlined buttons reading add to 'classics', add to 'fantasy' and add to 'to read'" caption="Each shelf gets its own add button.">}}

Press **add to 'to read'**.
There's no route for that address yet, so we get our **Not found** page.

```plaintext {linenos=false}

============================
POST /shelves/3/books/1/add
[error] POST /shelves/3/books/1/add doesn't match any known route
RESPONSE 404
============================
```

That's fine for now.
The log tells us the button sent a `POST` to the address we expected.

## A stub controller {#add-stub}

Before we make the button do anything, we will check that each button reaches the server with the right ids.
A *stub* is a controller that does the minimum: it answers the request, but doesn't do the work yet.

Adding a book to a shelf works on the links between shelves and books, which our `shelfBooks` model already looks after.
Create a new file, **server/controllers/shelfBooks.js**, for the controllers that change those links.

**server/controllers/shelfBooks.js**
```JavaScript {hl_lines="1-7"}
import { redirect } from "../tools/redirect.js";

export function addBookController(ctx) {
    const { params: { shelfId, bookId } } = ctx;
    ctx.flash = `Ignored request to add book ${bookId} to shelf ${shelfId}`;
    return redirect(`/books/${bookId}`, ctx);
}
```

- The route's address has two parameters, `:shelfId` and `:bookId`, and the router puts them both on `ctx.params`, just as it does for the book and shelf pages.
- The stub doesn't add anything. It sets a flash message saying what it was asked to do.
- It redirects back to the book page.

Even a request that we ignore is still a `POST`, so it still ends with a redirect, [Post/Redirect/Get][prg].
Because we pass `ctx` to `redirect`, it [sends the flash cookie][set flash] too.

Add the route to **server/main.js**.

**server/main.js**
```JavaScript {hl_lines="3 26"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import { addBookController } from "./controllers/shelfBooks.js";
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
app.post("/shelves/:shelfId/books/:bookId/add", addBookController);

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

Go back to [http://localhost:8000/books/1][book 1] and press **add to 'to read'**.
You come straight back to the same page, with the message **Ignored request to add book 1 to shelf 3**.
Try the other two buttons, and check that each one names the right shelf.
Then try a different book, such as [http://localhost:8000/books/2][book 2].

```plaintext {linenos=false}

============================
POST /shelves/3/books/1/add
[flash sent] Ignored request to add book 1 to shelf 3
RESPONSE 303
============================

============================
GET /books/1
[flash received] Ignored request to add book 1 to shelf 3
RESPONSE 200
============================
```

### Changing the address {#stub-address}

The stub believes whatever ids it is sent.
We can see that by changing a button in the browser's developer tools.

1. Open [http://localhost:8000/books/1][book 1], and open the developer tools (**F12**) on the **elements** panel.
2. Find the form for the first button: `<form method="POST" action="/shelves/1/books/1/add">`.
3. Double-click the `action` value, change it to `/shelves/999/books/1/add`, and press **Enter**.
4. Press the button.

The message says **Ignored request to add book 1 to shelf 999**, though there is no shelf 999.
The stub never looks anything up, so it can't tell.
The real controller will have to.

## Adding the book {#add-book}

Now we can make the button do its job.
Update **server/controllers/shelfBooks.js**.

**server/controllers/shelfBooks.js**
```JavaScript {hl_lines="1-4 9-21"}
import { getBookById } from "../models/books.js";
import { getShelfById } from "../models/shelves.js";
import { addBookToShelf } from "../models/shelfBooks.js";
import { NotFoundError } from "../tools/errors.js";
import { redirect } from "../tools/redirect.js";

export function addBookController(ctx) {
    const { params: { shelfId, bookId } } = ctx;
    const shelf = getShelfById(Number(shelfId));
    if (!shelf) {
        throw new NotFoundError(`Shelf ${shelfId} not found`);
    }
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
```

- We load the shelf and the book from their ids. The ids in `ctx.params` are strings, so we turn them into numbers first.
- If either one is missing, we throw a `NotFoundError`, just as the book and shelf pages do. Our `withErrors` middleware turns it into the not found page.
- `addBookToShelf` adds the link. It's the same function our seed task uses to put the seeded books on their shelves.
- The redirect and the flash message use the shelf and book we loaded, so the message can name them.

`addBookToShelf` uses `INSERT OR IGNORE`.
The `shelfBooks` table can only hold each pair of shelf and book once, so adding a book that's already on the shelf would normally fail.
With `OR IGNORE`, SQLite quietly skips it instead.

The `run` method gives back the number of rows the query changed.
That's `1` when the book was added, and `0` when it was ignored.
`0` is falsy, so the [conditional operator][conditional operator] chooses the right message.

Go to [http://localhost:8000/books/1][book 1] and press **add to 'to read'**.
This time the message is **Book 'The Hunger Games' added to 'to read'**.
Press it again, and the message is **Book 'The Hunger Games' is already on 'to read'**.

Open the **to read** shelf at [http://localhost:8000/shelves/3][shelf 3].
**The Hunger Games** is there with the seeded books, and the shelf card on [http://localhost:8000/shelves][shelves] counts it too.

### Changing the address again {#missing-shelf}

Try the [developer tools trick][stub address] again, pointing the first button at `/shelves/999/books/1/add`.
This time you get our **Not found** page.

```plaintext {linenos=false}

============================
POST /shelves/999/books/1/add
[error] Shelf 999 not found
RESPONSE 404
============================
```

Try a missing book too, by changing the book id to `99999`.
The terminal says **Book 99999 not found**.

> Refresh the page to get the real buttons back.

## A postButton partial {#post-button}

A form with nothing but a button is how we send a `POST` without asking the user for anything.
We are about to need more of them, so it's time for a partial.

Create a new file, **server/views/partials/postButton.js**.

**server/views/partials/postButton.js**
```JavaScript {hl_lines="1-9"}
import { html } from "../../tools/html.js";

export function postButton(action, label) {
    return html`
        <form method="POST" action="${action}">
            <button>${label}</button>
        </form>
    `;
}
```

`postButton` takes the form's `action` and the button's `label`, and gives us the form.

Update **server/views/book.js** to use it.

**server/views/book.js**
```JavaScript {hl_lines="3 35"}
import { html } from "../tools/html.js";
import { DEFAULT_COVER } from "./partials/bookCard.js";
import { postButton } from "./partials/postButton.js";

export function bookView(ctx) {
    const { data: { book, shelves } } = ctx;

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
        <aside>
            <h2>Add to a shelf</h2>
            <div class="grid">
            ${addToShelfForms(book, shelves)}
            </div>
        </aside>
    `;
}

function addToShelfForms(book, shelves) {
    return shelves.length
        ? shelves.map((shelf) =>
            postButton(`/shelves/${shelf.id}/books/${book.id}/add`, html`add to '${shelf.title}'`)
        )
        : html`<p>You don't have any shelves yet</p>`;
}
```

The label is made with our `html` helper, so the shelf title inside it is escaped once, and the rest of the label is used as it is.
A plain string, like the `"remove"` label we will pass soon, is escaped by `postButton` itself.
Either way, the title is escaped exactly once, just as we saw with [HTML inside HTML][nesting].

Refresh [http://localhost:8000/books/1][book 1].
The page looks exactly the same.

## Remove buttons on the shelf page {#remove-buttons}

Each book on a shelf's page needs a remove button.
The books are shown with `bookCard`, so the card needs somewhere to put a button.

Update **server/views/partials/bookCard.js**.

**server/views/partials/bookCard.js**
```JavaScript {hl_lines="5-6 14"}
import { html } from "../../tools/html.js";

export const DEFAULT_COVER = "/assets/default-cover.svg";

// A book as a card in a .grid list. Pass extra content (e.g. a form) to show it in the card footer
export function bookCard(book, footer = "") {
    const cover = book.coverUrl ?? DEFAULT_COVER;
    return html`
        <li class="card book">
            <img class="cover" src="${cover}" alt="" width="200" height="300" loading="lazy">
            <div>
                <h2><a href="/books/${book.id}">${book.title}</a></h2>
                <p>${book.author}${book.year ? ` (${book.year})` : ""}</p>
                ${footer ? html`<footer>${footer}</footer>` : ""}
            </div>
        </li>
    `;
}
```

- `bookCard` takes an optional second argument, `footer`. It is a [default parameter][default parameters], so it's an empty string if we leave it out.
- If there is a footer, the card shows it in a [`<footer>`][footer] at the bottom of its text.
- The comment explains the new argument.

Update **server/views/shelf.js** to pass a remove button to each card.

**server/views/shelf.js**
```JavaScript {hl_lines="4 15 22-25"}
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
    `;
}

function removeForm(shelf, book) {
    return postButton(`/shelves/${shelf.id}/books/${book.id}/delete`, "remove");
}
```

`removeForm` uses `postButton` for the second time, with an address that ends in `/delete`.

There's one more change, in **server/views/books.js**.

**server/views/books.js**
```JavaScript {hl_lines="17"}
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
                books.map((book) => bookCard(book)),
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

We used to pass `bookCard` straight to `map`.
But `map` calls its function with each item's index as the second argument, and `bookCard` would now take that as a footer.
The first card's index is `0`, which is falsy, but every other card would show its number in a footer.
The [arrow function][arrow functions] calls `bookCard` with just the book.

Open the **classics** shelf at [http://localhost:8000/shelves/1][shelf 1].
The cards are **To Kill a Mockingbird**, **The Great Gatsby**, **Pride and Prejudice** and **Jane Eyre**, each with a **remove** button.
The books page, [http://localhost:8000/books][books], has no buttons and no numbers.

{{<figure src="images/remove-buttons.webp" alt="The classics shelf, with four book cards in order: To Kill a Mockingbird, The Great Gatsby, Pride and Prejudice and Jane Eyre, each with an outlined remove button, and a Back to all shelves link underneath" caption="Each book on a shelf gets a remove button.">}}

Press a **remove** button.
There's no route yet, so we get the **Not found** page again.

## A stub for removing {#remove-stub}

Just like adding, we start with a stub.
Add `removeBookController` to the end of **server/controllers/shelfBooks.js**.

**server/controllers/shelfBooks.js**
```JavaScript {hl_lines="23-28"}
import { getBookById } from "../models/books.js";
import { getShelfById } from "../models/shelves.js";
import { addBookToShelf } from "../models/shelfBooks.js";
import { NotFoundError } from "../tools/errors.js";
import { redirect } from "../tools/redirect.js";

export function addBookController(ctx) {
    const { params: { shelfId, bookId } } = ctx;
    const shelf = getShelfById(Number(shelfId));
    if (!shelf) {
        throw new NotFoundError(`Shelf ${shelfId} not found`);
    }
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
    const { params: { shelfId, bookId } } = ctx;
    ctx.flash = `Ignored request to remove book ${bookId} from shelf ${shelfId}`;
    return redirect(`/shelves/${shelfId}`, ctx);
}
```

It flashes a message with the two ids and redirects back to the shelf page.

Add the route to **server/main.js**.

**server/main.js**
```JavaScript {hl_lines="3 27"}
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
app.post("/shelves/:shelfId/books/:bookId/add", addBookController);
app.post("/shelves/:shelfId/books/:bookId/delete", removeBookController);

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

Go back to [http://localhost:8000/shelves/1][shelf 1] and press **remove** on **Jane Eyre**.
You come back to the shelf, with a message like **Ignored request to remove book 40 from shelf 1**, and the book is still there.

## Removing the book {#remove-book}

Removing a book needs a new function in our model.
Add `removeBookFromShelf` to **server/models/shelfBooks.js**, after `addBookToShelf`.

**server/models/shelfBooks.js**
```JavaScript {linenostart="11", hl_lines="7-11"}
export function addBookToShelf(shelfId, bookId) {
    return db.prepare(
        `INSERT OR IGNORE INTO shelfBooks(shelfId, bookId) VALUES (:shelfId, :bookId)`,
    ).run({ shelfId, bookId });
}

export function removeBookFromShelf(shelfId, bookId) {
    return db.prepare(
        `DELETE FROM shelfBooks WHERE shelfId=:shelfId AND bookId=:bookId`,
    ).run({ shelfId, bookId });
}
```

It deletes the row that links the shelf to the book.
The book itself stays in the `books` table, so it is still in the catalogue.

Now update `removeBookController` in **server/controllers/shelfBooks.js**.

**server/controllers/shelfBooks.js**
```JavaScript {hl_lines="3 26-36"}
import { getBookById } from "../models/books.js";
import { getShelfById } from "../models/shelves.js";
import { addBookToShelf, removeBookFromShelf } from "../models/shelfBooks.js";
import { NotFoundError } from "../tools/errors.js";
import { redirect } from "../tools/redirect.js";

export function addBookController(ctx) {
    const { params: { shelfId, bookId } } = ctx;
    const shelf = getShelfById(Number(shelfId));
    if (!shelf) {
        throw new NotFoundError(`Shelf ${shelfId} not found`);
    }
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
    const { params: { shelfId, bookId } } = ctx;
    const shelf = getShelfById(Number(shelfId));
    if (!shelf) {
        throw new NotFoundError(`Shelf ${shelfId} not found`);
    }
    const book = getBookById(Number(bookId));
    if (!book) {
        throw new NotFoundError(`Book ${bookId} not found`);
    }
    removeBookFromShelf(shelf.id, book.id);
    ctx.flash = `Book '${book.title}' removed`;
    return redirect(`/shelves/${shelf.id}`, ctx);
}
```

It loads the shelf and the book exactly as `addBookController` does, removes the link, and redirects back to the shelf with a new message.

Press **remove** on **Jane Eyre** at [http://localhost:8000/shelves/1][shelf 1].
The message says **Book 'Jane Eyre' removed**, and the card has gone.
Search for [Jane Eyre][jane eyre] on the books page, and it's still in the catalogue.

```plaintext {linenos=false}

============================
POST /shelves/1/books/40/delete
[flash sent] Book 'Jane Eyre' removed
RESPONSE 303
============================

============================
GET /shelves/1
[flash received] Book 'Jane Eyre' removed
RESPONSE 200
============================
```

> Your book ids may be different, if your seed data are different.

## Conclusion

We can now put any book on any shelf, and take it off again.
Each button is a small form that sends a `POST` to an address with the ids in it, made with our new `postButton` partial.
We built each feature in three steps: a button that got the not found page, a stub that flashed and redirected, and then the real controller.
The real controllers check that the shelf and book exist, make the change, and redirect with a message that says what happened.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="18 40"}
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
│   │   └── logging.js
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

## Challenge: The same few lines

These are questions to investigate, not changes to keep.

- Look at `addBookController` and `removeBookController`, and at `bookController` and `shelfController`. How many times do we load a shelf or a book from the address, and throw a `NotFoundError` if it's missing?
- Remove **Jane Eyre** from **classics**, then use the [developer tools trick][stub address] to send the same remove request again. What does the message say? Is it true? How could `removeBookController` use the number that `removeBookFromShelf` gives back, as `addBookController` does?
- In **server/views/books.js**, put `books.map(bookCard)` back for a moment, and look at the books page. What do the cards show, and why doesn't the first one show anything?
- What would the book page show if there were no shelves at all? Which line of **server/views/book.js** would make that happen?

<details>
<summary>The repeated lookups</summary>

`shelfController`, `addBookController` and `removeBookController` all load the shelf, and `bookController`, `addBookController` and `removeBookController` all load the book.
That's six copies of nearly the same four lines.
In the [next exercise][next exercise] we will move each lookup into a middleware that runs before the controller, so each controller can start with the shelf and the book it needs.

</details>

> If you tried any changes, undo them, and run `deno task db:reset`, so that your code and data match the starting point of the next exercise.

[previous unit]: post-redirect-flash/
[prg]: post-redirect-flash/form-that-posts/#post-redirect-get
[aside styles]: post-redirect-flash/form-that-posts/#aside-styles
[set flash]: post-redirect-flash/cookies-and-flash/#set-flash
[book 1]: http://localhost:8000/books/1
[book 2]: http://localhost:8000/books/2
[shelf 1]: http://localhost:8000/shelves/1
[shelf 3]: http://localhost:8000/shelves/3
[shelves]: http://localhost:8000/shelves
[books]: http://localhost:8000/books
[jane eyre]: http://localhost:8000/books?q=jane+eyre
[stub address]: resource-middleware/adding-and-removing-books/#stub-address
[conditional operator]: user-input-and-security/search/#title
[nesting]: user-input-and-security/escaping/#nesting
[default parameters]: views-and-controllers/controllers/#options
[footer]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/footer
[arrow functions]: javascript-fundamentals/book-search/#arrow-functions
[next exercise]: resource-middleware/route-middleware/

---
type: exercise
title: Owners
weight: 2
author: Dr Graeme Stuart
---

In this exercise we will give every shelf an owner.
Each user will see their own shelves, other people's shelves will be shown without their buttons, and only a shelf's owner will be able to change it.

<!--more-->

> You should have the `mustBeSignedIn` and `mustBeSignedOut` guards from the [previous exercise][previous exercise].
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

Our guards ask one question: is anyone signed in?
Now we need to ask a harder one: is this *your* shelf?
By the end of this exercise you should be able to:

- Add a column that links each row to its owner, and seed data that belong to users
- Filter a list in SQL, so a user only gets their own rows
- Show a different view of the same page to its owner and to everyone else
- Explain why hiding a button is not enough, and forge a form in the browser's developer tools
- Guard a route with a middleware that checks the thing the route is about
- Explain when we answer `404 Not Found` rather than `403 Forbidden`

## Every shelf has an owner {#owner-id}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Shelves have belonged to nobody so far.
Now that we have users, each shelf should belong to the user who made it.

Update **tasks/db-create.js** to give the shelves table an `ownerId` column.

**tasks/db-create.js**
```JavaScript{hl_lines="18 22-23"}
import { db } from "../server/models/db.js";

db.exec("DROP TABLE IF EXISTS sessions");
db.exec("DROP TABLE IF EXISTS shelfBooks");
db.exec("DROP TABLE IF EXISTS shelves");
db.exec("DROP TABLE IF EXISTS books");
db.exec("DROP TABLE IF EXISTS users");

// Placeholder: this will need more columns later
db.exec(`
    CREATE TABLE users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT UNIQUE NOT NULL,
        hashedPassword TEXT NOT NULL
    )
`);

// Placeholder: this will need to change when shelves can be public
db.exec(`
    CREATE TABLE shelves (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT NOT NULL,
        ownerId INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE
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

db.exec(`
    CREATE TABLE sessions (
        id TEXT PRIMARY KEY,
        userId INTEGER NOT NULL,
        expires TEXT NOT NULL,
        FOREIGN KEY (userId) REFERENCES users(id) ON DELETE CASCADE
    )
`);
```

`ownerId` holds the `id` of the user who owns the shelf.

- `INTEGER NOT NULL` means every shelf *must* have an owner. The database won't store a shelf without one.
- `REFERENCES users(id)` makes it a *foreign key*, just like the columns in the `shelfBooks` and `sessions` tables. The database won't store an `ownerId` that isn't a real user.
- `ON DELETE CASCADE` means that deleting a user deletes their shelves too, so no shelf is ever left pointing at a user who has gone.

This keeps our data honest.
Whatever our code does, the database itself makes sure that every shelf has an owner, and that the owner exists.

> The placeholder has changed.
> Shelves have owners now, but they will need to change again when they can be public.

Now update **server/models/shelves.js**.

**server/models/shelves.js**
```JavaScript{hl_lines="3-4 6-9 12 14 17 28 33-35"}
import { db } from "./db.js";

// Placeholder: this will need to change when shelves can be public
export function createShelf({ title, ownerId }) {
    return db.prepare(`
        INSERT INTO shelves(title, ownerId)
        VALUES (:title, :ownerId)
        RETURNING id, title, ownerId
    `).get({ title, ownerId });
}

// Placeholder: this will need to change when shelves can be public
export function getShelfById(id) {
    return db.prepare(`SELECT id, title, ownerId FROM shelves WHERE id=:id`).get({ id });
}

// Placeholder: everyone's shelves, until each user sees only their own
export function getShelves() {
    return db.prepare(`
        SELECT id, title,
            (SELECT count(*) FROM shelfBooks WHERE shelfId = shelves.id) AS bookCount
        FROM shelves
    `).all();
}

export function updateShelfTitleById(id, title) {
    return db.prepare(
        `UPDATE shelves SET title=:title WHERE id=:id RETURNING id, title, ownerId`,
    ).get({ id, title });
}

export function deleteShelfById(id) {
    return db.prepare(
        `DELETE FROM shelves WHERE id=:id RETURNING id, title, ownerId`,
    ).get({ id });
}
```

`createShelf` now takes an `ownerId` as well as a `title`, and stores it.
`createShelf`, `getShelfById`, `updateShelfTitleById` and `deleteShelfById` all return the `ownerId` with the rest of the shelf.

> The placeholders on `createShelf` and `getShelfById` will change again when shelves can be public.
> The placeholder on `getShelves` says what it does for now: it still finds everyone's shelves, until each user sees only their own.
> `updateShelfTitleById` and `deleteShelfById` have lost their placeholders, because they are finished.

`createShelfController` needs to say who owns the new shelf.
Update **server/controllers/shelves.js**.

**server/controllers/shelves.js**
```JavaScript{hl_lines="20-21"}
import {
    createShelf,
    deleteShelfById,
    getShelves,
    updateShelfTitleById,
} from "../models/shelves.js";
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

export function createShelfController(ctx) {
    const { validated, user } = ctx;
    const shelf = createShelf({ title: validated.title, ownerId: user.id });
    ctx.flash = `Shelf '${shelf.title}' created`;
    return redirect(`/shelves/${shelf.id}`, ctx);
}

export function shelfController(ctx) {
    const { data: { shelf } } = ctx;
    ctx.data.title = shelf.title;
    ctx.data.books = getBooksByShelfId(shelf.id);
    return render(shelfView, ctx);
}

export function editShelfController(ctx) {
    const { validated, data: { shelf: { id } } } = ctx;
    const shelf = updateShelfTitleById(id, validated.title);
    ctx.flash = `Shelf renamed to '${shelf.title}'`;
    return redirect(`/shelves/${shelf.id}`, ctx);
}

export function deleteShelfController(ctx) {
    const { data: { shelf: { id } } } = ctx;
    const shelf = deleteShelfById(id);
    ctx.flash = `Shelf '${shelf.title}' deleted`;
    return redirect("/shelves", ctx);
}
```

The owner is the signed-in user.
We can use `user.id` without checking, because [`mustBeSignedIn`][must be signed in] runs first on this route, so `ctx.user` is always there.

The seed data need owners too.
Update **db/seed.json**, so that each user has their own shelves.

**db/seed.json**
```json{hl_lines="6-33 38-58"}
{
    "users": [
        {
            "username": "testUser1",
            "password": "testUser1!",
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
        },
        {
            "username": "testUser2",
            "password": "testUser2!",
            "shelves": [
                {
                    "title": "sci-fi",
                    "books": [
                        "Dune",
                        "Ender's Game",
                        "The Hitchhiker's Guide to the Galaxy",
                        "Fahrenheit 451",
                        "1984"
                    ]
                },
                {
                    "title": "children's favourites",
                    "books": [
                        "Charlotte's Web",
                        "Matilda",
                        "Where the Wild Things Are",
                        "The Lion, the Witch, and the Wardrobe"
                    ]
                }
            ]
        }
    ]
}
```

The shelves have moved inside the users, so the separate list of shelves has gone.
**testUser1** owns **classics**, **fantasy** and **to read**, and **testUser2** has two new shelves, **sci-fi** and **children's favourites**.

Update **tasks/db-seed.js** to match.

**tasks/db-seed.js**
```JavaScript{hl_lines="15-23"}
import seed from "../db/seed.json" with { type: "json" };
import books from "../db/books.json" with { type: "json" };
import { createUser } from "../server/models/users.js";
import { createShelf } from "../server/models/shelves.js";
import { createBook } from "../server/models/books.js";
import { addBookToShelf } from "../server/models/shelfBooks.js";

const bookIds = {};
for (const book of books) {
    const { id } = createBook(book);
    bookIds[book.title] = id;
}

for (const user of seed.users) {
    const { id: userId } = await createUser(user);
    for (const shelf of user.shelves) {
        const { id: shelfId } = createShelf({ title: shelf.title, ownerId: userId });
        for (const title of shelf.books) {
            const bookId = bookIds[title];
            if (!bookId) {
                throw new Error(`Book '${title}' not found in books.json`);
            }
            addBookToShelf(shelfId, bookId);
        }
    }
}
```

For each user, we create the user, then each of their shelves with the new user's id, then the books on that shelf.
`createUser` returns the new user, so we take its `id` out with destructuring and call it `userId`.

> The `Temporary:` comment has gone, because the shelves belong to users now.

The shelves table has a new column, so we need to rebuild the database.

```bash {linenos=false}
deno task db:reset
```

```plaintext {linenos=false}
Task db:reset deno task db:create && deno task db:seed
Task db:create deno -A tasks/db-create.js
Task db:seed deno -A tasks/db-seed.js
```

Resetting the database deletes every session too, so you are signed out.
Sign in as **testUser1** (the password is `testUser1!`) and open [http://localhost:8000/shelves][shelves].
There are five shelves now, three of them yours and two belonging to **testUser2**.
Every page still shows everyone's shelves.

## My shelves {#my-shelves}

The shelves page should show you your own shelves, not everyone's.

We need a way to find the shelves that belong to one user.
Add `getShelvesByOwnerId` to **server/models/shelves.js**.

**server/models/shelves.js**
```JavaScript{hl_lines="26-34"}
import { db } from "./db.js";

// Placeholder: this will need to change when shelves can be public
export function createShelf({ title, ownerId }) {
    return db.prepare(`
        INSERT INTO shelves(title, ownerId)
        VALUES (:title, :ownerId)
        RETURNING id, title, ownerId
    `).get({ title, ownerId });
}

// Placeholder: this will need to change when shelves can be public
export function getShelfById(id) {
    return db.prepare(`SELECT id, title, ownerId FROM shelves WHERE id=:id`).get({ id });
}

// Placeholder: everyone's shelves, until each user sees only their own
export function getShelves() {
    return db.prepare(`
        SELECT id, title,
            (SELECT count(*) FROM shelfBooks WHERE shelfId = shelves.id) AS bookCount
        FROM shelves
    `).all();
}

// Placeholder: this will need to change when shelves can be public
export function getShelvesByOwnerId(ownerId) {
    return db.prepare(`
        SELECT id, title, ownerId,
            (SELECT count(*) FROM shelfBooks WHERE shelfId = shelves.id) AS bookCount
        FROM shelves WHERE ownerId=:ownerId
    `).all({ ownerId });
}

export function updateShelfTitleById(id, title) {
    return db.prepare(
        `UPDATE shelves SET title=:title WHERE id=:id RETURNING id, title, ownerId`,
    ).get({ id, title });
}

export function deleteShelfById(id) {
    return db.prepare(
        `DELETE FROM shelves WHERE id=:id RETURNING id, title, ownerId`,
    ).get({ id });
}
```

`getShelvesByOwnerId` is just like `getShelves`, with a `WHERE` clause.
It finds only the shelves whose `ownerId` matches, along with the number of books on each.

We could fetch everyone's shelves and throw away the ones we don't want in JavaScript, but that would be wasteful, and easy to get wrong.
Instead, the database only ever gives us the rows we are allowed to show.
This is a rule we will keep: **a list is filtered in SQL**.

> The new function has a placeholder, because it will need to change when shelves can be public.
> `getShelves` is still here, with its placeholder, because a book's page still uses it.

The shelves page becomes **My shelves**.
Update **server/controllers/shelves.js**, renaming `shelvesController` to `myShelvesController`.

**server/controllers/shelves.js**
```JavaScript{hl_lines="4 10 13-17"}
import {
    createShelf,
    deleteShelfById,
    getShelvesByOwnerId,
    updateShelfTitleById,
} from "../models/shelves.js";
import { addThumbnails, getBooksByShelfId } from "../models/shelfBooks.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { myShelvesView } from "../views/myShelves.js";
import { shelfView } from "../views/shelf.js";

export function myShelvesController(ctx) {
    const { user } = ctx;
    ctx.data.shelves = addThumbnails(getShelvesByOwnerId(user.id));
    ctx.data.title = "My shelves";
    return render(myShelvesView, ctx);
}

export function createShelfController(ctx) {
    const { validated, user } = ctx;
    const shelf = createShelf({ title: validated.title, ownerId: user.id });
    ctx.flash = `Shelf '${shelf.title}' created`;
    return redirect(`/shelves/${shelf.id}`, ctx);
}

export function shelfController(ctx) {
    const { data: { shelf } } = ctx;
    ctx.data.title = shelf.title;
    ctx.data.books = getBooksByShelfId(shelf.id);
    return render(shelfView, ctx);
}

export function editShelfController(ctx) {
    const { validated, data: { shelf: { id } } } = ctx;
    const shelf = updateShelfTitleById(id, validated.title);
    ctx.flash = `Shelf renamed to '${shelf.title}'`;
    return redirect(`/shelves/${shelf.id}`, ctx);
}

export function deleteShelfController(ctx) {
    const { data: { shelf: { id } } } = ctx;
    const shelf = deleteShelfById(id);
    ctx.flash = `Shelf '${shelf.title}' deleted`;
    return redirect("/shelves", ctx);
}
```

- `myShelvesController` asks for the signed-in user's shelves, and gives the page the title **My shelves**.
- It uses `user.id` without checking, so we will need to put `mustBeSignedIn` in front of it.
- It renders `myShelvesView`, from a new **views/myShelves.js** module.

Rename **server/views/shelves.js** to **server/views/myShelves.js**, and update it.

**server/views/myShelves.js**
```JavaScript{hl_lines="6 12-13 15"}
import { html } from "../tools/html.js";
import { DEFAULT_COVER } from "./partials/bookCard.js";
import { cardGrid } from "./partials/cardGrid.js";
import { shelfTitleForm } from "./partials/shelfTitleForm.js";

export function myShelvesView(ctx) {
    const { data: { shelves, errors, values } } = ctx;

    return html`
        <section>
            <header>
                <h1>My shelves</h1>
                <p>Make a shelf, then add books to it from any book's page.</p>
            </header>
            <h2>Your shelves</h2>
            ${cardGrid(shelves.map(shelfCard), "No shelves yet")}
        </section>
        <aside>
            <h2>New shelf</h2>
            ${shelfTitleForm("/shelves", values?.title ?? "", errors?.title, "create")}
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

The view is now called `myShelvesView`.
It has the heading **My shelves**, a line saying what the page is for, and **Your shelves** above the list.

My shelves only makes sense for a signed-in user, so the nav link needs to change.
Update the `authLinks` function in **server/views/layout.js**.

**server/views/layout.js**
```JavaScript{hl_lines="51-54"}
import { html } from "../tools/html.js";
import { postButton } from "./partials/postButton.js";

export function layout(content, ctx) {
    const { data, url, user } = ctx;
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
                        ${authLinks(user, url.pathname)}
                    </nav>
                </header>
                ${flashMessage(data.flash)}
                <main>
                ${content}
                </main>
                <footer class="secondary">
                    ${userMessage(user)}
                    <p>
                        &copy; Bookshelves developers
                    </p>
                </footer>
            </body>
        </html>
    `;
}

function flashMessage(flash) {
    return flash ? html`<p id="flash" role="status" tabindex="0">${flash}</p>` : "";
}

function userMessage(user) {
    return user ? html`<p>Signed in as '${user.username}'</p>` : "";
}

function authLinks(user, path) {
    if (!user) return navLink("/signin", "sign in", path);
    return html`
        ${navLink("/shelves", "my shelves", path)}
        ${postButton("/signout", "sign out")}
    `;
}

// A header link, marked as the current page when it matches the path exactly
function navLink(href, content, path) {
    return html`<a href="${href}" aria-current="${
        href === path ? "page" : "false"
    }">${content}</a>`;
}
```

The **shelves** link has left the nav.
A signed-in user gets a **my shelves** link instead, next to the sign out button.
`authLinks` now returns two things, so we wrap them in one `html` template.

Finally, update **server/main.js**.

**server/main.js**
```JavaScript{hl_lines="8 31 54"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import { addBookController, removeBookController } from "./controllers/shelfBooks.js";
import {
    createShelfController,
    deleteShelfController,
    editShelfController,
    myShelvesController,
    shelfController,
} from "./controllers/shelves.js";
import { staticController } from "./controllers/static.js";
import { Router } from "./tools/router.js";
import { withLogs } from "./middleware/logging.js";
import { withErrors } from "./middleware/errors.js";
import { withValidation } from "./middleware/validation.js";
import { shelfSchema } from "./schema/shelf.js";
import { createUserController, signupFormController } from "./controllers/users.js";
import { userSchema } from "./schema/user.js";
import {
    alreadySignedInRedirectController,
    signinController,
    signinFormController,
    signinRedirectController,
    signoutController,
} from "./controllers/sessions.js";
import { signinSchema } from "./schema/signin.js";
import { withSession } from "./middleware/session.js";
import { requireGuest, requireUser } from "./middleware/access.js";
import { withBook, withShelf } from "./middleware/resources.js";

const validateNewShelf = withValidation(shelfSchema, myShelvesController);
const validateEditShelf = withValidation(shelfSchema, shelfController);
const validateNewUser = withValidation(userSchema, signupFormController);
const validateSignin = withValidation(signinSchema, signinFormController);

const mustBeSignedIn = requireUser(signinRedirectController);
const mustBeSignedOut = requireGuest(alreadySignedInRedirectController);

const app = new Router();

app.use(withLogs);
app.use(withErrors);
app.use(withSession);

app.get("/assets/*", staticController);
app.get("/", homeController);
app.get("/books", booksController);
app.get("/books/:bookId", withBook, bookController);
app.get("/signup", mustBeSignedOut, signupFormController);
app.post("/signup", mustBeSignedOut, validateNewUser, createUserController);
app.get("/signin", mustBeSignedOut, signinFormController);
app.post("/signin", mustBeSignedOut, validateSignin, signinController);
app.post("/signout", signoutController);
app.get("/shelves", mustBeSignedIn, myShelvesController);
app.post("/shelves", mustBeSignedIn, validateNewShelf, createShelfController);
app.get("/shelves/:shelfId", withShelf, shelfController);
app.post("/shelves/:shelfId/edit", mustBeSignedIn, withShelf, validateEditShelf, editShelfController);
app.post("/shelves/:shelfId/delete", mustBeSignedIn, withShelf, deleteShelfController);
app.post("/shelves/:shelfId/books/:bookId/add", mustBeSignedIn, withShelf, withBook, addBookController);
app.post("/shelves/:shelfId/books/:bookId/delete", mustBeSignedIn, withShelf, withBook, removeBookController);

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

- We import `myShelvesController` instead of `shelvesController`.
- When a new shelf's title is invalid, `validateNewShelf` falls back to `myShelvesController`, so the form is shown again on My shelves.
- `mustBeSignedIn` goes in front of `GET /shelves`, because `myShelvesController` needs a user.

As **testUser1**, click **my shelves** in the nav.
You see **classics**, **fantasy** and **to read**, and nothing else.
Sign out and sign in as **testUser2** (the password is `testUser2!`), and you see **sci-fi** and **children's favourites** instead.
Sign out and visit [http://localhost:8000/shelves][shelves], and you are asked to sign in.

{{<figure src="images/my-shelves.webp" alt="The My shelves page for testUser2, with sci-fi and children's favourites under the heading Your shelves, and the new shelf form beside them" caption="testUser2 sees only their own shelves.">}}

## Your shelves on a book's page {#book-shelves}

Sign in as **testUser2** and open [http://localhost:8000/books/1][book 1].
The book's page still offers an add button for every shelf, including **testUser1**'s **classics**.

Update `bookController` in **server/controllers/books.js** to ask for the signed-in user's shelves.

**server/controllers/books.js**
```JavaScript{hl_lines="2 28-31"}
import { countPages, searchBooks } from "../models/books.js";
import { getShelvesByOwnerId } from "../models/shelves.js";
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
    const { user } = ctx;
    if (user) {
        ctx.data.shelves = getShelvesByOwnerId(user.id);
    }
    ctx.data.title = ctx.data.book.title;
    return render(bookView, ctx);
}
```

This page is open to everyone, so we can't be sure there is a user.
With nobody signed in, we don't fetch any shelves at all.

Nothing calls `getShelves` now, so remove it, and its placeholder, from **server/models/shelves.js**.

**server/models/shelves.js**
```JavaScript
import { db } from "./db.js";

// Placeholder: this will need to change when shelves can be public
export function createShelf({ title, ownerId }) {
    return db.prepare(`
        INSERT INTO shelves(title, ownerId)
        VALUES (:title, :ownerId)
        RETURNING id, title, ownerId
    `).get({ title, ownerId });
}

// Placeholder: this will need to change when shelves can be public
export function getShelfById(id) {
    return db.prepare(`SELECT id, title, ownerId FROM shelves WHERE id=:id`).get({ id });
}

// Placeholder: this will need to change when shelves can be public
export function getShelvesByOwnerId(ownerId) {
    return db.prepare(`
        SELECT id, title, ownerId,
            (SELECT count(*) FROM shelfBooks WHERE shelfId = shelves.id) AS bookCount
        FROM shelves WHERE ownerId=:ownerId
    `).all({ ownerId });
}

export function updateShelfTitleById(id, title) {
    return db.prepare(
        `UPDATE shelves SET title=:title WHERE id=:id RETURNING id, title, ownerId`,
    ).get({ id, title });
}

export function deleteShelfById(id) {
    return db.prepare(
        `DELETE FROM shelves WHERE id=:id RETURNING id, title, ownerId`,
    ).get({ id });
}
```

A visitor who isn't signed in has no shelves, so the view needs to say something else.
Update **server/views/book.js**.

**server/views/book.js**
```JavaScript{hl_lines="6 26-28"}
import { html } from "../tools/html.js";
import { DEFAULT_COVER } from "./partials/bookCard.js";
import { postButton } from "./partials/postButton.js";

export function bookView(ctx) {
    const { user, data: { book, shelves } } = ctx;

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
            ${user
                ? addToShelfForms(book, shelves)
                : html`<p><a href="/signin">Sign in</a> to add this book to a shelf</p>`}
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

With a user, we show the add buttons as before.
Without one, we show a link to the sign in page instead.

Refresh [http://localhost:8000/books/1][book 1].
As **testUser2**, you can add it to **sci-fi** or **children's favourites**, and nothing else.
Signed out, the page says **Sign in to add this book to a shelf**.

## Other people's shelves {#public-shelf}

Sign in as **testUser2** and open [http://localhost:8000/shelves/1][shelf 1].
This is **testUser1**'s **classics** shelf.
It's fine for you to see it, but you also get the rename form, the delete button and a remove button on every book.
Those belong to the shelf's owner.

Update `shelfController` in **server/controllers/shelves.js** to choose a view for the shelf.

**server/controllers/shelves.js**
```JavaScript{hl_lines="8 12-13 30 33-37"}
import {
    createShelf,
    deleteShelfById,
    getShelvesByOwnerId,
    updateShelfTitleById,
} from "../models/shelves.js";
import { addThumbnails, getBooksByShelfId } from "../models/shelfBooks.js";
import { getUserById } from "../models/users.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { myShelvesView } from "../views/myShelves.js";
import { myShelfView } from "../views/myShelf.js";
import { publicShelfView } from "../views/publicShelf.js";

export function myShelvesController(ctx) {
    const { user } = ctx;
    ctx.data.shelves = addThumbnails(getShelvesByOwnerId(user.id));
    ctx.data.title = "My shelves";
    return render(myShelvesView, ctx);
}

export function createShelfController(ctx) {
    const { validated, user } = ctx;
    const shelf = createShelf({ title: validated.title, ownerId: user.id });
    ctx.flash = `Shelf '${shelf.title}' created`;
    return redirect(`/shelves/${shelf.id}`, ctx);
}

export function shelfController(ctx) {
    const { user, data: { shelf } } = ctx;
    ctx.data.title = shelf.title;
    ctx.data.books = getBooksByShelfId(shelf.id);
    if (user?.id === shelf.ownerId) {
        return render(myShelfView, ctx);
    }
    ctx.data.owner = getUserById(shelf.ownerId);
    return render(publicShelfView, ctx);
}

export function editShelfController(ctx) {
    const { validated, data: { shelf: { id } } } = ctx;
    const shelf = updateShelfTitleById(id, validated.title);
    ctx.flash = `Shelf renamed to '${shelf.title}'`;
    return redirect(`/shelves/${shelf.id}`, ctx);
}

export function deleteShelfController(ctx) {
    const { data: { shelf: { id } } } = ctx;
    const shelf = deleteShelfById(id);
    ctx.flash = `Shelf '${shelf.title}' deleted`;
    return redirect("/shelves", ctx);
}
```

- We compare the signed-in user's `id` with the shelf's `ownerId`.
- With nobody signed in, `user` is `undefined`, and `user?.id` is `undefined` too, rather than an error. `undefined` never matches an `ownerId`, so a visitor who isn't signed in owns nothing.
- The owner gets `myShelfView`, with all the controls.
- Everyone else gets `publicShelfView`. It needs to say whose shelf it is, so we look up the owner with `getUserById`.

Rename **server/views/shelf.js** to **server/views/myShelf.js**, and update it.

**server/views/myShelf.js**
```JavaScript{hl_lines="7 19"}
import { html } from "../tools/html.js";
import { bookCard } from "./partials/bookCard.js";
import { cardGrid } from "./partials/cardGrid.js";
import { postButton } from "./partials/postButton.js";
import { shelfTitleForm } from "./partials/shelfTitleForm.js";

export function myShelfView(ctx) {
    const { data: { shelf, books, errors, values } } = ctx;

    return html`
        <section>
            <header>
                <h1>${shelf.title}</h1>
            </header>
            ${cardGrid(
                books.map((book) => bookCard(book, removeForm(shelf, book))),
                html`No books yet, <a href="/books">find some books</a> to add`,
            )}
            <p><a href="/shelves">Back to my shelves</a></p>
        </section>
        <aside>
            <h2>Rename shelf</h2>
            ${shelfTitleForm(
                `/shelves/${shelf.id}/edit`,
                values?.title ?? shelf.title,
                errors?.title,
                "rename",
            )}
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

The view is now called `myShelfView`, and its link goes **Back to my shelves**.

Create **server/views/publicShelf.js** for everyone else.

**server/views/publicShelf.js**
```JavaScript{hl_lines="1-18"}
import { html } from "../tools/html.js";
import { bookCard } from "./partials/bookCard.js";
import { cardGrid } from "./partials/cardGrid.js";

export function publicShelfView(ctx) {
    const { data: { shelf, books, owner } } = ctx;

    return html`
        <section>
            <header>
                <h1>${shelf.title}</h1>
                <p>A shelf by ${owner.username}</p>
            </header>
            ${cardGrid(books.map((book) => bookCard(book)), "No books on this shelf yet")}
            <p><a href="/">Back to home</a></p>
        </section>
    `;
}
```

`publicShelfView` shows the shelf's books with no buttons at all, says whose shelf it is, and links back to the home page.
We call `bookCard(book)` without a footer, so the cards have no remove button.

Refresh [http://localhost:8000/shelves/1][shelf 1] as **testUser2**.
You see **A shelf by testUser1**, the books, and no buttons.
Sign in as **testUser1**, and the same address gives you the rename form, the delete button and the remove buttons.

{{<figure src="images/public-shelf.webp" alt="The classics shelf seen by testUser2, with the line A shelf by testUser1, four book cards with no remove buttons, and a Back to home link" caption="testUser2 sees testUser1's classics without the controls.">}}

### Hiding is not enough {#forged-form}

That only changes what the page *shows*.
The forms post to other routes, and those routes still let anyone who is signed in change any shelf.
We don't need any special tools to prove it, just the browser's developer tools.

> Try this on the seed data, then run `deno task db:reset` to get your shelves back.

1. Sign in as **testUser2** and open your own **sci-fi** shelf, [http://localhost:8000/shelves/4][shelf 4].
2. Open the developer tools (**F12**) on the **elements** panel, and find the delete button's form: `<form method="POST" action="/shelves/4/delete">`.
3. Double-click the `action` value, change it to `/shelves/1/delete`, and press **Enter**.
4. Press the **delete shelf** button.

You land on My shelves with the message **Shelf 'classics' deleted**.
**testUser1**'s classics shelf is gone, and **sci-fi** is still there.

The browser belongs to the user, and they can change anything in it.
Renaming a shelf, and adding or removing its books, are just as open.
Hiding a button only stops honest users from pressing it.

Reset the database to get **classics** back.

```bash {linenos=false}
deno task db:reset
```

## Only the owner {#may-use-shelf}

Every route that changes a shelf needs the same ownership check.
Like signing in, it becomes a middleware.

Add `mayUseShelf` to **server/middleware/access.js**.

**server/middleware/access.js**
```JavaScript{hl_lines="1-2 16-21"}
import { NotFoundError } from "../tools/errors.js";

export function requireUser(fallback) {
    return (ctx, next) => {
        if (ctx.user) return next(ctx);
        return fallback(ctx);
    };
}

export function requireGuest(fallback) {
    return (ctx, next) => {
        if (!ctx.user) return next(ctx);
        return fallback(ctx);
    };
}

export function mayUseShelf(ctx, next) {
    if (ctx.user?.id === ctx.data.shelf.ownerId) return next(ctx);
    const { url, request } = ctx;
    throw new NotFoundError(`${request.method} ${url.pathname} is not permitted`);
}
```

`mayUseShelf` is a plain middleware, like `withShelf`, not a factory like `requireUser`.

- It compares the signed-in user's `id` with the `ownerId` of the shelf on `ctx.data`, the same question that `shelfController` asks.
- If they match, the rest of the route runs.
- If not, it throws a [`NotFoundError`][not found error], so the not found page answers, and the message says which request was turned away.

The shelf has to be on `ctx.data` already, so `mayUseShelf` must come after `withShelf`, which loads it.

Update **server/main.js** to put `mayUseShelf` on the four routes that change a shelf.

**server/main.js**
```JavaScript{hl_lines="28 57-60"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import { addBookController, removeBookController } from "./controllers/shelfBooks.js";
import {
    createShelfController,
    deleteShelfController,
    editShelfController,
    myShelvesController,
    shelfController,
} from "./controllers/shelves.js";
import { staticController } from "./controllers/static.js";
import { Router } from "./tools/router.js";
import { withLogs } from "./middleware/logging.js";
import { withErrors } from "./middleware/errors.js";
import { withValidation } from "./middleware/validation.js";
import { shelfSchema } from "./schema/shelf.js";
import { createUserController, signupFormController } from "./controllers/users.js";
import { userSchema } from "./schema/user.js";
import {
    alreadySignedInRedirectController,
    signinController,
    signinFormController,
    signinRedirectController,
    signoutController,
} from "./controllers/sessions.js";
import { signinSchema } from "./schema/signin.js";
import { withSession } from "./middleware/session.js";
import { mayUseShelf, requireGuest, requireUser } from "./middleware/access.js";
import { withBook, withShelf } from "./middleware/resources.js";

const validateNewShelf = withValidation(shelfSchema, myShelvesController);
const validateEditShelf = withValidation(shelfSchema, shelfController);
const validateNewUser = withValidation(userSchema, signupFormController);
const validateSignin = withValidation(signinSchema, signinFormController);

const mustBeSignedIn = requireUser(signinRedirectController);
const mustBeSignedOut = requireGuest(alreadySignedInRedirectController);

const app = new Router();

app.use(withLogs);
app.use(withErrors);
app.use(withSession);

app.get("/assets/*", staticController);
app.get("/", homeController);
app.get("/books", booksController);
app.get("/books/:bookId", withBook, bookController);
app.get("/signup", mustBeSignedOut, signupFormController);
app.post("/signup", mustBeSignedOut, validateNewUser, createUserController);
app.get("/signin", mustBeSignedOut, signinFormController);
app.post("/signin", mustBeSignedOut, validateSignin, signinController);
app.post("/signout", signoutController);
app.get("/shelves", mustBeSignedIn, myShelvesController);
app.post("/shelves", mustBeSignedIn, validateNewShelf, createShelfController);
app.get("/shelves/:shelfId", withShelf, shelfController);
app.post("/shelves/:shelfId/edit", mustBeSignedIn, withShelf, mayUseShelf, validateEditShelf, editShelfController);
app.post("/shelves/:shelfId/delete", mustBeSignedIn, withShelf, mayUseShelf, deleteShelfController);
app.post("/shelves/:shelfId/books/:bookId/add", mustBeSignedIn, withShelf, withBook, mayUseShelf, addBookController);
app.post("/shelves/:shelfId/books/:bookId/delete", mustBeSignedIn, withShelf, withBook, mayUseShelf, removeBookController);

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

- On renaming and deleting a shelf, it goes straight after `withShelf`.
- On adding and removing a book, it goes after `withShelf` and `withBook`, so a book that doesn't exist is still not found.
- The shelf's page, `GET /shelves/:shelfId`, stays open to everyone, because `shelfController` already picks the right view.

Each route now reads from left to right: are you signed in, which shelf is it, and is it yours?

Sign in as **testUser2**, open [http://localhost:8000/shelves/4][shelf 4], and try the [devtools trick][forged form] again.
This time you get the not found page, and the terminal says why.

```plaintext {linenos=false}

============================
POST /shelves/1/delete
[session] {
  id: "80d0e540-f934-4b14-a1bd-9e8ddc983bb8",
  userId: 2,
  expires: "2026-11-25T15:49:02.948Z"
}
[user] { id: 2, username: "testUser2" }
[error] POST /shelves/1/delete is not permitted
RESPONSE 404
============================
```

Open [http://localhost:8000/shelves/1][shelf 1], and **classics** is still there.
As **testUser1**, you can still rename it, delete it, and add and remove its books.

{{<figure src="images/not-permitted.webp" alt="The not found page, shown after testUser2's forged request to delete the classics shelf" caption="A forged delete of someone else's shelf is not found.">}}

### Not found, or forbidden? {#not-found-or-forbidden}

**testUser2** can see **classics**, so why do they get `404 Not Found` rather than `403 Forbidden`?

We will use one rule for the rest of the module.

- When the answer depends on **what belongs to whom**, we answer `404 Not Found`. For **testUser2**, the routes that change **testUser1**'s shelves simply don't exist. This also gives nothing away: later, some shelves will be private, and a `403` would tell a stranger that a private shelf was there.
- When the answer depends only on **your status**, such as whether you are an admin, we answer [`403 Forbidden`][403]. The thing is there, and you may know about it, but you aren't allowed to do that.

Our `require` guards are about your status too, but they don't need a `403`.
A visitor who isn't signed in is sent to sign in, and a signed-in user is sent home, because each of them has something sensible to do next.

## The permissions pattern {#pattern}

We now have every part of a pattern for permissions that we will use for the rest of the module.
The names of our middleware tell us what kind they are.

- `withShelf` and `withBook` *load* something.
- `requireUser` and `requireGuest` make guards that are about *you* alone.
- `mayUseShelf` is a guard about you *and* a thing.

The pattern has four parts.

1. **Load the thing.** A `with` middleware, such as `withShelf`, loads the thing the route is about and puts it on `ctx.data`. If it doesn't exist, the route is not found.
2. **Ask one question.** Is this user allowed to use this thing? For a shelf, the question is `user?.id === shelf.ownerId`.
3. **Use the answer twice.** In the route, a `may` guard, `mayUseShelf`, turns away anyone the answer is no for. In the view, the same question decides whether to show the controls, so nobody is offered a button that won't work.
4. **Filter lists in SQL.** A list, such as My shelves or the shelves on a book's page, never fetches rows the user can't use, so nothing needs to be hidden.

There is one more rule about where each guard goes.
A `require` guard, such as `mustBeSignedIn`, only looks at `ctx.user`, so it goes first.
A `may` guard needs the thing as well, so it goes after the `with` middleware that loads it.

We are asking the same question in two places, `shelfController` and `mayUseShelf`, and writing it out each time.
That is fine for one question.
When we have more questions to ask, we will give each one a name of its own, and make the guards from them.

> Both halves matter.
> The guard on its own would leave users pressing buttons that only give them a not found page.
> The view on its own is just hiding, and we have seen how easy it is to get past that.

## Conclusion

Every shelf now has an owner, and the database makes sure of it.
My shelves and a book's page only fetch your own shelves, filtered in SQL.
A shelf's page shows its owner the controls, and shows everyone else a read-only view.
We saw that hiding the buttons is not enough, and added `mayUseShelf` to every route that changes a shelf, so a forged form gets the not found page.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="64-66"}
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
│   │   ├── sessions.js
│   │   ├── shelfBooks.js
│   │   ├── shelves.js
│   │   ├── static.js
│   │   └── users.js
│   ├── middleware/
│   │   ├── access.js
│   │   ├── errors.js
│   │   ├── logging.js
│   │   ├── resources.js
│   │   ├── session.js
│   │   └── validation.js
│   ├── models/
│   │   ├── books.js
│   │   ├── db.js
│   │   ├── sessions.js
│   │   ├── shelfBooks.js
│   │   ├── shelves.js
│   │   └── users.js
│   ├── schema/
│   │   ├── shelf.js
│   │   ├── signin.js
│   │   └── user.js
│   ├── tools/
│   │   ├── authentication.js
│   │   ├── errors.js
│   │   ├── flash.js
│   │   ├── hash.js
│   │   ├── html.js
│   │   ├── redirect.js
│   │   ├── render.js
│   │   ├── router.js
│   │   ├── validation.js
│   │   └── validators.js
│   ├── views/
│   │   ├── partials/
│   │   │   ├── bookCard.js
│   │   │   ├── cardGrid.js
│   │   │   ├── postButton.js
│   │   │   ├── searchForm.js
│   │   │   └── shelfTitleForm.js
│   │   ├── book.js
│   │   ├── books.js
│   │   ├── error.js
│   │   ├── home.js
│   │   ├── layout.js
│   │   ├── myShelf.js
│   │   ├── myShelves.js
│   │   ├── publicShelf.js
│   │   ├── signin.js
│   │   └── signup.js
│   └── main.js
├── tasks/
│   ├── db-create.js
│   ├── db-seed.js
│   └── server.js
├── .gitignore
├── deno.json
└── deno.lock
```

**myShelf.js** and **myShelves.js** are the old **shelf.js** and **shelves.js**, renamed.

## Challenge: Whose shelf?

These are questions to investigate, not changes to keep.

- As **testUser1**, make a new shelf. What is its address? Sign in as **testUser2** and visit it. What can you do there?
- As **testUser2**, use the devtools trick to point a **remove** button on **sci-fi** at one of the books on [http://localhost:8000/shelves/1][shelf 1]. What do you get? Now point it at a book that isn't on any shelf. Does anything change?
- On the rename route, move `mayUseShelf` in front of `withShelf`. What happens when **testUser1** renames **classics**, and why?
- `mustBeSignedIn` always runs before `mayUseShelf`, so `ctx.user` is always there. Why might the `?.` in `ctx.user?.id` still be worth keeping?

<details>
<summary>Why order matters</summary>

`mayUseShelf` reads `ctx.data.shelf.ownerId`.
In front of `withShelf`, nothing has loaded the shelf yet, so `ctx.data.shelf` is `undefined`, and reading `.ownerId` from it throws a `TypeError`.
That isn't a `NotFoundError`, so `withErrors` answers with the server error page, even for the shelf's owner.

The `?.` is a safety net.
If someone puts `mayUseShelf` on a route without `mustBeSignedIn`, a visitor who isn't signed in still gets the not found page, rather than a server error.

</details>

> If you tried any changes, undo them, and run `deno task db:reset`, so that your code and data match the starting point of the next exercise.

[previous exercise]: owners-and-guards/guards/
[must be signed in]: owners-and-guards/guards/#require-user
[shelves]: http://localhost:8000/shelves
[book 1]: http://localhost:8000/books/1
[shelf 1]: http://localhost:8000/shelves/1
[shelf 4]: http://localhost:8000/shelves/4
[forged form]: #forged-form
[not found error]: user-input-and-security/pagination/#invalid-page
[403]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/403

---
type: exercise
title: Avatars
weight: 2
author: Dr Graeme Stuart
---

In this exercise we will give every user a picture, called an *avatar*.
We will store pictures in the database, show them on profiles and beside users' names, and then let each user upload their own.

<!--more-->

> You should have a profile page for every user, with a **profile** link in the nav, from the [previous exercise][previous exercise].
> Your database should hold the seeded users and their shelves. If you have been experimenting, run `deno task db:reset` to get back to the seed data.
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

So far, everything in our database has been text and numbers.
A picture is a file, and the browser has to send it to us, and we have to send it back.
By the end of this exercise you should be able to:

- Store a file's bytes in the database, and read them back as a `File`
- Serve a file from a route, so that an `<img>` element can point at it
- Make sure the browser fetches a new picture rather than an old one from its cache
- Send a file from a form with `multipart/form-data`, and read it on the server
- Check an uploaded file's type and size before we keep it
- Guard a route that changes a user, just as we guarded the routes that change a shelf

## Somewhere to keep pictures {#files-table}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Profiles are going to show a picture of each user, so the database needs somewhere to keep pictures.
A picture is a file: some bytes, with a name and a type, such as `image/webp`.

Update **tasks/db-create.js** to add a files table, and give users an `avatarId` column.

**tasks/db-create.js**
```JavaScript{hl_lines="8-17 24-25"}
import { db } from "../server/models/db.js";

db.exec("DROP TABLE IF EXISTS sessions");
db.exec("DROP TABLE IF EXISTS shelfBooks");
db.exec("DROP TABLE IF EXISTS shelves");
db.exec("DROP TABLE IF EXISTS books");
db.exec("DROP TABLE IF EXISTS users");
db.exec("DROP TABLE IF EXISTS files");

db.exec(`
    CREATE TABLE files (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        bytes BLOB NOT NULL,
        type TEXT NOT NULL
    )
`);

// Placeholder: this will need more columns later
db.exec(`
    CREATE TABLE users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT UNIQUE NOT NULL,
        hashedPassword TEXT NOT NULL,
        avatarId INTEGER REFERENCES files(id) ON DELETE SET NULL
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

The files table has one row for each file.

- `name` is the file's name, such as **testUser1.webp**.
- `bytes` is the file itself. A [`BLOB`][sqlite blob] column holds bytes exactly as they are given, rather than as text.
- `type` is the file's [media type][media types], such as `image/webp`, which tells the browser what kind of file it is.

`avatarId` is a foreign key, like `ownerId` on the shelves table, but this one is allowed to be empty, because a user doesn't have to have an avatar.
`ON DELETE SET NULL` keeps our data honest the other way round from [`ON DELETE CASCADE`][owner id].
Deleting a file leaves its user without an avatar, rather than deleting the user, and no user is ever left pointing at a file that has gone.

The users table refers to the files table, so the files table is created first.
It's dropped last, after the users that refer to it.

> The users table still has its placeholder, because users will get more columns later.
> The shelves table's placeholder hasn't changed either.

We need a model for files.
Create **server/models/files.js**.

**server/models/files.js**
```JavaScript{hl_lines="1-11"}
import { db } from "./db.js";

export async function createFile(file) {
    const bytes = await file.bytes();
    const { id } = db.prepare(`
        INSERT INTO files(name, bytes, type)
        VALUES (:name, :bytes, :type)
        RETURNING id
    `).get({ name: file.name, bytes, type: file.type });
    return id;
}
```

`createFile` takes a [`File`][file], stores it, and returns the new row's `id`.
A `File` doesn't hand over its bytes straight away.
Its [`bytes`][blob bytes] method returns a promise, so `createFile` is `async`, and it `await`s the bytes before it stores them, just as we [awaited][async] the next handler in our middleware.

Each user needs to point at their avatar.
Add `updateUserAvatarById` to **server/models/users.js**.

**server/models/users.js**
```JavaScript{hl_lines="15-20"}
import { hashPassword } from "../tools/hash.js";
import { db } from "./db.js";

export function getUserByUsername(username) {
    return db.prepare(
        `SELECT * FROM users WHERE username=:username`,
    ).get({ username });
}

// Placeholder: this will need to change when users get more columns
export function getUserById(id) {
    return db.prepare(`SELECT id, username FROM users WHERE id=:id`).get({ id });
}

export function updateUserAvatarById(id, avatarId) {
    return db.prepare(
        `UPDATE users SET avatarId=:avatarId WHERE id=:id`,
    ).run({ id, avatarId });
}

// Placeholder: this will need to change when users get more columns
export async function createUser({ username, password }) {
    const hashedPassword = await hashPassword(password);
    return db.prepare(`
        INSERT INTO users(username, hashedPassword)
        VALUES (:username, :hashedPassword)
        RETURNING *
    `).get({ username, hashedPassword });
}

export function usernameExists(username) {
    return Boolean(
        db.prepare(`
            SELECT 1 FROM users
            WHERE username=:username
        `).get({ username }),
    );
}
```

The seed users need some pictures.
Make a **seed-avatars** folder in the **db** folder.
Save {{<a href="files/testUser1.webp" download="testUser1.webp" >}}this picture{{< /a >}} as **testUser1.webp** and {{<a href="files/testUser2.webp" download="testUser2.webp" >}}this picture{{< /a >}} as **testUser2.webp**, both in **db/seed-avatars**.

Update **db/seed.json** to give each user an avatar.

**db/seed.json**
```json{hl_lines="6 39"}
{
    "users": [
        {
            "username": "testUser1",
            "password": "testUser1!",
            "avatar": "testUser1.webp",
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
            "avatar": "testUser2.webp",
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

Then update **tasks/db-seed.js** to store them.

**tasks/db-seed.js**
```JavaScript{hl_lines="3-4 17-20"}
import seed from "../db/seed.json" with { type: "json" };
import books from "../db/books.json" with { type: "json" };
import { createUser, updateUserAvatarById } from "../server/models/users.js";
import { createFile } from "../server/models/files.js";
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
    const path = new URL(`../db/seed-avatars/${user.avatar}`, import.meta.url);
    const file = new File([await Deno.readFile(path)], user.avatar, { type: "image/webp" });
    updateUserAvatarById(userId, await createFile(file));

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

For each user, after creating them, we read their picture and store it.

- [`import.meta.url`][import meta] is the address of **db-seed.js** itself, so `new URL("../db/seed-avatars/testUser1.webp", import.meta.url)` finds the picture relative to this file, wherever the task is run from.
- [`Deno.readFile`][deno read file] reads the whole file as bytes. It returns a promise, so we `await` it.
- [`new File(...)`][file constructor] wraps the bytes in a `File`, with a name and a type, which is just what `createFile` expects.
- `createFile` stores it and gives us its `id`, and `updateUserAvatarById` points the user at it.

The tables have changed, so rebuild the database.

```bash {linenos=false}
deno task db:reset
```

Nothing in the app uses the pictures yet, but we can look in the database to see them.
Run this in the terminal, from your project folder.

```bash {linenos=false}
deno eval 'import { db } from "./server/models/db.js";
console.table(db.prepare("SELECT id, name, type, length(bytes) AS size FROM files").all());
console.table(db.prepare("SELECT id, username, avatarId FROM users").all())'
```

```plaintext {linenos=false}
┌───────┬────┬──────────────────┬──────────────┬──────┐
│ (idx) │ id │ name             │ type         │ size │
├───────┼────┼──────────────────┼──────────────┼──────┤
│     0 │ 1  │ "testUser1.webp" │ "image/webp" │ 9668 │
│     1 │ 2  │ "testUser2.webp" │ "image/webp" │ 7852 │
└───────┴────┴──────────────────┴──────────────┴──────┘
┌───────┬────┬─────────────┬──────────┐
│ (idx) │ id │ username    │ avatarId │
├───────┼────┼─────────────┼──────────┤
│     0 │ 1  │ "testUser1" │ 1        │
│     1 │ 2  │ "testUser2" │ 2        │
└───────┴────┴─────────────┴──────────┘
```

> [`deno eval`][deno eval] runs a little JavaScript without a file, here using our own `db` module.
> We ask for `length(bytes)` rather than the bytes themselves, which would fill the terminal.
> If you have the `sqlite3` command line installed, `sqlite3 db/app.db "SELECT id, name, type, length(bytes) FROM files"` shows the same thing.

Each user's `avatarId` is the `id` of their picture in the files table.

## Serving an avatar {#serve-avatar}

The pictures are in the database, but nothing can see them.
An `<img>` element needs an address to point at, so each user's avatar will have its own: `/users/:userId/avatar`.

First, we need to get a file back out of the database.
Add `getFileById` to **server/models/files.js**.

**server/models/files.js**
```JavaScript{hl_lines="12-19"}
import { db } from "./db.js";

export async function createFile(file) {
    const bytes = await file.bytes();
    const { id } = db.prepare(`
        INSERT INTO files(name, bytes, type)
        VALUES (:name, :bytes, :type)
        RETURNING id
    `).get({ name: file.name, bytes, type: file.type });
    return id;
}

export function getFileById(id) {
    const row = db.prepare(
        `SELECT name, bytes, type FROM files WHERE id=:id`,
    ).get({ id });
    if (!row) return null;
    return new File([row.bytes], row.name, { type: row.type });
}
```

`getFileById` turns a files row back into a `File`, with its bytes, name and type.
If there is no such row, it returns `null`.

The route will load the user, and we need to know their `avatarId`.
Update `getUserById` in **server/models/users.js** to return it.

**server/models/users.js**
```JavaScript{hl_lines="12"}
import { hashPassword } from "../tools/hash.js";
import { db } from "./db.js";

export function getUserByUsername(username) {
    return db.prepare(
        `SELECT * FROM users WHERE username=:username`,
    ).get({ username });
}

// Placeholder: this will need to change when users get more columns
export function getUserById(id) {
    return db.prepare(`SELECT id, username, avatarId FROM users WHERE id=:id`).get({ id });
}

export function updateUserAvatarById(id, avatarId) {
    return db.prepare(
        `UPDATE users SET avatarId=:avatarId WHERE id=:id`,
    ).run({ id, avatarId });
}

// Placeholder: this will need to change when users get more columns
export async function createUser({ username, password }) {
    const hashedPassword = await hashPassword(password);
    return db.prepare(`
        INSERT INTO users(username, hashedPassword)
        VALUES (:username, :hashedPassword)
        RETURNING *
    `).get({ username, hashedPassword });
}

export function usernameExists(username) {
    return Boolean(
        db.prepare(`
            SELECT 1 FROM users
            WHERE username=:username
        `).get({ username }),
    );
}
```

> The placeholder on `getUserById` stays, because users will get more columns later.

Add `avatarController` to **server/controllers/users.js**.

**server/controllers/users.js**
```JavaScript{hl_lines="4 6 31-39"}
import { createUser } from "../models/users.js";
import { getShelvesByOwnerId } from "../models/shelves.js";
import { addThumbnails } from "../models/shelfBooks.js";
import { getFileById } from "../models/files.js";
import { signin } from "../tools/authentication.js";
import { NotFoundError } from "../tools/errors.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { signupView } from "../views/signup.js";
import { userView } from "../views/user.js";

export function signupFormController(ctx) {
    ctx.data.title = "Sign up";
    return render(signupView, ctx);
}

export async function createUserController(ctx) {
    const { validated } = ctx;
    const user = await createUser(validated);
    signin(user.id, ctx);
    ctx.flash = `Welcome, ${user.username}`;
    return redirect("/", ctx);
}

export function userController(ctx) {
    const { data: { targetUser } } = ctx;
    ctx.data.shelves = addThumbnails(getShelvesByOwnerId(targetUser.id));
    ctx.data.title = targetUser.username;
    return render(userView, ctx);
}

export function avatarController(ctx) {
    const { data: { targetUser } } = ctx;
    const avatarFile = targetUser.avatarId && getFileById(targetUser.avatarId);
    if (!avatarFile) {
        throw new NotFoundError(`No avatar for user ${targetUser.id}`);
    }
    return new Response(avatarFile);
}
```

- `targetUser.avatarId && getFileById(targetUser.avatarId)` only looks for a file if the user has an `avatarId`. If they don't, `avatarFile` is `null`, and we don't ask the database at all.
- With no file, we throw a `NotFoundError`, so the not found page answers.
- Otherwise, the file *is* the body of the response. A `File` is a kind of [`Blob`][blob], and when a [`Response`][response] is given a `Blob`, it sends its bytes with its type as the `Content-Type`. We don't need to set any headers ourselves.

Update **server/main.js** to add the route.

**server/main.js**
```JavaScript{hl_lines="17-22 67"}
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
import {
    avatarController,
    createUserController,
    signupFormController,
    userController,
} from "./controllers/users.js";
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
import { withBook, withShelf, withTargetUser } from "./middleware/resources.js";

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
app.get("/users/:userId", withTargetUser, userController);
app.get("/users/:userId/avatar", withTargetUser, avatarController);

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

`withTargetUser` loads the user, just as it does for the profile page, so an unknown user is not found.

Open [http://localhost:8000/users/1/avatar][avatar 1] and [http://localhost:8000/users/2/avatar][avatar 2] to see **testUser1**'s and **testUser2**'s pictures.
[http://localhost:8000/users/99/avatar][avatar 99] is not found.

> Open the developer tools (**F12**) on the **network** panel, and refresh [http://localhost:8000/users/1/avatar][avatar 1].
> Click on the request, and look at the response headers.
> The `Content-Type` is `image/webp`, the type we stored in the database.

## Avatars on profiles {#profile-avatar}

Now we can put the picture on the profile page.
Not every user has an avatar, though.
Anyone who signs up starts without one, so we need a default picture for them.

Save {{<a href="files/default-avatar.svg" download="default-avatar.svg" >}}this default avatar{{< /a >}} as **default-avatar.svg** in the **public** folder.
It's a grey outline of a person.

Update **server/views/user.js**.

**server/views/user.js**
```JavaScript{hl_lines="5-6 9-12 19"}
import { html } from "../tools/html.js";
import { cardGrid } from "./partials/cardGrid.js";
import { shelfCard } from "./partials/shelfCard.js";

const DEFAULT_AVATAR = "/assets/default-avatar.svg";

export function userView(ctx) {
    const { data: { targetUser, shelves } } = ctx;
    // The avatarId in the query string changes with the picture, so a new one is never cached
    const avatarSrc = targetUser.avatarId
        ? `/users/${targetUser.id}/avatar?v=${targetUser.avatarId}`
        : DEFAULT_AVATAR;

    return html`
        <section>
            <header>
                <h1>${targetUser.username}</h1>
            </header>
            <img class="avatar" src="${avatarSrc}" alt="Avatar for '${targetUser.username}'">
            <h2>Shelves</h2>
            ${cardGrid(shelves.map((shelf) => shelfCard(shelf)), "No shelves yet")}
        </section>
    `;
}
```

- With an `avatarId`, the picture comes from the user's avatar route. Without one, it's the default avatar, which is served with the rest of our static files under `/assets`.
- The picture has `Avatar for` and the username as its alt text, for anyone who can't see it.

The address ends with `?v=` and the `avatarId`, which our route ignores.
It's there for the browser.
A browser may keep a copy of a picture it has already fetched, in its *cache*, and show that copy the next time it sees the same address.
When a user changes their picture, the new one will get a new `id`, so the address changes too, and the browser has to fetch it.

Add an `.avatar` rule to the end of **public/styles.css**.

**public/styles.css**
```css {linenostart="284"}
/* A square picture with 50% rounded corners is a circle */
.avatar {
    inline-size: 8rem;
    aspect-ratio: 1;
    border-radius: 50%;
    object-fit: cover;
    vertical-align: middle; /* lines up with the middle of the text beside it */
    background: gray; /* grey shows behind a transparent picture, or until it loads */
}
```

A square picture with corners rounded by half its width is a circle.
[`object-fit: cover`][object fit] crops a picture that isn't square, rather than squashing it.

Open [http://localhost:8000/users/1][user 1].
**testUser1**'s picture is under their name, in a circle.
Visit [http://localhost:8000/users/2][user 2] signed out, and **testUser2**'s picture is there too.

{{<figure src="images/profile-avatar.webp" alt="testUser2's profile, with their avatar in a circle under the heading testUser2, and their shelves below" caption="An avatar on testUser2's profile.">}}

## An avatar partial {#avatar-image}

Avatars are about to appear beside users' names all over the site.
Before we use one anywhere else, the `<img>` should be a partial.

Create **server/views/partials/avatarImage.js**.

**server/views/partials/avatarImage.js**
```JavaScript{hl_lines="1-8"}
import { html } from "../../tools/html.js";

const DEFAULT_AVATAR = "/assets/default-avatar.svg";

export function avatarImage({ id, avatarId }, alt = "") {
    const src = avatarId ? `/users/${id}/avatar?v=${avatarId}` : DEFAULT_AVATAR;
    return html`<img class="avatar" src="${src}" alt="${alt}">`;
}
```

`avatarImage` takes anything with an `id` and an `avatarId`, so a user will do.
It takes them apart in its parameter list, with `{ id, avatarId }`, just as we take `ctx` apart in our controllers.
It picks the avatar route or the default picture, as the profile did.

The alt text is empty unless you pass some.
Beside a name, the picture adds nothing for a screen reader, which reads the name anyway.

Update **server/views/user.js** to use it.

**server/views/user.js**
```JavaScript{hl_lines="2 14"}
import { html } from "../tools/html.js";
import { avatarImage } from "./partials/avatarImage.js";
import { cardGrid } from "./partials/cardGrid.js";
import { shelfCard } from "./partials/shelfCard.js";

export function userView(ctx) {
    const { data: { targetUser, shelves } } = ctx;

    return html`
        <section>
            <header>
                <h1>${targetUser.username}</h1>
            </header>
            ${avatarImage(targetUser, `Avatar for '${targetUser.username}'`)}
            <h2>Shelves</h2>
            ${cardGrid(shelves.map((shelf) => shelfCard(shelf)), "No shelves yet")}
        </section>
    `;
}
```

`DEFAULT_AVATAR` and the code that picked the address have moved into the partial.
The profile passes `Avatar for` and the username as the alt text, as before.

Refresh [http://localhost:8000/users/1][user 1].
It looks exactly as it did.

## Avatars beside names {#avatars-beside-names}

A user's picture can now go wherever their name links to their profile: the **profile** link in the nav, the owner on each shelf card on the home page, and the owner on a shelf's page.

An avatar inside a link should be the size of the text beside it.
Add an `a .avatar` rule to the end of **public/styles.css**.

**public/styles.css**
```css {linenostart="294"}
/* Inside a link, an avatar is one line of text tall */
a .avatar {
    inline-size: 1lh;
    margin-inline-end: 0.5ch;
}
```

`1lh` is the height of one line of text, and `0.5ch` is half the width of a character, a small gap before the name.

The signed-in user comes from `getUserById`, which already returns their `avatarId`.
Update the `authLinks` function in **server/views/layout.js**.

**server/views/layout.js**
```JavaScript{hl_lines="2 54"}
import { html } from "../tools/html.js";
import { avatarImage } from "./partials/avatarImage.js";
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
        ${navLink(`/users/${user.id}`, html`${avatarImage(user)}profile`, path)}
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

The link's content is now the avatar followed by the word **profile**, in one `html` template.

The home page's shelves come from `getShelves`, which needs to find each owner's `avatarId` too.
Update **server/models/shelves.js**.

**server/models/shelves.js**
```JavaScript{hl_lines="30"}
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

// All shelves, with the owner's username for display.
// Placeholder: this will need to change when shelves can be public
export function getShelves() {
    return db.prepare(`
        SELECT shelves.id, title, ownerId, username, avatarId,
            (SELECT count(*) FROM shelfBooks WHERE shelfId = shelves.id) AS bookCount
        FROM shelves JOIN users ON users.id = shelves.ownerId
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

> The placeholders haven't changed.

Update `ownerLink` in **server/views/home.js**.

**server/views/home.js**
```JavaScript{hl_lines="3 28-30"}
import { html } from "../tools/html.js";
import { cardGrid } from "./partials/cardGrid.js";
import { avatarImage } from "./partials/avatarImage.js";
import { shelfCard } from "./partials/shelfCard.js";
import { searchForm } from "./partials/searchForm.js";

export function homeView(ctx) {
    const { data: { shelves } } = ctx;

    return html`
        <section>
            <header>
                <h1>Welcome to Bookshelves</h1>
                <p>Organise your books with bookshelves.</p>
            </header>
            <h2>Find a book</h2>
            ${searchForm()}
            <h2>Everyone's shelves</h2>
            ${cardGrid(
                shelves.map((shelf) => shelfCard(shelf, ownerLink(shelf))),
                "Nobody has made a shelf yet",
            )}
        </section>
    `;
}

function ownerLink(shelf) {
    return html`<a href="/users/${shelf.ownerId}">${
        avatarImage({ id: shelf.ownerId, avatarId: shelf.avatarId })
    }${shelf.username}</a>`;
}
```

A shelf isn't a user, so we can't pass it to `avatarImage` as it is.
Its `id` is the shelf's, not the owner's.
Instead, we build what `avatarImage` needs from the shelf's `ownerId` and `avatarId`.

A shelf's owner comes from `getUserById` too.
Update **server/views/publicShelf.js**.

**server/views/publicShelf.js**
```JavaScript{hl_lines="2 13-14"}
import { html } from "../tools/html.js";
import { avatarImage } from "./partials/avatarImage.js";
import { bookCard } from "./partials/bookCard.js";
import { cardGrid } from "./partials/cardGrid.js";

export function publicShelfView(ctx) {
    const { data: { shelf, books, owner } } = ctx;

    return html`
        <section>
            <header>
                <h1>${shelf.title}</h1>
                <p>A shelf by <a href="/users/${owner.id}">${avatarImage(owner)}${owner
                    .username}</a></p>
            </header>
            ${cardGrid(books.map((book) => bookCard(book)), "No books on this shelf yet")}
            <p><a href="/">Back to home</a></p>
        </section>
    `;
}
```

> `owner.username` is split over two lines to keep the line short.
> It means exactly the same thing.

Refresh [the home page][home] while signed in as **testUser1** (the password is `testUser1!`).
Your avatar is beside **profile** in the nav, and each shelf card has its owner's picture beside their name.
Open **sci-fi**, [http://localhost:8000/shelves/4][shelf 4], and **testUser2**'s picture is beside their name.

{{<figure src="images/avatars-beside-names.webp" alt="The home page for testUser1, with a small avatar beside profile in the nav, and testUser1's small avatar on the shelf cards" caption="Avatars beside users' names.">}}

## An upload form {#upload-form}

Everyone who signs up gets the default picture, and there's no way to change it.
Your own profile should have a form for uploading a new one.

Update **server/views/user.js**.

**server/views/user.js**
```JavaScript{hl_lines="7 18-30"}
import { html } from "../tools/html.js";
import { avatarImage } from "./partials/avatarImage.js";
import { cardGrid } from "./partials/cardGrid.js";
import { shelfCard } from "./partials/shelfCard.js";

export function userView(ctx) {
    const { data: { targetUser, shelves }, user: currentUser } = ctx;

    return html`
        <section>
            <header>
                <h1>${targetUser.username}</h1>
            </header>
            ${avatarImage(targetUser, `Avatar for '${targetUser.username}'`)}
            <h2>Shelves</h2>
            ${cardGrid(shelves.map((shelf) => shelfCard(shelf)), "No shelves yet")}
        </section>
        ${currentUser?.id === targetUser.id ? html`<aside>${avatarForm(targetUser)}</aside>` : ""}
    `;
}

function avatarForm(user) {
    return html`
        <h2>Change avatar</h2>
        <form class="secondary" method="POST" action="/users/${user.id}/avatar">
            <label for="avatar">Avatar</label>
            <input id="avatar" type="file" name="avatar" accept="image/png,image/jpeg,image/webp"
                required>
            <button>upload avatar</button>
        </form>
    `;
}
```

- `userView` takes the signed-in user from `ctx` as well, and calls them `currentUser`, to keep them apart from `targetUser`.
- If they are the same person, the page gets an `<aside>` with the form. On anyone else's profile, and for a visitor who isn't signed in, there is no form at all.
- The form posts to `/users/:userId/avatar`, the same address that serves the avatar, but with `POST` rather than `GET`.

The form has an [`<input type="file">`][input file], which lets the user pick a file.

- [`accept`][accept] lists the types we want, so the browser's file picker offers PNG, JPEG and WebP pictures. It's only a suggestion: the picker can usually be switched to show all files.
- `required` means the browser won't send the form until a file has been picked.

This is the second part of our [permissions pattern][pattern] again.
The view asks a question, *is this your profile?*, and only shows the form if the answer is yes.
We have seen that [hiding is not enough][forged form], so we will come back to the route.

For now, the controller is a stub, so we can see what arrives.
Add `uploadAvatarController` to **server/controllers/users.js**.

**server/controllers/users.js**
```JavaScript{hl_lines="40-47"}
import { createUser } from "../models/users.js";
import { getShelvesByOwnerId } from "../models/shelves.js";
import { addThumbnails } from "../models/shelfBooks.js";
import { getFileById } from "../models/files.js";
import { signin } from "../tools/authentication.js";
import { NotFoundError } from "../tools/errors.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { signupView } from "../views/signup.js";
import { userView } from "../views/user.js";

export function signupFormController(ctx) {
    ctx.data.title = "Sign up";
    return render(signupView, ctx);
}

export async function createUserController(ctx) {
    const { validated } = ctx;
    const user = await createUser(validated);
    signin(user.id, ctx);
    ctx.flash = `Welcome, ${user.username}`;
    return redirect("/", ctx);
}

export function userController(ctx) {
    const { data: { targetUser } } = ctx;
    ctx.data.shelves = addThumbnails(getShelvesByOwnerId(targetUser.id));
    ctx.data.title = targetUser.username;
    return render(userView, ctx);
}

export function avatarController(ctx) {
    const { data: { targetUser } } = ctx;
    const avatarFile = targetUser.avatarId && getFileById(targetUser.avatarId);
    if (!avatarFile) {
        throw new NotFoundError(`No avatar for user ${targetUser.id}`);
    }
    return new Response(avatarFile);
}

export async function uploadAvatarController(ctx) {
    const { request, data: { targetUser } } = ctx;
    const formData = await request.formData();
    console.log(formData);
    ctx.flash = `Ignored avatar upload for '${targetUser.username}'`;
    return redirect(`/users/${targetUser.id}`, ctx);
}
```

`uploadAvatarController` reads the form data from the request, with [`request.formData()`][form data], and logs it in the terminal.
Then it flashes a message and sends you back to the profile.

Update **server/main.js** to add the route.

**server/main.js**
```JavaScript{hl_lines="21 69"}
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
import {
    avatarController,
    createUserController,
    signupFormController,
    uploadAvatarController,
    userController,
} from "./controllers/users.js";
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
import { withBook, withShelf, withTargetUser } from "./middleware/resources.js";

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
app.get("/users/:userId", withTargetUser, userController);
app.get("/users/:userId/avatar", withTargetUser, avatarController);
app.post("/users/:userId/avatar", mustBeSignedIn, withTargetUser, uploadAvatarController);

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

Only a signed-in user can upload, so `mustBeSignedIn` goes first, then `withTargetUser` loads the user the upload is for.

As **testUser1**, open [your profile][user 1].
You see the **Change avatar** form.
Click **Browse...** (or **Choose File**, depending on your browser), pick **testUser2.webp** from your **db/seed-avatars** folder, and press **upload avatar**.

You land back on your profile, with the message **Ignored avatar upload for 'testUser1'**, and the terminal shows what arrived.

```plaintext {linenos=false}

============================
POST /users/1/avatar
FormData { avatar: "testUser2.webp" }
[session] {
  id: "5032ec63-e6d6-4b9c-a21b-0f5cfdbc020f",
  userId: 1,
  expires: "2026-11-25T15:57:46.649Z"
}
[user] { id: 1, username: "testUser1", avatarId: 1 }
[flash sent] Ignored avatar upload for 'testUser1'
RESPONSE 303
============================
```

The form data has only the file's *name*, as a string.
The picture itself never left the browser.

{{<figure src="images/upload-form.webp" alt="testUser1's own profile, with the Change avatar form beside it, showing a file input and an upload avatar button" caption="Your own profile has a form for changing your avatar.">}}

Visit [**testUser2**'s profile][user 2], and there is no form.

## Sending the file {#multipart}

A form sends its fields as text unless it says otherwise.
That's fine for a title or a password, but a file isn't text, so only its name was sent.

With [`enctype="multipart/form-data"`][enctype], the browser sends each field as a separate *part*.
A file's part carries its bytes, its name and its type.

Update the form in **server/views/user.js**.

**server/views/user.js**
```JavaScript{hl_lines="25-26"}
import { html } from "../tools/html.js";
import { avatarImage } from "./partials/avatarImage.js";
import { cardGrid } from "./partials/cardGrid.js";
import { shelfCard } from "./partials/shelfCard.js";

export function userView(ctx) {
    const { data: { targetUser, shelves }, user: currentUser } = ctx;

    return html`
        <section>
            <header>
                <h1>${targetUser.username}</h1>
            </header>
            ${avatarImage(targetUser, `Avatar for '${targetUser.username}'`)}
            <h2>Shelves</h2>
            ${cardGrid(shelves.map((shelf) => shelfCard(shelf)), "No shelves yet")}
        </section>
        ${currentUser?.id === targetUser.id ? html`<aside>${avatarForm(targetUser)}</aside>` : ""}
    `;
}

function avatarForm(user) {
    return html`
        <h2>Change avatar</h2>
        <form class="secondary" method="POST" action="/users/${user.id}/avatar"
            enctype="multipart/form-data">
            <label for="avatar">Avatar</label>
            <input id="avatar" type="file" name="avatar" accept="image/png,image/jpeg,image/webp"
                required>
            <button>upload avatar</button>
        </form>
    `;
}
```

Refresh your profile, pick **testUser2.webp** again, and upload it.
This time the terminal shows a `File`.

```plaintext {linenos=false}

============================
POST /users/1/avatar
FormData {
  avatar: File { name: "testUser2.webp", size: 7852, type: "image/webp" }
}
[session] {
  id: "4fd1ded5-b99f-47fc-8543-1a8bc12627af",
  userId: 1,
  expires: "2026-11-25T15:57:49.699Z"
}
[user] { id: 1, username: "testUser1", avatarId: 1 }
[flash sent] Ignored avatar upload for 'testUser1'
RESPONSE 303
============================
```

`request.formData()` gives us the same kind of [`FormData`][form data object] as before, but the `avatar` field is now a `File`, with its name, its size in bytes and its type.
It's the same kind of `File` that our seed task made, so we already know how to store it.
The stub still ignores it.

## Saving the upload {#save-upload}

Now that the file arrives, the controller can keep it.

Update `uploadAvatarController` in **server/controllers/users.js**.

**server/controllers/users.js**
```JavaScript{hl_lines="1 4 44-46"}
import { createUser, updateUserAvatarById } from "../models/users.js";
import { getShelvesByOwnerId } from "../models/shelves.js";
import { addThumbnails } from "../models/shelfBooks.js";
import { createFile, getFileById } from "../models/files.js";
import { signin } from "../tools/authentication.js";
import { NotFoundError } from "../tools/errors.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { signupView } from "../views/signup.js";
import { userView } from "../views/user.js";

export function signupFormController(ctx) {
    ctx.data.title = "Sign up";
    return render(signupView, ctx);
}

export async function createUserController(ctx) {
    const { validated } = ctx;
    const user = await createUser(validated);
    signin(user.id, ctx);
    ctx.flash = `Welcome, ${user.username}`;
    return redirect("/", ctx);
}

export function userController(ctx) {
    const { data: { targetUser } } = ctx;
    ctx.data.shelves = addThumbnails(getShelvesByOwnerId(targetUser.id));
    ctx.data.title = targetUser.username;
    return render(userView, ctx);
}

export function avatarController(ctx) {
    const { data: { targetUser } } = ctx;
    const avatarFile = targetUser.avatarId && getFileById(targetUser.avatarId);
    if (!avatarFile) {
        throw new NotFoundError(`No avatar for user ${targetUser.id}`);
    }
    return new Response(avatarFile);
}

export async function uploadAvatarController(ctx) {
    const { request, data: { targetUser } } = ctx;
    const formData = await request.formData();
    const avatarId = await createFile(formData.get("avatar"));
    updateUserAvatarById(targetUser.id, avatarId);
    ctx.flash = "Avatar updated";
    return redirect(`/users/${targetUser.id}`, ctx);
}
```

- `formData.get("avatar")` is the `File`.
- `createFile` stores it, the same function the seed uses, and gives us the new row's `id`. It's `async`, so we `await` it.
- `updateUserAvatarById` points the user at the new file.
- The flash says **Avatar updated**, and the log has done its job, so it goes.

Upload **testUser2.webp** again.
You land on your profile with the message **Avatar updated**, and **testUser1** now has **testUser2**'s picture, on the profile and in the nav.
The address of the picture ends with `?v=3`, because it's the third file in the database, so the browser fetches it rather than showing the old one.
The message slides away after a few seconds; hover over the red strip to bring it back.

{{<figure src="images/avatar-updated.webp" alt="testUser1's profile after the upload, with the message Avatar updated and testUser2's picture in place of testUser1's" caption="The new avatar is saved.">}}

There is a problem, though.
Each upload adds a row to the files table, and the old picture stays there, with nothing pointing at it.
Upload a few times, and the database fills up with pictures nobody can see.

## One file per avatar {#delete-old}

A picture that has been replaced is no longer needed, so the controller should delete it.

Add `deleteFileById` to **server/models/files.js**.

**server/models/files.js**
```JavaScript{hl_lines="20-23"}
import { db } from "./db.js";

export async function createFile(file) {
    const bytes = await file.bytes();
    const { id } = db.prepare(`
        INSERT INTO files(name, bytes, type)
        VALUES (:name, :bytes, :type)
        RETURNING id
    `).get({ name: file.name, bytes, type: file.type });
    return id;
}

export function getFileById(id) {
    const row = db.prepare(
        `SELECT name, bytes, type FROM files WHERE id=:id`,
    ).get({ id });
    if (!row) return null;
    return new File([row.bytes], row.name, { type: row.type });
}

export function deleteFileById(id) {
    return db.prepare(`DELETE FROM files WHERE id=:id`).run({ id });
}
```

Update **server/controllers/users.js**.

**server/controllers/users.js**
```JavaScript{hl_lines="4 46-48 50-54"}
import { createUser, updateUserAvatarById } from "../models/users.js";
import { getShelvesByOwnerId } from "../models/shelves.js";
import { addThumbnails } from "../models/shelfBooks.js";
import { createFile, deleteFileById, getFileById } from "../models/files.js";
import { signin } from "../tools/authentication.js";
import { NotFoundError } from "../tools/errors.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { signupView } from "../views/signup.js";
import { userView } from "../views/user.js";

export function signupFormController(ctx) {
    ctx.data.title = "Sign up";
    return render(signupView, ctx);
}

export async function createUserController(ctx) {
    const { validated } = ctx;
    const user = await createUser(validated);
    signin(user.id, ctx);
    ctx.flash = `Welcome, ${user.username}`;
    return redirect("/", ctx);
}

export function userController(ctx) {
    const { data: { targetUser } } = ctx;
    ctx.data.shelves = addThumbnails(getShelvesByOwnerId(targetUser.id));
    ctx.data.title = targetUser.username;
    return render(userView, ctx);
}

export function avatarController(ctx) {
    const { data: { targetUser } } = ctx;
    const avatarFile = targetUser.avatarId && getFileById(targetUser.avatarId);
    if (!avatarFile) {
        throw new NotFoundError(`No avatar for user ${targetUser.id}`);
    }
    return new Response(avatarFile);
}

export async function uploadAvatarController(ctx) {
    const { request, data: { targetUser } } = ctx;
    const formData = await request.formData();
    const avatarId = await createFile(formData.get("avatar"));
    updateUserAvatarById(targetUser.id, avatarId);
    if (targetUser.avatarId) {
        deleteFileById(targetUser.avatarId);
    }
    ctx.flash = "Avatar updated";
    return redirect("/profile", ctx);
}

export function profileController(ctx) {
    return redirect(`/users/${ctx.user.id}`, ctx);
}
```

`withTargetUser` loaded the user *before* the upload, so `targetUser.avatarId` is still the old picture's `id`.
Once the user points at the new file, we delete the old one.
Users who had the default picture have no old file, so we only delete when there is one.
However many times you upload, there is one file per avatar.

The controller now redirects to `/profile`, and `profileController` sends you on to your own profile at `/users/:userId`.
`/profile` is an address for *my profile* that works whoever you are.

Update **server/main.js** to add the route.

**server/main.js**
```JavaScript{hl_lines="20 61"}
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
import {
    avatarController,
    createUserController,
    profileController,
    signupFormController,
    uploadAvatarController,
    userController,
} from "./controllers/users.js";
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
import { withBook, withShelf, withTargetUser } from "./middleware/resources.js";

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
app.get("/profile", mustBeSignedIn, profileController);
app.get("/shelves", mustBeSignedIn, myShelvesController);
app.post("/shelves", mustBeSignedIn, validateNewShelf, createShelfController);
app.get("/shelves/:shelfId", withShelf, shelfController);
app.post("/shelves/:shelfId/edit", mustBeSignedIn, withShelf, mayUseShelf, validateEditShelf, editShelfController);
app.post("/shelves/:shelfId/delete", mustBeSignedIn, withShelf, mayUseShelf, deleteShelfController);
app.post("/shelves/:shelfId/books/:bookId/add", mustBeSignedIn, withShelf, withBook, mayUseShelf, addBookController);
app.post("/shelves/:shelfId/books/:bookId/delete", mustBeSignedIn, withShelf, withBook, mayUseShelf, removeBookController);
app.get("/users/:userId", withTargetUser, userController);
app.get("/users/:userId/avatar", withTargetUser, avatarController);
app.post("/users/:userId/avatar", mustBeSignedIn, withTargetUser, uploadAvatarController);

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

`profileController` uses `ctx.user.id` without checking, so `mustBeSignedIn` goes in front of it.
A visitor who isn't signed in is sent to sign in.
That's a `require` guard doing its job: it's about [you alone][require user], not about any other user or shelf.

Run `deno task db:reset`, sign in as **testUser1** again, and upload **testUser2.webp**.
You land on `/profile`, which sends you straight on to [http://localhost:8000/users/1][user 1], with the message **Avatar updated**.
Upload it again, and run the `deno eval` command from [the start of this exercise][files table].
There are still only two files.

Nothing checks the file on the server yet.
The browser won't send the form without a file, because the input is `required`, but it will send any file you pick.
Make a text file, such as **notes.txt**, with a line of text in it.
On your profile, switch the file picker to show all files, pick **notes.txt**, and upload it.

The upload goes through, with **Avatar updated**, and the profile shows a broken picture.
The browser was told the file is a picture, and it isn't.
Open [http://localhost:8000/users/1/avatar][avatar 1], and you'll see the text of the file, served as `text/plain`.

{{<figure src="images/broken-avatar.webp" alt="testUser1's profile after uploading a text file, with the message Avatar updated and a broken picture where the avatar should be" caption="A text file is not an avatar.">}}

## Checking the upload {#avatar-schema}

We check every other form with a schema, so let's check this one too.
An avatar must be a file, of one of the types we accept, and no bigger than 1 MB.

We need three new validators.
Add `isFile`, `isType` and `maxSize` to **server/tools/validators.js**.

**server/tools/validators.js**
```JavaScript{hl_lines="48-73"}
import { ValidationError } from "./errors.js";

export function required(name, value) {
    if (!value.length) {
        throw new ValidationError(`${name} is a required field.`);
    }
}

export function minLength(len) {
    return (name, value) => {
        if (value.length < len) {
            throw new ValidationError(
                `${name} must be at least ${len} characters.`,
            );
        }
    };
}

export function maxLength(len) {
    return (name, value) => {
        if (value.length > len) {
            throw new ValidationError(
                `${name} must be no more than ${len} characters.`,
            );
        }
    };
}

export function hasDigit(name, value) {
    if (!/\d/.test(value)) {
        throw new ValidationError(`${name} must include a digit.`);
    }
}

export function hasSymbol(name, value) {
    if (!/[^A-Za-z0-9\s]/.test(value)) {
        throw new ValidationError(`${name} must include a symbol.`);
    }
}

export function unique(exists) {
    return (name, value) => {
        if (exists(value)) {
            throw new ValidationError(`${name} already taken.`);
        }
    };
}

export function isFile(name, value) {
    if (!(value instanceof File) || !value.size) {
        throw new ValidationError(`${name} is a required field.`);
    }
}

export function isType(types) {
    return (name, value) => {
        if (!types.includes(value.type)) {
            throw new ValidationError(
                `${name} must be one of these types: ${types.join(", ")}.`,
            );
        }
    };
}

export function maxSize(bytes) {
    return (name, value) => {
        if (value.size > bytes) {
            throw new ValidationError(
                `${name} must be no more than ${bytes / 1024 ** 2} MB.`,
            );
        }
    };
}
```

- `isFile` checks that the value is a `File` with something in it. A form whose file input is left empty sends an empty file, with a size of `0`, not nothing at all.
- `isType` is made from a list of types, like `minLength` is made from a length. It checks that the file's type is one of them.
- `maxSize` is made from a size in bytes. The message turns the bytes back into megabytes.

> The browser won't send our form without a file, because the input is `required`.
> So why does `isFile` check?
> The browser belongs to the user, and the [developer tools][forged form] can remove `required` from the input as easily as they can change a form's `action`.
> The server can't trust anything the browser sends, so it checks for itself.

Create **server/schema/avatar.js**.

**server/schema/avatar.js**
```JavaScript{hl_lines="1-11"}
import { isFile, isType, maxSize } from "../tools/validators.js";

export const AVATAR_TYPES = ["image/png", "image/jpeg", "image/webp"];
const MAX_AVATAR_SIZE = 1024 ** 2; // 1 MB

export const avatarSchema = {
    avatar: {
        displayName: "Avatar",
        validators: [isFile, isType(AVATAR_TYPES), maxSize(MAX_AVATAR_SIZE)],
    },
};
```

`AVATAR_TYPES` is exported, because the form's `accept` list will use it too.
`MAX_AVATAR_SIZE` is `1024 ** 2`, the number of bytes in a megabyte.

Update **server/main.js** to make `validateAvatar`, and put it on the route.

**server/main.js**
```JavaScript{hl_lines="26 43 73"}
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
import {
    avatarController,
    createUserController,
    profileController,
    signupFormController,
    uploadAvatarController,
    userController,
} from "./controllers/users.js";
import { userSchema } from "./schema/user.js";
import { avatarSchema } from "./schema/avatar.js";
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
import { withBook, withShelf, withTargetUser } from "./middleware/resources.js";

const validateNewShelf = withValidation(shelfSchema, myShelvesController);
const validateEditShelf = withValidation(shelfSchema, shelfController);
const validateNewUser = withValidation(userSchema, signupFormController);
const validateSignin = withValidation(signinSchema, signinFormController);
const validateAvatar = withValidation(avatarSchema, userController);

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
app.get("/profile", mustBeSignedIn, profileController);
app.get("/shelves", mustBeSignedIn, myShelvesController);
app.post("/shelves", mustBeSignedIn, validateNewShelf, createShelfController);
app.get("/shelves/:shelfId", withShelf, shelfController);
app.post("/shelves/:shelfId/edit", mustBeSignedIn, withShelf, mayUseShelf, validateEditShelf, editShelfController);
app.post("/shelves/:shelfId/delete", mustBeSignedIn, withShelf, mayUseShelf, deleteShelfController);
app.post("/shelves/:shelfId/books/:bookId/add", mustBeSignedIn, withShelf, withBook, mayUseShelf, addBookController);
app.post("/shelves/:shelfId/books/:bookId/delete", mustBeSignedIn, withShelf, withBook, mayUseShelf, removeBookController);
app.get("/users/:userId", withTargetUser, userController);
app.get("/users/:userId/avatar", withTargetUser, avatarController);
app.post("/users/:userId/avatar", mustBeSignedIn, withTargetUser, validateAvatar, uploadAvatarController);

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

If the file fails, `withValidation` falls back to `userController`, which shows the profile again with the error.

`withValidation` reads the form data, so the controller mustn't read it again.
Update `uploadAvatarController` in **server/controllers/users.js** to take the avatar from `ctx.validated`.

**server/controllers/users.js**
```JavaScript{hl_lines="42-43"}
import { createUser, updateUserAvatarById } from "../models/users.js";
import { getShelvesByOwnerId } from "../models/shelves.js";
import { addThumbnails } from "../models/shelfBooks.js";
import { createFile, deleteFileById, getFileById } from "../models/files.js";
import { signin } from "../tools/authentication.js";
import { NotFoundError } from "../tools/errors.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { signupView } from "../views/signup.js";
import { userView } from "../views/user.js";

export function signupFormController(ctx) {
    ctx.data.title = "Sign up";
    return render(signupView, ctx);
}

export async function createUserController(ctx) {
    const { validated } = ctx;
    const user = await createUser(validated);
    signin(user.id, ctx);
    ctx.flash = `Welcome, ${user.username}`;
    return redirect("/", ctx);
}

export function userController(ctx) {
    const { data: { targetUser } } = ctx;
    ctx.data.shelves = addThumbnails(getShelvesByOwnerId(targetUser.id));
    ctx.data.title = targetUser.username;
    return render(userView, ctx);
}

export function avatarController(ctx) {
    const { data: { targetUser } } = ctx;
    const avatarFile = targetUser.avatarId && getFileById(targetUser.avatarId);
    if (!avatarFile) {
        throw new NotFoundError(`No avatar for user ${targetUser.id}`);
    }
    return new Response(avatarFile);
}

export async function uploadAvatarController(ctx) {
    const { validated: { avatar }, data: { targetUser } } = ctx;
    const avatarId = await createFile(avatar);
    updateUserAvatarById(targetUser.id, avatarId);
    if (targetUser.avatarId) {
        deleteFileById(targetUser.avatarId);
    }
    ctx.flash = "Avatar updated";
    return redirect("/profile", ctx);
}

export function profileController(ctx) {
    return redirect(`/users/${ctx.user.id}`, ctx);
}
```

Update **server/views/user.js** to show the error.

**server/views/user.js**
```JavaScript{hl_lines="5 8 19-21 25 31-33"}
import { html } from "../tools/html.js";
import { avatarImage } from "./partials/avatarImage.js";
import { cardGrid } from "./partials/cardGrid.js";
import { shelfCard } from "./partials/shelfCard.js";
import { AVATAR_TYPES } from "../schema/avatar.js";

export function userView(ctx) {
    const { data: { targetUser, shelves, errors }, user: currentUser } = ctx;

    return html`
        <section>
            <header>
                <h1>${targetUser.username}</h1>
            </header>
            ${avatarImage(targetUser, `Avatar for '${targetUser.username}'`)}
            <h2>Shelves</h2>
            ${cardGrid(shelves.map((shelf) => shelfCard(shelf)), "No shelves yet")}
        </section>
        ${currentUser?.id === targetUser.id
            ? html`<aside>${avatarForm(targetUser, errors)}</aside>`
            : ""}
    `;
}

function avatarForm(user, errors) {
    return html`
        <h2>Change avatar</h2>
        <form class="secondary" method="POST" action="/users/${user.id}/avatar"
            enctype="multipart/form-data">
            <label for="avatar">Avatar</label>
            <input id="avatar" type="file" name="avatar" accept="${AVATAR_TYPES.join(",")}" required
                aria-describedby="avatar-error" aria-invalid="${Boolean(errors?.avatar)}">
            <span id="avatar-error" class="error">${errors?.avatar ?? ""}</span>
            <button>upload avatar</button>
        </form>
    `;
}
```

- `userView` takes `errors` from `ctx.data`, and passes them to `avatarForm`.
- The error shows under the input, with `aria-describedby` and `aria-invalid`, just like the shelf title form.
- The `accept` list comes from `AVATAR_TYPES`, so the form and the schema can't disagree.

Upload **notes.txt** again.
This time the page comes back at [http://localhost:8000/users/1/avatar][avatar 1], with **Avatar must be one of these types: image/png, image/jpeg, image/webp.** under the input, and the avatar stays as it was.
The terminal shows the `400` response.

```plaintext {linenos=false}

============================
POST /users/1/avatar
[session] {
  id: "8d595030-c2db-47b3-9923-d250cb0cf711",
  userId: 1,
  expires: "2026-11-25T15:58:03.623Z"
}
[user] { id: 1, username: "testUser1", avatarId: 1 }
RESPONSE 400
============================
```

{{<figure src="images/avatar-error.webp" alt="testUser1's profile after uploading a text file, with the error Avatar must be one of these types: image/png, image/jpeg, image/webp under the file input" caption="The text file is turned away.">}}

A picture still works.
Upload **testUser2.webp**, and you get **Avatar updated**.

## Only you {#may-change-avatar}

Only your own profile shows the upload form, but the route behind it takes an upload for any user.
We have seen this [before][forged form], with shelves.

> Try this on the seed data, then run `deno task db:reset` to get your avatars back.

1. Sign in as **testUser2** (the password is `testUser2!`) and open your own profile, [http://localhost:8000/users/2][user 2].
2. Open the developer tools (**F12**) on the **elements** panel, and find the upload form: `<form class="secondary" method="POST" action="/users/2/avatar" ...>`.
3. Double-click the `action` value, change it to `/users/1/avatar`, and press **Enter**.
4. Pick **testUser2.webp** and press **upload avatar**.

You get **Avatar updated**, and it's **testUser1**'s picture that has changed.
Look at [the home page][home]: **testUser1**'s shelves now have **testUser2**'s face beside them.

Reset the database to get **testUser1**'s picture back.

```bash {linenos=false}
deno task db:reset
```

This needs a `may` guard, just like [`mayUseShelf`][may use shelf].
Add `mayChangeAvatar` to **server/middleware/access.js**.

**server/middleware/access.js**
```JavaScript{hl_lines="22-27"}
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

export function mayChangeAvatar(ctx, next) {
    if (ctx.user?.id === ctx.data.targetUser.id) return next(ctx);
    const { url, request } = ctx;
    throw new NotFoundError(`${request.method} ${url.pathname} is not permitted`);
}
```

`mayChangeAvatar` is a plain middleware, like `mayUseShelf`.
It compares the signed-in user's `id` with the `id` of the user on `ctx.data`, the same question that `userView` asks before showing the form.
If they match, the rest of the route runs.
If not, it throws a `NotFoundError`.

Update **server/main.js** to put it on the upload route.

**server/main.js**
```JavaScript{hl_lines="36 73"}
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
import {
    avatarController,
    createUserController,
    profileController,
    signupFormController,
    uploadAvatarController,
    userController,
} from "./controllers/users.js";
import { userSchema } from "./schema/user.js";
import { avatarSchema } from "./schema/avatar.js";
import {
    alreadySignedInRedirectController,
    signinController,
    signinFormController,
    signinRedirectController,
    signoutController,
} from "./controllers/sessions.js";
import { signinSchema } from "./schema/signin.js";
import { withSession } from "./middleware/session.js";
import { mayChangeAvatar, mayUseShelf, requireGuest, requireUser } from "./middleware/access.js";
import { withBook, withShelf, withTargetUser } from "./middleware/resources.js";

const validateNewShelf = withValidation(shelfSchema, myShelvesController);
const validateEditShelf = withValidation(shelfSchema, shelfController);
const validateNewUser = withValidation(userSchema, signupFormController);
const validateSignin = withValidation(signinSchema, signinFormController);
const validateAvatar = withValidation(avatarSchema, userController);

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
app.get("/profile", mustBeSignedIn, profileController);
app.get("/shelves", mustBeSignedIn, myShelvesController);
app.post("/shelves", mustBeSignedIn, validateNewShelf, createShelfController);
app.get("/shelves/:shelfId", withShelf, shelfController);
app.post("/shelves/:shelfId/edit", mustBeSignedIn, withShelf, mayUseShelf, validateEditShelf, editShelfController);
app.post("/shelves/:shelfId/delete", mustBeSignedIn, withShelf, mayUseShelf, deleteShelfController);
app.post("/shelves/:shelfId/books/:bookId/add", mustBeSignedIn, withShelf, withBook, mayUseShelf, addBookController);
app.post("/shelves/:shelfId/books/:bookId/delete", mustBeSignedIn, withShelf, withBook, mayUseShelf, removeBookController);
app.get("/users/:userId", withTargetUser, userController);
app.get("/users/:userId/avatar", withTargetUser, avatarController);
app.post("/users/:userId/avatar", mustBeSignedIn, withTargetUser, mayChangeAvatar, validateAvatar, uploadAvatarController);

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

`mayChangeAvatar` goes straight after `withTargetUser`, which loads the user it checks.
It goes before `validateAvatar`, so a forged upload is turned away before its file is even read.
The route reads from left to right, just like the routes that change a shelf: are you signed in, which user is it, and is it you?

Sign in as **testUser2** and try the devtools trick again.
This time you get the not found page, and the terminal says why.

```plaintext {linenos=false}

============================
POST /users/1/avatar
[session] {
  id: "bf31970a-73cd-4238-a177-17484ddfae90",
  userId: 2,
  expires: "2026-11-25T15:58:09.561Z"
}
[user] { id: 2, username: "testUser2", avatarId: 2 }
[error] POST /users/1/avatar is not permitted
RESPONSE 404
============================
```

**testUser1** keeps their picture, and you can still change your own.

The answer is `404 Not Found`, not `403 Forbidden`, by the [rule from the last unit][not found or forbidden].
Whether you may change an avatar depends on *whose* it is, so for **testUser2**, the route that changes **testUser1**'s avatar simply doesn't exist.

### The pattern again {#pattern-again}

We have used every part of our [permissions pattern][pattern] again.

1. **Load the thing.** `withTargetUser` loads the user the route is about, or the route is not found.
2. **Ask one question.** Is this user allowed to change this avatar? The question is `currentUser?.id === targetUser.id`.
3. **Use the answer twice.** `mayChangeAvatar` turns away anyone the answer is no for, and `userView` only shows the form to anyone it's yes for.
4. **Filter lists in SQL.** The profile's shelves come from `getShelvesByOwnerId`, and the home page's from `getShelves`, so each list only fetches the rows it shows.

`/profile` only needed `mustBeSignedIn`, because it's about you alone.
`mayChangeAvatar` is about you *and* another user, so it comes after the `with` middleware that loads them.

We now have two questions, *is this your shelf?* and *is this you?*, and we write each one out in two places.
In the next unit, we will give each question a name of its own, and make our guards from them.

## Conclusion

Every user now has an avatar, or the default picture.
The pictures are stored in the database as files, and served from their own route, with an address that changes when the picture does.
Avatars show on profiles and beside users' names.
You can upload a new avatar from your own profile: the form sends the file with `multipart/form-data`, `avatarSchema` checks its type and size, the old file is deleted, and `mayChangeAvatar` makes sure it's your own.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="6-7 13 37 43 60"}
my-app/
├── .vscode/
│   └── settings.json
├── db/
│   ├── seed-avatars/
│   │   ├── testUser1.webp
│   │   └── testUser2.webp
│   ├── app.db
│   ├── BOOKS-ATTRIBUTION.md
│   ├── books.json
│   └── seed.json
├── public/
│   ├── default-avatar.svg
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
│   │   ├── files.js
│   │   ├── sessions.js
│   │   ├── shelfBooks.js
│   │   ├── shelves.js
│   │   └── users.js
│   ├── schema/
│   │   ├── avatar.js
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
│   │   │   ├── avatarImage.js
│   │   │   ├── bookCard.js
│   │   │   ├── cardGrid.js
│   │   │   ├── postButton.js
│   │   │   ├── searchForm.js
│   │   │   ├── shelfCard.js
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
│   │   ├── signup.js
│   │   └── user.js
│   └── main.js
├── tasks/
│   ├── db-create.js
│   ├── db-seed.js
│   └── server.js
├── .gitignore
├── deno.json
└── deno.lock
```

## Challenge: Whose picture?

These are questions to investigate, not changes to keep.

- Find or make a picture bigger than 1 MB, and upload it. Which validator turns it away, and what does the message say?
- Sign up a new user. What does `/users/:userId/avatar` give for them, and what does their profile show instead?
- On the upload route, move `mayChangeAvatar` after `validateAvatar`. As **testUser2**, use the devtools trick to upload **notes.txt** to **testUser1**. What do you get, and why is that worse?
- Why does `GET /users/:userId/avatar` need no `may` guard?

<details>
<summary>Why the order matters</summary>

With `validateAvatar` first, the file is read and checked before anyone asks whose avatar it is.
**testUser2**'s forged upload of **notes.txt** gets **testUser1**'s profile back with a `400`, and no form, because it isn't their profile.
The upload still fails, but the server has read the file for someone who was never allowed to send it.
A `400` also tells them that the route is there and their file was the problem, when the route should simply not exist for them.
With `mayChangeAvatar` first, they get the not found page, and the file is never read.

Anyone may *see* an avatar, so the route that serves them has nothing to guard.
That will change when profiles can be private.

</details>

> If you tried any changes, undo them, and run `deno task db:reset`, so that your code and data match the starting point of the next unit.

[previous exercise]: profiles-and-avatars/profile-pages/
[owner id]: owners-and-guards/owners/#owner-id
[pattern]: owners-and-guards/owners/#pattern
[forged form]: owners-and-guards/owners/#forged-form
[may use shelf]: owners-and-guards/owners/#may-use-shelf
[not found or forbidden]: owners-and-guards/owners/#not-found-or-forbidden
[require user]: owners-and-guards/guards/#require-user
[async]: router-class/middleware/#async
[files table]: #files-table
[home]: http://localhost:8000/
[user 1]: http://localhost:8000/users/1
[user 2]: http://localhost:8000/users/2
[avatar 1]: http://localhost:8000/users/1/avatar
[avatar 2]: http://localhost:8000/users/2/avatar
[avatar 99]: http://localhost:8000/users/99/avatar
[shelf 4]: http://localhost:8000/shelves/4
[sqlite blob]: https://www.sqlite.org/datatype3.html
[media types]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/MIME_types
[file]: https://developer.mozilla.org/en-US/docs/Web/API/File
[file constructor]: https://developer.mozilla.org/en-US/docs/Web/API/File/File
[blob]: https://developer.mozilla.org/en-US/docs/Web/API/Blob
[blob bytes]: https://developer.mozilla.org/en-US/docs/Web/API/Blob/bytes
[response]: https://developer.mozilla.org/en-US/docs/Web/API/Response/Response
[import meta]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import.meta
[deno read file]: https://docs.deno.com/api/deno/~/Deno.readFile
[deno eval]: https://docs.deno.com/runtime/reference/cli/eval/
[object fit]: https://developer.mozilla.org/en-US/docs/Web/CSS/object-fit
[input file]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/input/file
[accept]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Attributes/accept
[enctype]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/form#enctype
[form data]: https://developer.mozilla.org/en-US/docs/Web/API/Request/formData
[form data object]: https://developer.mozilla.org/en-US/docs/Web/API/FormData

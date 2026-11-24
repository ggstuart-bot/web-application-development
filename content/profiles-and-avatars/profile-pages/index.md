---
type: exercise
title: Profile pages
weight: 1
author: Dr Graeme Stuart
---

In this exercise we will list everyone's shelves on the home page, and say whose they are.
Each user will get a profile page that lists their shelves, with a link to your own profile in the nav.

<!--more-->

> You should have owners for every shelf, and the `mayUseShelf` guard, from the [previous exercise][previous exercise].
> Your database should hold the seeded users and their shelves. If you have been experimenting, run `deno task db:reset` to get back to the seed data.
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

Every shelf has an owner, but you can only find your own.
In this exercise we will make everyone's shelves easy to find, and give every user a page of their own.
By the end of this exercise you should be able to:

- Move a view function into a partial so that more than one view can use it
- Join two tables in a query, so each row comes with what we need to show it
- Explain why passing a function straight to `map` can give it more arguments than it expects
- Load a user named in the path with a `with` middleware, and keep them apart from the signed-in user
- Reuse a query that is already filtered in SQL on a new page

## A shelf card partial {#shelf-card}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

We are going to list shelves on the home page, and on every user's profile.
So far, only My shelves shows shelf cards, and `shelfCard` lives in **views/myShelves.js**.
Before we use it anywhere else, it needs a module of its own.

Create **server/views/partials/shelfCard.js**, and move `shelfCard` into it, along with `coverImage`, which only `shelfCard` uses.

**server/views/partials/shelfCard.js**
```JavaScript{hl_lines="1-26"}
import { html } from "../../tools/html.js";
import { DEFAULT_COVER } from "./bookCard.js";

// A shelf as a card in a .grid list, with a fanned row of its first few covers.
// Shelf lists always sit under an <h2>, so the title is an <h3>
export function shelfCard(shelf) {
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

`shelfCard` is exported now, because another module needs it.
`coverImage` isn't, because nothing else uses it.
The import of `DEFAULT_COVER` comes with it, and the paths to **html.js** and **bookCard.js** have changed, because the file is one folder deeper.

Now update **server/views/myShelves.js** to import it.

**server/views/myShelves.js**
```JavaScript{hl_lines="3"}
import { html } from "../tools/html.js";
import { cardGrid } from "./partials/cardGrid.js";
import { shelfCard } from "./partials/shelfCard.js";
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
```

The two functions have gone from the bottom of the file, and so has the import of `DEFAULT_COVER`, which only they used.

Refresh [My shelves][shelves] while signed in as **testUser1** (the password is `testUser1!`).
It looks exactly as it did.

## Everyone's shelves {#everyones-shelves}

The home page only has a search form.
Shelves are worth sharing, so let's list every shelf under it.

Add `getShelves` to **server/models/shelves.js**.

**server/models/shelves.js**
```JavaScript{hl_lines="26-35"}
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
        SELECT shelves.id, title, ownerId, username,
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

`getShelves` finds every shelf, with the number of books on each, like `getShelvesByOwnerId` without the `WHERE` clause.
It also *joins* each shelf to its owner in the users table, with `JOIN users ON users.id = shelves.ownerId`, just as the shelf's books are joined to the books table.
So each row comes with its owner's `username`, ready to show.
Both tables have an `id` column, so we say `shelves.id` to make it clear which one we mean.

> The placeholder says this will change when shelves can be public.
> For now, every shelf is everyone's to see, so the list is everything.
> When some shelves are private, the `WHERE` clause that leaves them out will go here, in the SQL, as for [My shelves][my shelves].

Update **server/controllers/home.js** to fetch the shelves.

**server/controllers/home.js**
```JavaScript{hl_lines="1-2 7"}
import { getShelves } from "../models/shelves.js";
import { addThumbnails } from "../models/shelfBooks.js";
import { render } from "../tools/render.js";
import { homeView } from "../views/home.js";

export function homeController(ctx) {
    ctx.data.shelves = addThumbnails(getShelves());
    return render(homeView, ctx);
}
```

`homeController` adds the covers to each shelf with `addThumbnails`, just as `myShelvesController` does, and puts the shelves on `ctx.data`.

Update **server/views/home.js** to show them.

**server/views/home.js**
```JavaScript{hl_lines="2-3 6-8 17-18"}
import { html } from "../tools/html.js";
import { cardGrid } from "./partials/cardGrid.js";
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
            ${cardGrid(shelves.map(shelfCard), "Nobody has made a shelf yet")}
        </section>
    `;
}
```

`homeView` needs the shelves now, so it takes `ctx`.
The shelves go under an **Everyone's shelves** heading, as shelf cards in a grid.
That's `shelfCard`'s second use.

Open [http://localhost:8000][home].
Under the search form, you see all five shelves, from both users.
The cards don't say whose shelves they are yet.

## Whose shelf? {#shelf-owner}

A shelf card on the home page should say who it belongs to, but My shelves doesn't need to, because they are all yours.
So `shelfCard` will take some extra content for the card's footer, and leave the footer out when there's nothing to show.

Update **server/views/partials/shelfCard.js**.

**server/views/partials/shelfCard.js**
```JavaScript{hl_lines="5 7 18"}
import { html } from "../../tools/html.js";
import { DEFAULT_COVER } from "./bookCard.js";

// A shelf as a card in a .grid list, with a fanned row of its first few covers.
// Pass extra content (e.g. the owner) to show it in the card footer.
// Shelf lists always sit under an <h2>, so the title is an <h3>
export function shelfCard(shelf, footer = "") {
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
                ${footer ? html`<footer>${footer}</footer>` : ""}
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

`footer` has a [default value][default parameters] of `""`, so a call without it gets no footer at all.

The home page passes each shelf's owner.
Update **server/views/home.js**.

**server/views/home.js**
```JavaScript{hl_lines="18-21"}
import { html } from "../tools/html.js";
import { cardGrid } from "./partials/cardGrid.js";
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
                shelves.map((shelf) => shelfCard(shelf, shelf.username)),
                "Nobody has made a shelf yet",
            )}
        </section>
    `;
}
```

Notice that we can no longer write `shelves.map(shelfCard)`.
That only passed the shelf, but [`map`][map] calls its function with *three* arguments: the item, its index and the whole array.
Until now, `shelfCard` ignored the extra two, but its second parameter is now `footer`, so the index would end up in the footer.
The first card has index `0`, which counts as empty, but the second would say **1**, the third **2**, and so on.

So we write a small arrow function that passes exactly what we mean: `(shelf) => shelfCard(shelf, shelf.username)`.

My shelves has the same problem.
Update **server/views/myShelves.js**.

**server/views/myShelves.js**
```JavaScript{hl_lines="16"}
import { html } from "../tools/html.js";
import { cardGrid } from "./partials/cardGrid.js";
import { shelfCard } from "./partials/shelfCard.js";
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
            ${cardGrid(shelves.map((shelf) => shelfCard(shelf)), "No shelves yet")}
        </section>
        <aside>
            <h2>New shelf</h2>
            ${shelfTitleForm("/shelves", values?.title ?? "", errors?.title, "create")}
        </aside>
    `;
}
```

Here, `(shelf) => shelfCard(shelf)` passes the shelf and nothing else, so My shelves has no footers.

Refresh [the home page][home].
Each card now has its owner's name at the bottom: three for **testUser1** and two for **testUser2**.
[My shelves][shelves] still has no footers.

## Link to the owner {#owner-link}

Seeing whose shelf it is makes you want to see who they are, and what else they have put on their shelves.
Each user is going to get a profile page at `/users/:userId`, so the owner's name should be a link to it.

Update **server/views/home.js** again.

**server/views/home.js**
```JavaScript{hl_lines="19 25-28"}
import { html } from "../tools/html.js";
import { cardGrid } from "./partials/cardGrid.js";
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
    return html`<a href="/users/${shelf.ownerId}">${shelf.username}</a>`;
}
```

`ownerLink` makes the link from the shelf's `ownerId` and `username`.
It isn't exported, because only the home page uses it.

A shelf's page says whose shelf it is too.
Update **server/views/publicShelf.js**.

**server/views/publicShelf.js**
```JavaScript{hl_lines="12"}
import { html } from "../tools/html.js";
import { bookCard } from "./partials/bookCard.js";
import { cardGrid } from "./partials/cardGrid.js";

export function publicShelfView(ctx) {
    const { data: { shelf, books, owner } } = ctx;

    return html`
        <section>
            <header>
                <h1>${shelf.title}</h1>
                <p>A shelf by <a href="/users/${owner.id}">${owner.username}</a></p>
            </header>
            ${cardGrid(books.map((book) => bookCard(book)), "No books on this shelf yet")}
            <p><a href="/">Back to home</a></p>
        </section>
    `;
}
```

Refresh [the home page][home] and follow one of the owner links.
You get the not found page, because there is no such route yet.

## A profile page {#target-user}

Each user needs a profile page.
It starts as a stub, with just their name.

The route will be `/users/:userId`, so first we need to load the user it names.
That's a job for a `with` middleware, the first part of our [permissions pattern][pattern].
Add `withTargetUser` to **server/middleware/resources.js**.

**server/middleware/resources.js**
```JavaScript{hl_lines="1 6-15"}
import { getUserById } from "../models/users.js";
import { getShelfById } from "../models/shelves.js";
import { getBookById } from "../models/books.js";
import { NotFoundError } from "../tools/errors.js";

export function withTargetUser(ctx, next) {
    const { params: { userId } } = ctx;
    const targetUser = getUserById(Number(userId));
    if (!targetUser) {
        throw new NotFoundError(`User ${userId} not found`);
    }
    ctx.data.targetUser = targetUser;
    return next(ctx);
}

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

`withTargetUser` works just like `withShelf`.
It looks up the user with `getUserById`, puts them on `ctx.data`, and throws a `NotFoundError` if there is no such user.

We call them `targetUser`, not `user`, for a good reason.
`ctx.user` is already whoever is signed in, and the two are often different people.
When **testUser2** looks at **testUser1**'s profile, `ctx.user` is **testUser2**, and `ctx.data.targetUser` is **testUser1**.

Create **server/views/user.js**.

**server/views/user.js**
```JavaScript{hl_lines="1-13"}
import { html } from "../tools/html.js";

export function userView(ctx) {
    const { data: { targetUser } } = ctx;

    return html`
        <section>
            <header>
                <h1>${targetUser.username}</h1>
            </header>
        </section>
    `;
}
```

For now, `userView` has the username as its heading, and nothing else.

Add `userController` to **server/controllers/users.js**.

**server/controllers/users.js**
```JavaScript{hl_lines="6 20-25"}
import { createUser } from "../models/users.js";
import { signin } from "../tools/authentication.js";
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
    ctx.data.title = targetUser.username;
    return render(userView, ctx);
}
```

`userController` gives the page the username as its title, and renders `userView`.
It doesn't need to check that the user exists, because `withTargetUser` runs first.

Finally, update **server/main.js** to add the route.

**server/main.js**
```JavaScript{hl_lines="17 29 61"}
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
import { createUserController, signupFormController, userController } from "./controllers/users.js";
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

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

`withTargetUser` goes in front of `userController`, so an unknown user is not found.
The route has no guards, because a profile is open to everyone.

Follow an owner link on [the home page][home].
You get [http://localhost:8000/users/1][user 1], with **testUser1** as the heading.
Sign out, and the profile is still there, because profiles are open to everyone.
Still signed out, try [http://localhost:8000/users/99][user 99], and you get the not found page, and the terminal says why.

```plaintext {linenos=false}

============================
GET /users/99
[session] undefined
[user] undefined
[error] User 99 not found
RESPONSE 404
============================
```

## A user's shelves {#profile-shelves}

A profile with only a name doesn't say much.
It should list the user's shelves, so you can find the rest of someone's shelves from any one of them.

We already have a query for one user's shelves, the one that [My shelves][my shelves] uses.
Update `userController` in **server/controllers/users.js**.

**server/controllers/users.js**
```JavaScript{hl_lines="2-3 25"}
import { createUser } from "../models/users.js";
import { getShelvesByOwnerId } from "../models/shelves.js";
import { addThumbnails } from "../models/shelfBooks.js";
import { signin } from "../tools/authentication.js";
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
```

`getShelvesByOwnerId` finds the target user's shelves, filtered in SQL, and `addThumbnails` adds their covers.

Update **server/views/user.js** to show them.

**server/views/user.js**
```JavaScript{hl_lines="2-3 6 13-14"}
import { html } from "../tools/html.js";
import { cardGrid } from "./partials/cardGrid.js";
import { shelfCard } from "./partials/shelfCard.js";

export function userView(ctx) {
    const { data: { targetUser, shelves } } = ctx;

    return html`
        <section>
            <header>
                <h1>${targetUser.username}</h1>
            </header>
            <h2>Shelves</h2>
            ${cardGrid(shelves.map((shelf) => shelfCard(shelf)), "No shelves yet")}
        </section>
    `;
}
```

The shelves go under a **Shelves** heading, as shelf cards with no footer, because they all belong to the same person.
That's `shelfCard`'s third use, and the reason we moved it into a partial.

Refresh [http://localhost:8000/users/1][user 1].
You see **classics**, **fantasy** and **to read**.
[http://localhost:8000/users/2][user 2] has **sci-fi** and **children's favourites**.

## Your profile in the nav {#profile-link}

To reach your own profile, you have to find one of your shelves on the home page, and follow your name.
A signed-in user should have a link to it in the nav.

Update the `authLinks` function in **server/views/layout.js**.

**server/views/layout.js**
```JavaScript{hl_lines="53"}
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
        ${navLink(`/users/${user.id}`, "profile", path)}
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

The **profile** link goes between **my shelves** and the sign out button, to `/users/` followed by your own `id`.
`navLink` marks it as the current page when you are on your own profile, because then the path matches exactly.
On anyone else's profile, the path is different, so it isn't marked.

As **testUser1**, click **profile** in the nav.
You land on [http://localhost:8000/users/1][user 1], and the link is marked as the current page.
Visit [http://localhost:8000/users/2][user 2], and it isn't.

{{<figure src="images/profile.webp" alt="testUser1's profile page, with the heading testUser1 and three shelf cards under Shelves, and the profile link marked in the nav" caption="testUser1's own profile.">}}

## Conclusion

The home page now lists everyone's shelves, with a link to each owner.
Every user has a profile page, loaded by `withTargetUser`, that lists their shelves with the same SQL query as My shelves.
A signed-in user can reach their own profile from the nav.
`shelfCard` is a partial with three uses, and we saw why we pass it exactly the arguments we mean.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="58 70"}
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

## Challenge: Who's who?

These are questions to investigate, not changes to keep.

- Visit [http://localhost:8000/users/abc][user abc]. What does `Number("abc")` give, and why do you still get the not found page?
- Sign up a new user. What does their profile show, and why doesn't their name appear on the home page?
- On the home page, change the `map` back to `shelves.map(shelfCard)`. Which cards get a footer, and what does it say?
- Sign in as **testUser2** and visit **testUser1**'s profile. What is `ctx.user`, and what is `ctx.data.targetUser`?

<details>
<summary>What map passes</summary>

`map` calls `shelfCard(shelf, index, shelves)`, so `footer` is the index.
The first card's index is `0`, which counts as empty, so it has no footer.
Every other card gets its index as a footer: **1**, **2**, **3** and **4**.

`Number("abc")` is `NaN`, and no user has `NaN` as an `id`.
`getUserById` finds nothing, so `withTargetUser` throws a `NotFoundError`.

</details>

> If you tried any changes, undo them, and run `deno task db:reset`, so that your code and data match the starting point of the next exercise.

[previous exercise]: owners-and-guards/owners/
[my shelves]: owners-and-guards/owners/#my-shelves
[pattern]: owners-and-guards/owners/#pattern
[home]: http://localhost:8000/
[shelves]: http://localhost:8000/shelves
[user 1]: http://localhost:8000/users/1
[user 2]: http://localhost:8000/users/2
[user 99]: http://localhost:8000/users/99
[user abc]: http://localhost:8000/users/abc
[default parameters]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Default_parameters
[map]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map

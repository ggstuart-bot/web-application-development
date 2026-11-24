---
type: exercise
title: Private profiles
weight: 1
author: Dr Graeme Stuart
---

In this exercise we will let each user keep their profile to themselves.
A private profile, and its avatar, will be not found for everyone but its owner, and you will be able to make your profile public, and private again.

<!--more-->

> You should have profile pages, avatars and the `mayChangeAvatar` guard from the [avatars exercise][avatars].
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

Everything about a user is on show to anyone: their profile, their picture and every shelf they make.
Some people would rather keep their reading to themselves.
By the end of this exercise you should be able to:

- Add a setting to a table with a default, so new rows get the safe choice
- Hide a thing that can't be seen behind a `404 Not Found`, so it doesn't even admit that it exists
- Write the questions our app asks about permissions as small, named functions in one module
- Make `may` guards from those functions with a factory, `requirePermission`
- Ask the same named question in a route and in a view

## Public or private {#is-public}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

A user's profile is going to be public or private.
Update **tasks/db-create.js** to give the users table an `isPublic` column.

**tasks/db-create.js**
```JavaScript{hl_lines="25-26"}
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
        avatarId INTEGER REFERENCES files(id) ON DELETE SET NULL,
        isPublic INTEGER NOT NULL DEFAULT 0
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

SQLite has no true or false, so we use `1` for public and `0` for private.
`NOT NULL DEFAULT 0` means a profile is private unless its owner says otherwise.
When there's a choice, the default should be the safe one: nobody's reading is shared until they choose to share it.

> The users table keeps its placeholder, because users will get one more column in the next unit.
> The shelves table's placeholder hasn't changed.

Update **server/models/users.js**.

**server/models/users.js**
```JavaScript{hl_lines="12-14 24 27-28 30"}
import { hashPassword } from "../tools/hash.js";
import { db } from "./db.js";

export function getUserByUsername(username) {
    return db.prepare(
        `SELECT * FROM users WHERE username=:username`,
    ).get({ username });
}

// Placeholder: this will need to change when users get more columns
export function getUserById(id) {
    return db.prepare(
        `SELECT id, username, avatarId, isPublic FROM users WHERE id=:id`,
    ).get({ id });
}

export function updateUserAvatarById(id, avatarId) {
    return db.prepare(
        `UPDATE users SET avatarId=:avatarId WHERE id=:id`,
    ).run({ id, avatarId });
}

// Placeholder: this will need to change when users get more columns
export async function createUser({ username, password, isPublic = 0 }) {
    const hashedPassword = await hashPassword(password);
    return db.prepare(`
        INSERT INTO users(username, hashedPassword, isPublic)
        VALUES (:username, :hashedPassword, :isPublic)
        RETURNING *
    `).get({ username, hashedPassword, isPublic });
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

- `createUser` takes `isPublic`, with `0` as its default, so signing up still makes a private profile.
- `getUserById` returns `isPublic`. The signed-in user comes from `getUserById` too, so `ctx.user` has it as well.

> Both placeholders stay, because users will get one more column in the next unit.

The seed data should have one of each.
Update **db/seed.json** so that **testUser1** is public.

**db/seed.json**
```json{hl_lines="7"}
{
    "users": [
        {
            "username": "testUser1",
            "password": "testUser1!",
            "avatar": "testUser1.webp",
            "isPublic": 1,
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

**testUser2** has no `isPublic`, so `createUser` makes them private.

Your own profile should say whether it's public or private.
Update **server/views/user.js**.

**server/views/user.js**
```JavaScript{hl_lines="20-25 43-49"}
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
            ? html`
                <aside>
                    ${avatarForm(targetUser, errors)}
                    ${visibilityForm(targetUser)}
                </aside>
            `
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

function visibilityForm(user) {
    return html`
        <h2>Profile is ${user.isPublic ? "public" : "private"}</h2>
        <p>${user.isPublic ? "Anyone" : "Only you"} can see your profile</p>
    `;
}
```

The aside now has two parts, the avatar form and `visibilityForm`.
For now, `visibilityForm` only says who can see your profile.

Rebuild the database.

```bash {linenos=false}
deno task db:reset
```

Sign in as **testUser2** (the password is `testUser2!`) and click **profile**.
The aside says **Profile is private** and **Only you can see your profile**.

That isn't true yet.
Sign out and open [http://localhost:8000/users/2][user 2]: **testUser2**'s profile is there for anyone.

## A private profile {#may-view-user}

A private profile still opens for anyone who has the link.
We need another `may` guard, like [`mayChangeAvatar`][may change avatar].

Add `mayViewUser` to **server/middleware/access.js**.

**server/middleware/access.js**
```JavaScript{hl_lines="28-33"}
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

export function mayViewUser(ctx, next) {
    if (ctx.user?.id === ctx.data.targetUser.id || ctx.data.targetUser.isPublic) return next(ctx);
    const { url, request } = ctx;
    throw new NotFoundError(`${request.method} ${url.pathname} is not permitted`);
}
```

`mayViewUser` is a near-copy of `mayChangeAvatar`.
The rest of the route runs if you are the user whose profile it is, *or* if the profile is public.
Otherwise it throws a `NotFoundError`.

Update **server/main.js** to put it on the profile route.

**server/main.js**
```JavaScript{hl_lines="36-42 77"}
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
import {
    mayChangeAvatar,
    mayUseShelf,
    mayViewUser,
    requireGuest,
    requireUser,
} from "./middleware/access.js";
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
app.get("/users/:userId", withTargetUser, mayViewUser, userController);
app.get("/users/:userId/avatar", withTargetUser, avatarController);
app.post("/users/:userId/avatar", mustBeSignedIn, withTargetUser, mayChangeAvatar, validateAvatar, uploadAvatarController);

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

`mayViewUser` goes straight after `withTargetUser`, which loads the user it checks.
There's no `mustBeSignedIn` in front, because a public profile is for everyone, signed in or not.

Sign out and open [http://localhost:8000/users/1][user 1].
**testUser1** is public, so their profile opens.
Now open [http://localhost:8000/users/2][user 2], and you get the not found page.

```plaintext {linenos=false}

============================
GET /users/2
[session] undefined
[user] undefined
[error] GET /users/2 is not permitted
RESPONSE 404
============================
```

Sign in as **testUser2**, and your own profile opens as before.

This is the [404 rule][not found or forbidden] again, and it matters even more here.
Whether you can see a profile depends on whose it is, so the answer is `404 Not Found`, exactly as if there were no such user.
A `403 Forbidden` would say *there is someone here, but you can't see them*.
A private profile doesn't even admit that it exists.

> **testUser2**'s name on the home page now leads to the not found page for everyone else.
> We will put that right when the home page shows only public shelves, in the [next exercise][next exercise].

## A private avatar {#private-avatar}

In the [avatars exercise][avatars], we said that anyone may see an avatar, and that this would change when profiles can be private.
**testUser2**'s profile is not found for everyone else now, but their picture isn't.
Signed out, [http://localhost:8000/users/2/avatar][avatar 2] still serves it, to anyone who guesses the address.

Update **server/main.js** to put `mayViewUser` on the avatar route too.

**server/main.js**
```JavaScript{hl_lines="78"}
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
import {
    mayChangeAvatar,
    mayUseShelf,
    mayViewUser,
    requireGuest,
    requireUser,
} from "./middleware/access.js";
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
app.get("/users/:userId", withTargetUser, mayViewUser, userController);
app.get("/users/:userId/avatar", withTargetUser, mayViewUser, avatarController);
app.post("/users/:userId/avatar", mustBeSignedIn, withTargetUser, mayChangeAvatar, validateAvatar, uploadAvatarController);

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

A private user's picture is now treated just like their profile.
Signed out, [http://localhost:8000/users/1/avatar][avatar 1] still works, and [http://localhost:8000/users/2/avatar][avatar 2] is not found.
Sign in as **testUser2**, and your own picture still shows in the nav and on your profile.

> On the home page, **testUser2**'s avatar beside their shelves is now a broken picture for everyone else.
> Like the link beside it, it goes when the home page shows only public shelves.

## Name the questions {#permissions}

The same question is now asked in three places: *is the signed-in user the one whose profile this is?*

- `mayChangeAvatar` asks it.
- `mayViewUser` asks it, as part of a bigger question.
- `userView` asks it, to decide whether to show the aside.

Each one writes out the `id` comparison again, and one day they will disagree.
In the [last unit][pattern again] we said that each question would get a name of its own.
Now it does.

Create **server/tools/permissions.js**.

**server/tools/permissions.js**
```JavaScript{hl_lines="1-4"}
export const isSelf = (user, targetUser) => user?.id === targetUser.id;
export const canViewUser = (user, targetUser) =>
    isSelf(user, targetUser) || Boolean(targetUser.isPublic);
export const canChangeAvatar = (user, targetUser) => isSelf(user, targetUser);
```

Each function takes the signed-in user and the thing being looked at, and answers yes or no.
They are short [arrow functions][arrow functions], so each answer fits on a line or two.

- `isSelf` is the `id` comparison, written once. `user?.id` means a visitor who isn't signed in is never anyone.
- `canViewUser` is `isSelf`, or a public profile. [`Boolean`][boolean] turns the `1` or `0` from the database into `true` or `false`, so every question answers with `true` or `false`.
- `canChangeAvatar` is just `isSelf`, but its name says what it's *for*. If the rule for changing an avatar ever changes, it changes here, and only here.

We call these *predicates*: functions that answer a question with `true` or `false`.
A `can` predicate asks whether a user can do something to a thing.

Update **server/middleware/access.js** to ask them.

**server/middleware/access.js**
```JavaScript{hl_lines="2 25 31"}
import { NotFoundError } from "../tools/errors.js";
import { canChangeAvatar, canViewUser } from "../tools/permissions.js";

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
    if (canChangeAvatar(ctx.user, ctx.data.targetUser)) return next(ctx);
    const { url, request } = ctx;
    throw new NotFoundError(`${request.method} ${url.pathname} is not permitted`);
}

export function mayViewUser(ctx, next) {
    if (canViewUser(ctx.user, ctx.data.targetUser)) return next(ctx);
    const { url, request } = ctx;
    throw new NotFoundError(`${request.method} ${url.pathname} is not permitted`);
}
```

Update **server/views/user.js** to ask them too.

**server/views/user.js**
```JavaScript{hl_lines="2 10-11 22 25-27"}
import { html } from "../tools/html.js";
import { canChangeAvatar, isSelf } from "../tools/permissions.js";
import { avatarImage } from "./partials/avatarImage.js";
import { cardGrid } from "./partials/cardGrid.js";
import { shelfCard } from "./partials/shelfCard.js";
import { AVATAR_TYPES } from "../schema/avatar.js";

export function userView(ctx) {
    const { data: { targetUser, shelves, errors }, user: currentUser } = ctx;
    // Only you get controls, so only then is there an aside
    const hasControls = isSelf(currentUser, targetUser);

    return html`
        <section>
            <header>
                <h1>${targetUser.username}</h1>
            </header>
            ${avatarImage(targetUser, `Avatar for '${targetUser.username}'`)}
            <h2>Shelves</h2>
            ${cardGrid(shelves.map((shelf) => shelfCard(shelf)), "No shelves yet")}
        </section>
        ${hasControls
            ? html`
                <aside>
                    ${canChangeAvatar(currentUser, targetUser)
                        ? avatarForm(targetUser, errors)
                        : ""}
                    ${visibilityForm(targetUser)}
                </aside>
            `
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

function visibilityForm(user) {
    return html`
        <h2>Profile is ${user.isPublic ? "public" : "private"}</h2>
        <p>${user.isPublic ? "Anyone" : "Only you"} can see your profile</p>
    `;
}
```

- `hasControls` uses `isSelf` to decide whether there is an aside at all, because only you get controls on your profile.
- `canChangeAvatar` decides whether the avatar form is in it. It's the same question that `mayChangeAvatar` asks, so the view and the route can't disagree.

Nothing changes in the browser.
The same profiles and avatars open, and the same ones are not found.

## Guards from questions {#require-permission}

Look at `mayUseShelf`, `mayChangeAvatar` and `mayViewUser`.
They are three middlewares with one shape.
Each asks a question about the signed-in user and something on `ctx.data`, runs the rest of the route if the answer is yes, and throws a `NotFoundError` if not.
Only the question, and the thing it's asked about, change.

When functions differ only in a value, we make them with a factory, just as [`requireUser`][require user] makes a guard from a fallback.
Replace the three middlewares in **server/middleware/access.js** with `requirePermission`.

**server/middleware/access.js**
```JavaScript{hl_lines="17-22"}
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

export function requirePermission(predicate, key) {
    return (ctx, next) => {
        if (predicate(ctx.user, ctx.data[key])) return next(ctx);
        const { url, request } = ctx;
        throw new NotFoundError(`${request.method} ${url.pathname} is not permitted`);
    };
}
```

`requirePermission` takes a predicate and a *key*, and returns a middleware.

- The key says where in `ctx.data` to find the thing: `"targetUser"` or `"shelf"`.
- `ctx.data[key]` uses [square brackets][property accessors] to look up the property whose name is in `key`, so `ctx.data["shelf"]` is `ctx.data.shelf`.
- If the predicate says yes, the rest of the route runs. If not, it throws a `NotFoundError`, as before.

`mayUseShelf` asked its own question inline, so it needs a predicate too.
Add `canUseShelf` to **server/tools/permissions.js**.

**server/tools/permissions.js**
```JavaScript{hl_lines="5"}
export const isSelf = (user, targetUser) => user?.id === targetUser.id;
export const canViewUser = (user, targetUser) =>
    isSelf(user, targetUser) || Boolean(targetUser.isPublic);
export const canChangeAvatar = (user, targetUser) => isSelf(user, targetUser);
export const canUseShelf = (user, shelf) => user?.id === shelf.ownerId;
```

Now update **server/main.js** to make the three guards, next to `mustBeSignedIn` and `mustBeSignedOut`.

**server/main.js**
```JavaScript{hl_lines="36 38 48-50"}
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
import { requireGuest, requirePermission, requireUser } from "./middleware/access.js";
import { withBook, withShelf, withTargetUser } from "./middleware/resources.js";
import { canChangeAvatar, canUseShelf, canViewUser } from "./tools/permissions.js";

const validateNewShelf = withValidation(shelfSchema, myShelvesController);
const validateEditShelf = withValidation(shelfSchema, shelfController);
const validateNewUser = withValidation(userSchema, signupFormController);
const validateSignin = withValidation(signinSchema, signinFormController);
const validateAvatar = withValidation(avatarSchema, userController);

const mustBeSignedIn = requireUser(signinRedirectController);
const mustBeSignedOut = requireGuest(alreadySignedInRedirectController);
const mayViewUser = requirePermission(canViewUser, "targetUser");
const mayChangeAvatar = requirePermission(canChangeAvatar, "targetUser");
const mayUseShelf = requirePermission(canUseShelf, "shelf");

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
app.get("/users/:userId", withTargetUser, mayViewUser, userController);
app.get("/users/:userId/avatar", withTargetUser, mayViewUser, avatarController);
app.post("/users/:userId/avatar", mustBeSignedIn, withTargetUser, mayChangeAvatar, validateAvatar, uploadAvatarController);

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

Each `may` guard is one line: a question, and where to find the thing it's about.
The routes don't change at all, because the guards have the same names as before.

`shelfController` asks the same question as `mayUseShelf`, to decide whether you get your own shelf's page or the public one.
Update **server/controllers/shelves.js** to ask `canUseShelf` too.

**server/controllers/shelves.js**
```JavaScript{hl_lines="9 34"}
import {
    createShelf,
    deleteShelfById,
    getShelvesByOwnerId,
    updateShelfTitleById,
} from "../models/shelves.js";
import { addThumbnails, getBooksByShelfId } from "../models/shelfBooks.js";
import { getUserById } from "../models/users.js";
import { canUseShelf } from "../tools/permissions.js";
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
    if (canUseShelf(user, shelf)) {
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

Nothing changes in the browser.
A forged change to someone else's shelf or avatar is still not found, and private profiles are still hidden.

### The pattern, with names {#named-pattern}

Our [permissions pattern][pattern] now has a name for every part.

1. **Load the thing** with a `with` middleware: `withTargetUser`, `withShelf`.
2. **Ask a `can` predicate** in **tools/permissions.js**: `canViewUser`, `canChangeAvatar`, `canUseShelf`.
3. **Use it twice.** In the route, a `may` guard made by `requirePermission` turns away anyone the answer is no for. In the view or the controller, the same predicate decides what to show.

Notice the names.
`requireUser`, `requireGuest` and `requirePermission` are all factories that *make* guards.
The guards they make are named for what they ask.
A `mustBe` guard, such as `mustBeSignedIn`, is about you alone, so it goes first.
A `may` guard is about you *and* a thing, so it goes after the `with` middleware that loads the thing.

## A make profile public button {#publish-button}

The aside on your own profile says whether it's public, but gives you no way to change it.
Update `visibilityForm` in **server/views/user.js**.

**server/views/user.js**
```JavaScript{hl_lines="5 54"}
import { html } from "../tools/html.js";
import { canChangeAvatar, isSelf } from "../tools/permissions.js";
import { avatarImage } from "./partials/avatarImage.js";
import { cardGrid } from "./partials/cardGrid.js";
import { postButton } from "./partials/postButton.js";
import { shelfCard } from "./partials/shelfCard.js";
import { AVATAR_TYPES } from "../schema/avatar.js";

export function userView(ctx) {
    const { data: { targetUser, shelves, errors }, user: currentUser } = ctx;
    // Only you get controls, so only then is there an aside
    const hasControls = isSelf(currentUser, targetUser);

    return html`
        <section>
            <header>
                <h1>${targetUser.username}</h1>
            </header>
            ${avatarImage(targetUser, `Avatar for '${targetUser.username}'`)}
            <h2>Shelves</h2>
            ${cardGrid(shelves.map((shelf) => shelfCard(shelf)), "No shelves yet")}
        </section>
        ${hasControls
            ? html`
                <aside>
                    ${canChangeAvatar(currentUser, targetUser)
                        ? avatarForm(targetUser, errors)
                        : ""}
                    ${visibilityForm(targetUser)}
                </aside>
            `
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

function visibilityForm(user) {
    return html`
        <h2>Profile is ${user.isPublic ? "public" : "private"}</h2>
        <p>${user.isPublic ? "Anyone" : "Only you"} can see your profile</p>
        ${user.isPublic ? "" : postButton(`/users/${user.id}/publish`, "make profile public")}
    `;
}
```

A private profile gets a **make profile public** button, made with `postButton` like the other buttons that change something.
It posts to `/users/:userId/publish`, with your own `id`.
**testUser1** is already public, so their profile has no button.

As **testUser2**, press **make profile public**.
There's no such route yet, so you get the not found page.

## Make your profile public {#publish-profile}

Making a profile public is another permission, so it needs another predicate.
Add `canPublishProfile` to **server/tools/permissions.js**.

**server/tools/permissions.js**
```JavaScript{hl_lines="5"}
export const isSelf = (user, targetUser) => user?.id === targetUser.id;
export const canViewUser = (user, targetUser) =>
    isSelf(user, targetUser) || Boolean(targetUser.isPublic);
export const canChangeAvatar = (user, targetUser) => isSelf(user, targetUser);
export const canPublishProfile = (user, targetUser) => isSelf(user, targetUser);
export const canUseShelf = (user, shelf) => user?.id === shelf.ownerId;
```

Only you can publish your own profile.
That's `isSelf` again, but it's a different question from `canChangeAvatar`, and naming it keeps the rule in one place.

Add `updateUserIsPublicById` to **server/models/users.js**.

**server/models/users.js**
```JavaScript{hl_lines="17-22"}
import { hashPassword } from "../tools/hash.js";
import { db } from "./db.js";

export function getUserByUsername(username) {
    return db.prepare(
        `SELECT * FROM users WHERE username=:username`,
    ).get({ username });
}

// Placeholder: this will need to change when users get more columns
export function getUserById(id) {
    return db.prepare(
        `SELECT id, username, avatarId, isPublic FROM users WHERE id=:id`,
    ).get({ id });
}

export function updateUserIsPublicById(id, isPublic) {
    return db.prepare(
        `UPDATE users SET isPublic=:isPublic WHERE id=:id`,
    ).run({ id, isPublic });
}

export function updateUserAvatarById(id, avatarId) {
    return db.prepare(
        `UPDATE users SET avatarId=:avatarId WHERE id=:id`,
    ).run({ id, avatarId });
}

// Placeholder: this will need to change when users get more columns
export async function createUser({ username, password, isPublic = 0 }) {
    const hashedPassword = await hashPassword(password);
    return db.prepare(`
        INSERT INTO users(username, hashedPassword, isPublic)
        VALUES (:username, :hashedPassword, :isPublic)
        RETURNING *
    `).get({ username, hashedPassword, isPublic });
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

> The placeholders haven't changed.

Add `publishProfileController` to **server/controllers/users.js**.

**server/controllers/users.js**
```JavaScript{hl_lines="1 55-61"}
import { createUser, updateUserAvatarById, updateUserIsPublicById } from "../models/users.js";
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

export function publishProfileController(ctx) {
    const { data: { targetUser } } = ctx;
    updateUserIsPublicById(targetUser.id, 1);
    ctx.flash = "Your profile is now public";
    return redirect("/profile", ctx);
}
```

It makes the profile public, flashes **Your profile is now public**, and redirects to [`/profile`][profile], your own page.

Update **server/main.js** to make `mayPublishProfile` and add the route.

**server/main.js**
```JavaScript{hl_lines="21 39-44 56 85"}
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
    publishProfileController,
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
import { requireGuest, requirePermission, requireUser } from "./middleware/access.js";
import { withBook, withShelf, withTargetUser } from "./middleware/resources.js";
import {
    canChangeAvatar,
    canPublishProfile,
    canUseShelf,
    canViewUser,
} from "./tools/permissions.js";

const validateNewShelf = withValidation(shelfSchema, myShelvesController);
const validateEditShelf = withValidation(shelfSchema, shelfController);
const validateNewUser = withValidation(userSchema, signupFormController);
const validateSignin = withValidation(signinSchema, signinFormController);
const validateAvatar = withValidation(avatarSchema, userController);

const mustBeSignedIn = requireUser(signinRedirectController);
const mustBeSignedOut = requireGuest(alreadySignedInRedirectController);
const mayViewUser = requirePermission(canViewUser, "targetUser");
const mayChangeAvatar = requirePermission(canChangeAvatar, "targetUser");
const mayPublishProfile = requirePermission(canPublishProfile, "targetUser");
const mayUseShelf = requirePermission(canUseShelf, "shelf");

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
app.get("/users/:userId", withTargetUser, mayViewUser, userController);
app.get("/users/:userId/avatar", withTargetUser, mayViewUser, avatarController);
app.post("/users/:userId/avatar", mustBeSignedIn, withTargetUser, mayChangeAvatar, validateAvatar, uploadAvatarController);
app.post("/users/:userId/publish", mustBeSignedIn, withTargetUser, mayPublishProfile, publishProfileController);

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

The route reads from left to right: are you signed in, which user is it, and may you publish them?

The view should ask the same question.
Update **server/views/user.js**.

**server/views/user.js**
```JavaScript{hl_lines="2 29"}
import { html } from "../tools/html.js";
import { canChangeAvatar, canPublishProfile, isSelf } from "../tools/permissions.js";
import { avatarImage } from "./partials/avatarImage.js";
import { cardGrid } from "./partials/cardGrid.js";
import { postButton } from "./partials/postButton.js";
import { shelfCard } from "./partials/shelfCard.js";
import { AVATAR_TYPES } from "../schema/avatar.js";

export function userView(ctx) {
    const { data: { targetUser, shelves, errors }, user: currentUser } = ctx;
    // Only you get controls, so only then is there an aside
    const hasControls = isSelf(currentUser, targetUser);

    return html`
        <section>
            <header>
                <h1>${targetUser.username}</h1>
            </header>
            ${avatarImage(targetUser, `Avatar for '${targetUser.username}'`)}
            <h2>Shelves</h2>
            ${cardGrid(shelves.map((shelf) => shelfCard(shelf)), "No shelves yet")}
        </section>
        ${hasControls
            ? html`
                <aside>
                    ${canChangeAvatar(currentUser, targetUser)
                        ? avatarForm(targetUser, errors)
                        : ""}
                    ${canPublishProfile(currentUser, targetUser) ? visibilityForm(targetUser) : ""}
                </aside>
            `
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

function visibilityForm(user) {
    return html`
        <h2>Profile is ${user.isPublic ? "public" : "private"}</h2>
        <p>${user.isPublic ? "Anyone" : "Only you"} can see your profile</p>
        ${user.isPublic ? "" : postButton(`/users/${user.id}/publish`, "make profile public")}
    `;
}
```

`canPublishProfile` now decides whether the visibility section is in the aside, just as `mayPublishProfile` decides whether the route runs.

As **testUser2**, press **make profile public**.
The aside now says **Profile is public**.
Sign out, and [http://localhost:8000/users/2][user 2] opens.

> Try the [devtools trick][forged form] on the button.
> Run `deno task db:reset`, so that **testUser2** is private again, and sign in as **testUser2**.
> Open the developer tools on your profile, and change the button's form `action` from `/users/2/publish` to `/users/1/publish`.
> Press it, and you get the not found page, because `canPublishProfile` says no.

```plaintext {linenos=false}

============================
POST /users/1/publish
[session] {
  id: "72e6dff7-2734-44b8-85f5-75c50f829286",
  userId: 2,
  expires: "2026-11-25T16:08:17.628Z"
}
[user] { id: 2, username: "testUser2", avatarId: 2, isPublic: 0 }
[error] POST /users/1/publish is not permitted
RESPONSE 404
============================
```

There's no way back to private yet.

## And private again {#unpublish-profile}

Once your profile is public, there's no going back.
A public profile should have a **make profile private** button instead.

Update `visibilityForm` in **server/views/user.js**.

**server/views/user.js**
```JavaScript{hl_lines="51-52 56"}
import { html } from "../tools/html.js";
import { canChangeAvatar, canPublishProfile, isSelf } from "../tools/permissions.js";
import { avatarImage } from "./partials/avatarImage.js";
import { cardGrid } from "./partials/cardGrid.js";
import { postButton } from "./partials/postButton.js";
import { shelfCard } from "./partials/shelfCard.js";
import { AVATAR_TYPES } from "../schema/avatar.js";

export function userView(ctx) {
    const { data: { targetUser, shelves, errors }, user: currentUser } = ctx;
    // Only you get controls, so only then is there an aside
    const hasControls = isSelf(currentUser, targetUser);

    return html`
        <section>
            <header>
                <h1>${targetUser.username}</h1>
            </header>
            ${avatarImage(targetUser, `Avatar for '${targetUser.username}'`)}
            <h2>Shelves</h2>
            ${cardGrid(shelves.map((shelf) => shelfCard(shelf)), "No shelves yet")}
        </section>
        ${hasControls
            ? html`
                <aside>
                    ${canChangeAvatar(currentUser, targetUser)
                        ? avatarForm(targetUser, errors)
                        : ""}
                    ${canPublishProfile(currentUser, targetUser) ? visibilityForm(targetUser) : ""}
                </aside>
            `
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

function visibilityForm(user) {
    const action = user.isPublic ? "unpublish" : "publish";
    const label = user.isPublic ? "make profile private" : "make profile public";
    return html`
        <h2>Profile is ${user.isPublic ? "public" : "private"}</h2>
        <p>${user.isPublic ? "Anyone" : "Only you"} can see your profile</p>
        ${postButton(`/users/${user.id}/${action}`, label)}
    `;
}
```

`visibilityForm` picks the action and the label from `isPublic`, so there's one button that always does the opposite of what the profile is now.

The two controllers would be the same apart from one number and the flash message.
Update **server/controllers/users.js**.

**server/controllers/users.js**
```JavaScript{hl_lines="56 58-59 62-69"}
import { createUser, updateUserAvatarById, updateUserIsPublicById } from "../models/users.js";
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

function changeProfileVisibility(ctx, isPublic) {
    const { data: { targetUser } } = ctx;
    updateUserIsPublicById(targetUser.id, isPublic);
    ctx.flash = isPublic ? "Your profile is now public" : "Your profile is now private";
    return redirect("/profile", ctx);
}

export function publishProfileController(ctx) {
    return changeProfileVisibility(ctx, 1);
}

export function unpublishProfileController(ctx) {
    return changeProfileVisibility(ctx, 0);
}
```

`changeProfileVisibility(ctx, isPublic)` does the work for both.
It isn't exported, because only this module uses it.
`publishProfileController` and `unpublishProfileController` each call it, with `1` or `0`.

Update **server/main.js** to add the route.

**server/main.js**
```JavaScript{hl_lines="23 87"}
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
    publishProfileController,
    signupFormController,
    unpublishProfileController,
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
import { requireGuest, requirePermission, requireUser } from "./middleware/access.js";
import { withBook, withShelf, withTargetUser } from "./middleware/resources.js";
import {
    canChangeAvatar,
    canPublishProfile,
    canUseShelf,
    canViewUser,
} from "./tools/permissions.js";

const validateNewShelf = withValidation(shelfSchema, myShelvesController);
const validateEditShelf = withValidation(shelfSchema, shelfController);
const validateNewUser = withValidation(userSchema, signupFormController);
const validateSignin = withValidation(signinSchema, signinFormController);
const validateAvatar = withValidation(avatarSchema, userController);

const mustBeSignedIn = requireUser(signinRedirectController);
const mustBeSignedOut = requireGuest(alreadySignedInRedirectController);
const mayViewUser = requirePermission(canViewUser, "targetUser");
const mayChangeAvatar = requirePermission(canChangeAvatar, "targetUser");
const mayPublishProfile = requirePermission(canPublishProfile, "targetUser");
const mayUseShelf = requirePermission(canUseShelf, "shelf");

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
app.get("/users/:userId", withTargetUser, mayViewUser, userController);
app.get("/users/:userId/avatar", withTargetUser, mayViewUser, avatarController);
app.post("/users/:userId/avatar", mustBeSignedIn, withTargetUser, mayChangeAvatar, validateAvatar, uploadAvatarController);
app.post("/users/:userId/publish", mustBeSignedIn, withTargetUser, mayPublishProfile, publishProfileController);
app.post("/users/:userId/unpublish", mustBeSignedIn, withTargetUser, mayPublishProfile, unpublishProfileController);

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

The same people can publish and unpublish a profile, so the new route is guarded by `mayPublishProfile` too.

Sign in as **testUser1** (the password is `testUser1!`), and press **make profile private**.
Sign out, and [http://localhost:8000/users/1][user 1] is not found.
Sign in again, make it public, and it's back.

{{<figure src="images/profile-private.webp" alt="testUser2's own profile, with an aside showing the Change avatar form and, below it, Profile is private, Only you can see your profile and a make profile public button" caption="Your own profile says who can see it.">}}

## Conclusion

Every user now has a public or private profile, and new profiles are private.
A private profile and its avatar are not found for everyone but their owner.
Our permission questions have names, in **tools/permissions.js**, and `requirePermission` makes our `may` guards from them, so the route and the view always ask the same question.
You can make your profile public, and private again.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="53"}
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
│   │   ├── permissions.js
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

## Challenge: Who can see whom?

These are questions to investigate, not changes to keep.

- Sign up a new user. Is their profile public or private? Sign out and visit it.
- In **tools/permissions.js**, take the `Boolean(...)` away from `canViewUser`. Does anything change in the browser? What does `canViewUser` return now for a public profile?
- Why does the profile route have no `mustBeSignedIn`, while the publish route does?
- Make **testUser2**'s profile public, and then private again. What happens to the shelves on the home page? Why might that be a problem?

<details>
<summary>The shelves on the home page</summary>

The home page still lists every shelf, including **testUser2**'s, whether their profile is public or not.
When it's private, their name and picture on those shelves lead nowhere.
In the next exercise, shelves get their own setting, and lists only show public ones.

Without `Boolean`, `canViewUser` returns `1` for a public profile, rather than `true`.
`1` counts as true, so the guard behaves the same, but a question that sometimes answers `1` and sometimes `true` is harder to trust.

</details>

> If you tried any changes, undo them, and run `deno task db:reset`, so that your code and data match the starting point of the next exercise.

[avatars]: profiles-and-avatars/avatars/
[next exercise]: privacy/private-shelves/
[may change avatar]: profiles-and-avatars/avatars/#may-change-avatar
[pattern again]: profiles-and-avatars/avatars/#pattern-again
[pattern]: owners-and-guards/owners/#pattern
[forged form]: owners-and-guards/owners/#forged-form
[not found or forbidden]: owners-and-guards/owners/#not-found-or-forbidden
[require user]: owners-and-guards/guards/#require-user
[user 1]: http://localhost:8000/users/1
[user 2]: http://localhost:8000/users/2
[avatar 1]: http://localhost:8000/users/1/avatar
[avatar 2]: http://localhost:8000/users/2/avatar
[profile]: http://localhost:8000/profile
[arrow functions]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions
[boolean]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Boolean/Boolean
[property accessors]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Property_accessors

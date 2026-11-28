---
type: exercise
title: Signing in and out
weight: 2
author: Dr Graeme Stuart
---

In this exercise we will check the username and password when signing in, and remember who signed in.
A cookie that holds the user id will turn out to be unsafe, so we will replace it with a session, add a sign out button, and sign new users in when they sign up.

<!--more-->

> You should have users, hashed passwords and the two test users from the [previous exercise][previous exercise].
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

Signing up creates a real account now, but signing in still ignores the password, and the next page has no idea who you are.
HTTP doesn't remember anything between requests.
By the end of this exercise you should be able to:

- Look up a user and check their password, and give the same `401` for a missing user or a wrong password
- Remember who signed in with a cookie, and read it on every request with middleware
- Explain why a cookie that holds the user id is not safe, and replace it with a session
- Sign out with a button that posts, and sign new users in when they sign up

## Check the password {#check-password}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

The sign in stub ignored what was typed.
`signinController` needs to look the user up and check the password.

Add `getUserByUsername` to **server/models/users.js**.

**server/models/users.js**
```JavaScript{hl_lines="4-9"}
import { hashPassword } from "../tools/hash.js";
import { db } from "./db.js";

export function getUserByUsername(username) {
    return db.prepare(
        `SELECT * FROM users WHERE username=:username`,
    ).get({ username });
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

> The placeholder on `createUser` is unchanged.
> `getUserByUsername` selects every column, because it needs the hash to check the password.

Add `verifyPassword` to **server/tools/hash.js**.

**server/tools/hash.js**
```JavaScript{hl_lines="1-2 27-32"}
import { decodeHex, encodeHex } from "@std/encoding/hex";
import { timingSafeEqual } from "@std/crypto/timing-safe-equal";

const ITERATIONS = 600000;

async function derive(password, salt) {
    const key = await crypto.subtle.importKey(
        "raw",
        new TextEncoder().encode(password),
        "PBKDF2",
        false,
        ["deriveBits"],
    );
    const bits = await crypto.subtle.deriveBits(
        { name: "PBKDF2", hash: "SHA-256", iterations: ITERATIONS, salt },
        key,
        256,
    );
    return new Uint8Array(bits);
}

export async function hashPassword(password) {
    const salt = crypto.getRandomValues(new Uint8Array(16));
    const hash = await derive(password, salt);
    return `${encodeHex(salt)}:${encodeHex(hash)}`;
}

export async function verifyPassword(password, stored) {
    const [salt, hash] = stored.split(":").map(decodeHex);
    const attempt = await derive(password, salt);
    return timingSafeEqual(attempt, hash);
}
```

`verifyPassword` splits the stored string into its salt and hash, runs the typed password through `derive` with the same salt, and compares the two hashes.
It uses `timingSafeEqual` from `@std/crypto`, so the comparison takes the same time however many bytes match, and its timing can't give the hash away.

Add `@std/crypto` to **deno.json**.

**deno.json**
```JavaScript{hl_lines="11"}
{
    "tasks": {
        "serve": "deno -A tasks/server.js",
        "dev": "deno --watch -A tasks/server.js",
        "db:create": "deno -A tasks/db-create.js",
        "db:seed": "deno -A tasks/db-seed.js",
        "db:reset": "deno task db:create && deno task db:seed"
    },
    "imports": {
        "@db/sqlite": "jsr:@db/sqlite@^0.12.0",
        "@std/crypto": "jsr:@std/crypto@^1.1.0",
        "@std/encoding": "jsr:@std/encoding@^1.0.11",
        "@std/html": "jsr:@std/html@^1.0.7",
        "@std/http": "jsr:@std/http@^1.1.4"
    }
}
```

`deno.lock` is updated too.

Update **server/controllers/sessions.js**.

**server/controllers/sessions.js**
```JavaScript{hl_lines="1-2 12-22"}
import { getUserByUsername } from "../models/users.js";
import { verifyPassword } from "../tools/hash.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { signinView } from "../views/signin.js";

export function signinFormController(ctx) {
    ctx.data.title = "Sign in";
    return render(signinView, ctx);
}

export async function signinController(ctx) {
    const { validated: { username, password } } = ctx;
    const user = getUserByUsername(username);
    if (!user || !(await verifyPassword(password, user.hashedPassword))) {
        ctx.data.values = { username };
        ctx.data.errors = { credentials: `Invalid credentials` };
        ctx.status = 401;
        return signinFormController(ctx);
    }
    ctx.flash = `Signed in as '${username}'`;
    return redirect("/", ctx);
}
```

A missing user or a wrong password gives the same `401` and the same **Invalid credentials** message, with the username kept, so the page doesn't say which usernames exist.
The fields themselves were valid, so this is not a `400` from `withValidation`.
It's `401 Unauthorized`: the credentials were not accepted.

Correct details flash **Signed in as '…'** and redirect to the home page.
Nothing remembers the user yet, so the next page doesn't know who signed in.

Update **server/views/signin.js** to show the credentials message above the form.

**server/views/signin.js**
```JavaScript{hl_lines="12 16-18 22-24"}
import { html } from "../tools/html.js";

export function signinView(ctx) {
    const { data: { errors, values } } = ctx;
    return html`
        <section>
            <header>
                <h1>Sign in</h1>
                <p>Don't have an account? <a href="/signup">sign up</a> to gain access.</p>
            </header>
            <form class="secondary" method="POST">
                <span id="credentials-error" class="error">${errors?.credentials ?? ""}</span>
                <label for="username">Username</label>
                <input id="username" name="username" autocomplete="username" value="${values
                    ?.username ?? ""}"
                    aria-describedby="credentials-error username-error" aria-invalid="${Boolean(
                        errors?.username,
                    )}">
                <span id="username-error" class="error">${errors?.username ?? ""}</span>
                <label for="password">Password</label>
                <input id="password" type="password" name="password" autocomplete="current-password"
                    aria-describedby="credentials-error password-error" aria-invalid="${Boolean(
                        errors?.password,
                    )}">
                <span id="password-error" class="error">${errors?.password ?? ""}</span>
                <button>sign in</button>
            </form>
        </section>
    `;
}
```

The message sits in a `span` that both inputs point to with `aria-describedby`, so a screen reader is told about it from either field.
The username is kept, and the password is not.

Open [sign in][signin] as **testUser1** with the wrong password, or with a username that doesn't exist.
You get the same message either way.

{{<figure src="images/invalid-credentials.webp" alt="The sign in page with testUser1 in the username field and Invalid credentials above the form" caption="A missing user and a wrong password get the same message.">}}

```plaintext {linenos=false}

============================
POST /signin
RESPONSE 401
============================
```

Now sign in as **testUser1** with the password `testUser1!`.
You land on the home page with the message **Signed in as 'testUser1'**, but the nav still has a **sign in** link, because nothing has remembered you.

```plaintext {linenos=false}

============================
POST /signin
[flash sent] Signed in as 'testUser1'
RESPONSE 303
============================

============================
GET /
[flash received] Signed in as 'testUser1'
RESPONSE 200
============================
```

## Remember who signed in {#user-id-cookie}

HTTP doesn't remember anything between requests, so signing in has to leave something behind that the browser sends back with every request.
The simplest thing to leave is a cookie that says who you are.
We have already used a [flash cookie][flash cookie] to carry a message across a redirect.

Create **server/tools/authentication.js**.

**server/tools/authentication.js**
```JavaScript{hl_lines="1-12"}
import { setCookie } from "@std/http";

export function signin(userId, ctx) {
    const { headers } = ctx;
    setCookie(headers, {
        name: "userId",
        value: String(userId),
        path: "/",
        httpOnly: true,
        sameSite: "Lax",
    });
}
```

`signin(userId, ctx)` sets a cookie called `userId`, holding the user's id.

- It is [`HttpOnly`][http only], so page scripts can't read it, just like the flash cookie.
- `sameSite: "Lax"` means other sites can't send it with their forms.
- Its `path` is `/`, so the browser sends it to every page.
- It has no expiry date, so the browser keeps it until it's closed.

Update `signinController` to call `signin` once the password checks out.

**server/controllers/sessions.js**
```JavaScript{hl_lines="2 22"}
import { getUserByUsername } from "../models/users.js";
import { signin } from "../tools/authentication.js";
import { verifyPassword } from "../tools/hash.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { signinView } from "../views/signin.js";

export function signinFormController(ctx) {
    ctx.data.title = "Sign in";
    return render(signinView, ctx);
}

export async function signinController(ctx) {
    const { validated: { username, password } } = ctx;
    const user = getUserByUsername(username);
    if (!user || !(await verifyPassword(password, user.hashedPassword))) {
        ctx.data.values = { username };
        ctx.data.errors = { credentials: `Invalid credentials` };
        ctx.status = 401;
        return signinFormController(ctx);
    }
    signin(user.id, ctx);
    ctx.flash = `Signed in as '${username}'`;
    return redirect("/", ctx);
}
```

Sign in as **testUser1** again.
In the browser's developer tools, open the **application** panel (**storage** in Firefox), choose **cookies**, then `http://localhost:8000`.
The `userId` cookie is there, with the value `1`, and a tick in its **HttpOnly** column.

Nothing reads the cookie yet, so the pages still look signed out.

## Read the cookie on every request {#with-session}

The browser now sends the `userId` cookie with every request, so the server can work out who's signed in.

Add `getUserById` to **server/models/users.js**.

**server/models/users.js**
```JavaScript{hl_lines="10-14"}
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

> Both functions that write or read a user's row still have the placeholder for extra columns.
> `getUserById` returns only the id and the username, never the password hash.

Create **server/middleware/session.js**.

**server/middleware/session.js**
```JavaScript{hl_lines="1-9"}
import { getCookies } from "@std/http";
import { getUserById } from "../models/users.js";

export function withSession(ctx, next) {
    const { request } = ctx;
    const { userId } = getCookies(request.headers);
    ctx.user = userId && getUserById(userId);
    return next(ctx);
}
```

`withSession` reads the cookie with `getCookies` and puts that user on `ctx.user`.
With no cookie, or an id that matches no user, `ctx.user` is empty.

Update **server/main.js** to run `withSession` on every request, after `withLogs` and `withErrors`.

**server/main.js**
```JavaScript{hl_lines="21 33"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import { addBookController, removeBookController } from "./controllers/shelfBooks.js";
import {
    createShelfController,
    deleteShelfController,
    editShelfController,
    shelfController,
    shelvesController,
} from "./controllers/shelves.js";
import { staticController } from "./controllers/static.js";
import { Router } from "./tools/router.js";
import { withLogs } from "./middleware/logging.js";
import { withErrors } from "./middleware/errors.js";
import { withValidation } from "./middleware/validation.js";
import { shelfSchema } from "./schema/shelf.js";
import { createUserController, signupFormController } from "./controllers/users.js";
import { userSchema } from "./schema/user.js";
import { signinController, signinFormController } from "./controllers/sessions.js";
import { signinSchema } from "./schema/signin.js";
import { withSession } from "./middleware/session.js";
import { withBook, withShelf } from "./middleware/resources.js";

const validateNewShelf = withValidation(shelfSchema, shelvesController);
const validateEditShelf = withValidation(shelfSchema, shelfController);
const validateNewUser = withValidation(userSchema, signupFormController);
const validateSignin = withValidation(signinSchema, signinFormController);

const app = new Router();

app.use(withLogs);
app.use(withErrors);
app.use(withSession);

app.get("/assets/*", staticController);
app.get("/", homeController);
app.get("/books", booksController);
app.get("/books/:bookId", withBook, bookController);
app.get("/signup", signupFormController);
app.post("/signup", validateNewUser, createUserController);
app.get("/signin", signinFormController);
app.post("/signin", validateSignin, signinController);
app.get("/shelves", shelvesController);
app.post("/shelves", validateNewShelf, createShelfController);
app.get("/shelves/:shelfId", withShelf, shelfController);
app.post("/shelves/:shelfId/edit", withShelf, validateEditShelf, editShelfController);
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

The layout can now use `ctx.user`.
Update **server/views/layout.js**.

**server/views/layout.js**
```JavaScript{hl_lines="4 23 31 45-53"}
import { html } from "../tools/html.js";

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
                        ${navLink("/shelves", "shelves", url.pathname)}
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
    return "";
}

// A header link, marked as the current page when it matches the path exactly
function navLink(href, content, path) {
    return html`<a href="${href}" aria-current="${
        href === path ? "page" : "false"
    }">${content}</a>`;
}
```

The placeholder comment from the last unit is gone, because signing in is real now.
When someone's signed in, the footer says **Signed in as '…'**, and `authLinks` gives the **sign in** link only to visitors who aren't signed in.

Update **server/middleware/logging.js** so the terminal shows who made each request.

**server/middleware/logging.js**
```JavaScript{hl_lines="9"}
import { UnexpectedError } from "../tools/errors.js";

// Placeholder: this will need to change when we add sessions
export async function withLogs(ctx, next) {
    const { url, request } = ctx;
    console.log("\n============================");
    console.log(`${request.method} ${url.pathname}`);
    const response = await next(ctx);
    console.log("[user]", ctx.user);
    if (ctx.flash) console.log("[flash sent]", ctx.flash);
    if (ctx.data.flash) console.log("[flash received]", ctx.data.flash);
    if (ctx.error instanceof UnexpectedError) console.log("[error]", ctx.error);
    else if (ctx.error) console.log("[error]", ctx.error.message);
    console.log("RESPONSE", response.status);
    console.log("============================");

    return response;
}
```

> The placeholder is there because this log will need to change when we add sessions.

Sign in as **testUser1** again.
The home page's footer says **Signed in as 'testUser1'**, and the **sign in** link has left the nav.

{{<figure src="images/signed-in.webp" alt="The home page for testUser1, with no sign in link in the nav, the message Signed in as 'testUser1', and Signed in as 'testUser1' in the footer" caption="The page now knows who signed in.">}}

```plaintext {linenos=false}

============================
POST /signin
[user] undefined
[flash sent] Signed in as 'testUser1'
RESPONSE 303
============================

============================
GET /
[user] { id: 1, username: "testUser1" }
[flash received] Signed in as 'testUser1'
RESPONSE 200
============================
```

The `POST` has no user yet, because `withSession` ran before `signin` set the cookie.
The next request, the `GET` after the redirect, has the cookie, and the log shows **testUser1**.

There's no way to sign out yet.
Closing the browser forgets the cookie, because it has no expiry date.

## A session, not a user id {#sessions}

The `userId` cookie trusts the browser to say who it is, and the browser belongs to the user.

> Try this on the seed data, then we will replace the cookie.

1. Sign in as **testUser1** (the password is `testUser1!`).
2. Open the developer tools (**F12**), go to the **application** panel (**storage** in Firefox), and choose **cookies**, then `http://localhost:8000`.
3. The `userId` cookie's value is `1`. Double-click the value, change it to `2`, and press **Enter**.
4. Reload the page.

The footer now says **Signed in as 'testUser2'**.
Anyone can be anyone.

The fix is a session: a row in a new sessions table, and a cookie holding only its id.
Update **tasks/db-create.js**.

**tasks/db-create.js**
```JavaScript{hl_lines="3 44-52"}
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

db.exec(`
    CREATE TABLE sessions (
        id TEXT PRIMARY KEY,
        userId INTEGER NOT NULL,
        expires TEXT NOT NULL,
        FOREIGN KEY (userId) REFERENCES users(id) ON DELETE CASCADE
    )
`);
```

> The users and shelves placeholders are unchanged.

The sessions table is dropped first, and created after the users table, because each session points at a user.
Its `id` is a random [UUID][uuid], so it can't be guessed, and `userId` points at the user, so deleting a user deletes their sessions.
`expires` is `TEXT` holding an ISO date string.

Create **server/models/sessions.js**.

**server/models/sessions.js**
```JavaScript{hl_lines="1-16"}
import { db } from "./db.js";

export function createSession(userId, expires) {
    const id = crypto.randomUUID();
    return db.prepare(`
        INSERT INTO sessions(id, userId, expires)
        VALUES (:id, :userId, :expires)
        RETURNING *;
    `).get({ id, userId, expires: new Date(expires).toISOString() });
}

export function getSessionById(id) {
    return db.prepare(
        `SELECT * FROM sessions WHERE id=:id AND expires > :now`,
    ).get({ id, now: new Date().toISOString() });
}
```

`createSession` stores `new Date(expires).toISOString()`, and `getSessionById` only returns a session whose expiry is later than now, as an ISO string too.
ISO strings in the same format sort in time order, so SQLite can compare them as text.

Update **server/tools/authentication.js**.

**server/tools/authentication.js**
```JavaScript{hl_lines="1-4 7-8 11-12 16 19-24"}
import { getCookies, setCookie } from "@std/http";
import { createSession, getSessionById } from "../models/sessions.js";

const SESSION_LENGTH = 1000 * 60 * 60 * 24;

export function signin(userId, ctx) {
    const expires = Date.now() + SESSION_LENGTH;
    const session = createSession(userId, expires);
    const { headers } = ctx;
    setCookie(headers, {
        name: "sessionId",
        value: session.id,
        path: "/",
        httpOnly: true,
        sameSite: "Lax",
        maxAge: SESSION_LENGTH / 1000,
    });
}

export function currentSession(ctx) {
    const { request } = ctx;
    const { sessionId } = getCookies(request.headers);
    return sessionId && getSessionById(sessionId);
}
```

`signin` now creates a session that lasts a day, `SESSION_LENGTH`, and sets a `sessionId` cookie with the same lifetime instead of the `userId` cookie.
The cookie has a `maxAge` of one day, so the browser keeps it even if it's closed, unlike the `userId` cookie, which lasted only until the browser was closed.

`currentSession` reads the cookie and looks the session up.

Update **server/middleware/session.js** to put the session on `ctx.session` and its user on `ctx.user`.

**server/middleware/session.js**
```JavaScript{hl_lines="1 5-6"}
import { currentSession } from "../tools/authentication.js";
import { getUserById } from "../models/users.js";

export function withSession(ctx, next) {
    ctx.session = currentSession(ctx);
    ctx.user = ctx.session && getUserById(ctx.session.userId);
    return next(ctx);
}
```

Update **server/middleware/logging.js** to print the session as well as the user.

**server/middleware/logging.js**
```JavaScript{hl_lines="8"}
import { UnexpectedError } from "../tools/errors.js";

export async function withLogs(ctx, next) {
    const { url, request } = ctx;
    console.log("\n============================");
    console.log(`${request.method} ${url.pathname}`);
    const response = await next(ctx);
    console.log("[session]", ctx.session);
    console.log("[user]", ctx.user);
    if (ctx.flash) console.log("[flash sent]", ctx.flash);
    if (ctx.data.flash) console.log("[flash received]", ctx.data.flash);
    if (ctx.error instanceof UnexpectedError) console.log("[error]", ctx.error);
    else if (ctx.error) console.log("[error]", ctx.error.message);
    console.log("RESPONSE", response.status);
    console.log("============================");

    return response;
}
```

The placeholder is gone.

Rebuild the database, so the sessions table is created.

```bash {linenos=false}
deno task db:reset
```

Sign in as **testUser1** again.
The footer still says **Signed in as 'testUser1'**, and the **application** panel now shows a `sessionId` cookie, a UUID, with a **Max-Age** of 86400 seconds.
The terminal shows the session and the user.

```plaintext {linenos=false}

============================
POST /signin
[session] undefined
[user] undefined
[flash sent] Signed in as 'testUser1'
RESPONSE 303
============================

============================
GET /
[session] {
  id: "e3ab1b91-e33a-485a-8745-96aa13131425",
  userId: 1,
  expires: "2026-11-29T10:10:31.905Z"
}
[user] { id: 1, username: "testUser1" }
[flash received] Signed in as 'testUser1'
RESPONSE 200
============================
```

Your session's id and expiry will be different, because the id is random and the expiry is a day from now.

Try the same devtools trick on the `sessionId` cookie: change its value to some other id and reload.
The pages look signed out, because that id matches no session.

## Sign out {#sign-out}

Once signed in, the nav needs a way out.
Update `authLinks` in **server/views/layout.js**.

**server/views/layout.js**
```JavaScript{hl_lines="2 52"}
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
                        ${navLink("/shelves", "shelves", url.pathname)}
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
    return postButton("/signout", "sign out");
}

// A header link, marked as the current page when it matches the path exactly
function navLink(href, content, path) {
    return html`<a href="${href}" aria-current="${
        href === path ? "page" : "false"
    }">${content}</a>`;
}
```

`authLinks` now gives a signed in user a **sign out** button, using the `postButton` partial, in place of the **sign in** link.

It's a button in a form that posts to `/signout` rather than a link, because signing out changes something on the server.
A link would be a `GET`, which a browser or a link preview might follow without asking.

Add `deleteSessionById` to **server/models/sessions.js**.

**server/models/sessions.js**
```JavaScript{hl_lines="17-20"}
import { db } from "./db.js";

export function createSession(userId, expires) {
    const id = crypto.randomUUID();
    return db.prepare(`
        INSERT INTO sessions(id, userId, expires)
        VALUES (:id, :userId, :expires)
        RETURNING *;
    `).get({ id, userId, expires: new Date(expires).toISOString() });
}

export function getSessionById(id) {
    return db.prepare(
        `SELECT * FROM sessions WHERE id=:id AND expires > :now`,
    ).get({ id, now: new Date().toISOString() });
}

export function deleteSessionById(id) {
    db.prepare(`DELETE FROM sessions WHERE id=:id`).run({ id });
}
```

Add `signout` to **server/tools/authentication.js**.

**server/tools/authentication.js**
```JavaScript{hl_lines="1-2 25-31"}
import { deleteCookie, getCookies, setCookie } from "@std/http";
import { createSession, deleteSessionById, getSessionById } from "../models/sessions.js";

const SESSION_LENGTH = 1000 * 60 * 60 * 24;

export function signin(userId, ctx) {
    const expires = Date.now() + SESSION_LENGTH;
    const session = createSession(userId, expires);
    const { headers } = ctx;
    setCookie(headers, {
        name: "sessionId",
        value: session.id,
        path: "/",
        httpOnly: true,
        sameSite: "Lax",
        maxAge: SESSION_LENGTH / 1000,
    });
}

export function currentSession(ctx) {
    const { request } = ctx;
    const { sessionId } = getCookies(request.headers);
    return sessionId && getSessionById(sessionId);
}

export function signout(ctx) {
    if (!ctx.session) return;
    deleteSessionById(ctx.session.id);
    const { headers } = ctx;
    deleteCookie(headers, "sessionId", { path: "/" });
}
```

`signout` deletes the session row with `deleteSessionById`, and deletes the `sessionId` cookie, so the old id is useless even if it's kept somewhere.
If you're not signed in, it does nothing.

Add `signoutController` to **server/controllers/sessions.js**.

**server/controllers/sessions.js**
```JavaScript{hl_lines="2 26-31"}
import { getUserByUsername } from "../models/users.js";
import { signin, signout } from "../tools/authentication.js";
import { verifyPassword } from "../tools/hash.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { signinView } from "../views/signin.js";

export function signinFormController(ctx) {
    ctx.data.title = "Sign in";
    return render(signinView, ctx);
}

export async function signinController(ctx) {
    const { validated: { username, password } } = ctx;
    const user = getUserByUsername(username);
    if (!user || !(await verifyPassword(password, user.hashedPassword))) {
        ctx.data.values = { username };
        ctx.data.errors = { credentials: `Invalid credentials` };
        ctx.status = 401;
        return signinFormController(ctx);
    }
    signin(user.id, ctx);
    ctx.flash = `Signed in as '${username}'`;
    return redirect("/", ctx);
}

export function signoutController(ctx) {
    signout(ctx);
    ctx.flash = "Signed out";
    return redirect("/", ctx);
}
```

The controller calls `signout`, flashes **Signed out**, and redirects to the home page, where the **sign in** link is back.

Update **server/main.js** to add the route.

**server/main.js**
```JavaScript{hl_lines="19-23 47"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import { addBookController, removeBookController } from "./controllers/shelfBooks.js";
import {
    createShelfController,
    deleteShelfController,
    editShelfController,
    shelfController,
    shelvesController,
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
    signinController,
    signinFormController,
    signoutController,
} from "./controllers/sessions.js";
import { signinSchema } from "./schema/signin.js";
import { withSession } from "./middleware/session.js";
import { withBook, withShelf } from "./middleware/resources.js";

const validateNewShelf = withValidation(shelfSchema, shelvesController);
const validateEditShelf = withValidation(shelfSchema, shelfController);
const validateNewUser = withValidation(userSchema, signupFormController);
const validateSignin = withValidation(signinSchema, signinFormController);

const app = new Router();

app.use(withLogs);
app.use(withErrors);
app.use(withSession);

app.get("/assets/*", staticController);
app.get("/", homeController);
app.get("/books", booksController);
app.get("/books/:bookId", withBook, bookController);
app.get("/signup", signupFormController);
app.post("/signup", validateNewUser, createUserController);
app.get("/signin", signinFormController);
app.post("/signin", validateSignin, signinController);
app.post("/signout", signoutController);
app.get("/shelves", shelvesController);
app.post("/shelves", validateNewShelf, createShelfController);
app.get("/shelves/:shelfId", withShelf, shelfController);
app.post("/shelves/:shelfId/edit", withShelf, validateEditShelf, editShelfController);
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

Sign in as **testUser1**, then press **sign out**.
You land on the home page with the message **Signed out**, the **sign in** link is back, and the footer no longer says who you are.

{{<figure src="images/sign-out-button.webp" alt="The home page for testUser1, with a sign out button in the nav, the message Signed in as 'testUser1', and Signed in as 'testUser1' in the footer" caption="The nav now has a way out.">}}

{{<figure src="images/signed-out.webp" alt="The home page after signing out, with a sign in link in the nav and the message Signed out" caption="Signing out forgets the session.">}}

```plaintext {linenos=false}

============================
POST /signout
[session] {
  id: "b43aaa4c-7cf5-451c-8bca-8c64c9af725b",
  userId: 1,
  expires: "2026-11-29T10:10:33.483Z"
}
[user] { id: 1, username: "testUser1" }
[flash sent] Signed out
RESPONSE 303
============================

============================
GET /
[session] undefined
[user] undefined
[flash received] Signed out
RESPONSE 200
============================
```

Signing out when you're not signed in still redirects to the home page, and still flashes **Signed out**.

## Sign new users in {#welcome}

Having just chosen a username and a password, a new user had to type them again on the sign in page.
Update **server/controllers/users.js**.

**server/controllers/users.js**
```JavaScript{hl_lines="2 15-17"}
import { createUser } from "../models/users.js";
import { signin } from "../tools/authentication.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { signupView } from "../views/signup.js";

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
```

`createUserController` now signs them in straight away with `signin`, as soon as `createUser` gives back the new row.
It welcomes them in a flash and redirects to the home page, just as signing in does.

Open [sign up][signup], register as **alice** with the password `Alice123!`, and you land on the home page signed in, with the message **Welcome, alice**.

{{<figure src="images/welcome.webp" alt="The home page after signing up as alice, with a sign out button in the nav, the message Welcome, alice, and Signed in as 'alice' in the footer" caption="A new user is signed in as soon as they sign up.">}}

```plaintext {linenos=false}

============================
POST /signup
[session] undefined
[user] undefined
[flash sent] Welcome, alice
RESPONSE 303
============================

============================
GET /
[session] {
  id: "0b7de405-71ac-4f1d-91dd-c87d749e3fc7",
  userId: 3,
  expires: "2026-11-29T10:10:35.541Z"
}
[user] { id: 3, username: "alice" }
[flash received] Welcome, alice
RESPONSE 200
============================
```

**alice** is user `3`, because **testUser1** and **testUser2** were already seeded.

## Conclusion

Signing in now checks the username and the password, and a missing user or a wrong password gets the same `401` and the same message, so the page doesn't reveal which usernames exist.
We first remembered who signed in with a `userId` cookie, and saw that anyone can edit a cookie and become someone else.
A session stores the user id on the server, and the cookie holds only a random id that can't be guessed.
You can sign out, which deletes the session and the cookie, and a new user is signed in as soon as they sign up.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="27 32 41"}
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
│   │   ├── shelf.js
│   │   ├── shelves.js
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

## Challenge: Who do you trust?

These are questions to investigate, not changes to keep.

- Why do a missing username and a wrong password get the same message and the same `401`?
- After signing in, change the `sessionId` cookie's value to some invented id and reload. What happens? What happens if you put `1` in it, as we did with `userId`?
- Why is **sign out** a button that posts, rather than a link?

<details>
<summary>The same message, a useless cookie, and a POST</summary>

If a missing username and a wrong password gave different messages, anyone could find out which usernames have an account by trying them on the sign in form.
The same `401` and the same **Invalid credentials** message hide that.

An invented `sessionId`, or the value `1`, matches no session, so `currentSession` returns nothing, `ctx.user` is empty, and the pages look signed out.
The user id no longer lives in the cookie, so putting `1` in it doesn't make you **testUser1**.

A link is a `GET`.
Signing out deletes a session on the server, and a browser, a crawler or a link preview might follow a `GET` without asking.
A button in a form that posts only runs when the user presses it.

</details>

> If you tried any changes, undo them, and run `deno task db:reset`, so that your code and data match the starting point of the next unit.

[previous exercise]: authentication/signing-up/
[flash cookie]: post-redirect-flash/cookies-and-flash/#set-flash
[http only]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie#httponly
[uuid]: https://developer.mozilla.org/en-US/docs/Glossary/UUID
[signup]: http://localhost:8000/signup
[signin]: http://localhost:8000/signin
[home]: http://localhost:8000/

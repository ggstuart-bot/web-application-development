---
type: exercise
title: Guards
weight: 1
author: Dr Graeme Stuart
---

In this exercise we will stop visitors who aren't signed in from changing shelves.
We will start with a simple check in one controller, then turn it into a *guard*, a middleware that protects any route we put it on.

<!--more-->

> You should be able to sign up, sign in and sign out, with a session, from the [previous exercise][previous exercise].
> Your database should hold the seeded users, **testUser1** and **testUser2**. If you have been experimenting, run `deno task db:reset` to get back to the seed data.
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

We know who is signed in, but we don't do anything with it yet.
Anyone can make a shelf, rename it or delete it, signed in or not.
By the end of this exercise you should be able to:

- Check for a signed-in user in a controller, and send a visitor to the sign in page instead
- Write a *guard*, a middleware that decides whether the rest of a route runs
- Make a guard with a *factory* function that takes a fallback controller
- Explain why a guard goes first in a route's list of handlers
- Send signed-in users away from the sign up and sign in pages

## Ask visitors to sign in {#sign-in-check}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Make sure you are signed out, then visit [http://localhost:8000/shelves][shelves] and make a new shelf.
It works, and the shelf belongs to nobody.
Now that we know who is signed in, we can stop that.

Update **server/controllers/shelves.js** so that `createShelfController` checks for a user first.

**server/controllers/shelves.js**
```JavaScript{hl_lines="20-24"}
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
    if (!user) {
        ctx.flash = "Please sign in to continue";
        return redirect("/signin", ctx);
    }
    const shelf = createShelf({ title: validated.title });
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

- `ctx.user` is the signed-in user that `withSession` found, or `undefined` if nobody is signed in.
- With nobody signed in, we set a flash message and redirect to the sign in page, just as `signoutController` redirects home. The `return` means the shelf is never created.
- Otherwise, the controller carries on as before.

Sign out, go back to [http://localhost:8000/shelves][shelves] and try to make a shelf called **poetry**.
This time you land on the sign in page, with the message **Please sign in to continue**.
The terminal shows the redirect.

```plaintext {linenos=false}

============================
POST /shelves
[session] undefined
[user] undefined
[flash sent] Please sign in to continue
RESPONSE 303
============================
```

Try making a shelf with an empty title too.
You see the form's error message, not the sign in page.
`validateNewShelf` runs before `createShelfController`, so the title is checked before we ever ask who you are.

The check only guards this one controller.
Still signed out, open [http://localhost:8000/shelves/1][shelf 1].
You can rename the shelf, delete it, and remove its books, and you can add books to it from any book's page.

We could copy the same `if` into every controller that changes a shelf.
That would work, but there's a better way.

## A guard {#require-user}

The same check belongs in front of every route that changes a shelf.
When the same thing needs to happen on many routes, we make it a middleware, just like `withShelf` and `withValidation`.

Create a new file, **server/middleware/access.js**, for our first *guard*.

**server/middleware/access.js**
```JavaScript{hl_lines="1-6"}
export function requireUser(fallback) {
    return (ctx, next) => {
        if (ctx.user) return next(ctx);
        return fallback(ctx);
    };
}
```

`requireUser` is a *factory*: it doesn't guard anything itself, it makes a middleware and returns it.
This is the same shape as `withValidation`, which takes a schema and a fallback, and returns the middleware that uses them.

- If someone is signed in, the middleware calls `next(ctx)`, and the rest of the route runs.
- If not, it never calls `next`. The `fallback` controller answers instead.

That is what makes a guard: it decides whether the rest of the route runs at all.
The guard doesn't know what the fallback does, so we can choose a different fallback for each guard we make.

Our fallback will do what the inline check did.
Add `signinRedirectController` to the end of **server/controllers/sessions.js**.

**server/controllers/sessions.js**
```JavaScript{hl_lines="32-36"}
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

export function signinRedirectController(ctx) {
    ctx.flash = "Please sign in to continue";
    return redirect("/signin", ctx);
}
```

Now update **server/main.js** to make the guard, and put it on every route that changes a shelf.

**server/main.js**
```JavaScript{hl_lines="22 27 35-36 53 55-58"}
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
    signinRedirectController,
    signoutController,
} from "./controllers/sessions.js";
import { signinSchema } from "./schema/signin.js";
import { withSession } from "./middleware/session.js";
import { requireUser } from "./middleware/access.js";
import { withBook, withShelf } from "./middleware/resources.js";

const validateNewShelf = withValidation(shelfSchema, shelvesController);
const validateEditShelf = withValidation(shelfSchema, shelfController);
const validateNewUser = withValidation(userSchema, signupFormController);
const validateSignin = withValidation(signinSchema, signinFormController);

const mustBeSignedIn = requireUser(signinRedirectController);

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

- We import `signinRedirectController` and `requireUser`.
- `mustBeSignedIn` is the guard, made from the two. We name it once, next to the validation middleware, and use it on many routes.
- It goes on the routes that create, rename and delete a shelf, and add or remove a book. Reading pages stays open to everyone.

Order matters.
The router runs a route's handlers from left to right, so `mustBeSignedIn` runs first.
A visitor who isn't signed in is sent to sign in before we load the shelf or read the form, whatever they sent.

The inline check isn't needed any more.
Remove it from `createShelfController` in **server/controllers/shelves.js**.

**server/controllers/shelves.js**
```JavaScript{hl_lines="20"}
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
    const { validated } = ctx;
    const shelf = createShelf({ title: validated.title });
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

The controller goes back to just creating the shelf.

Signed out, open [http://localhost:8000/shelves/1][shelf 1] again and try to rename it.
This time you are asked to sign in, and the shelf keeps its name.
The same happens for deleting the shelf and removing a book, and for the add buttons on a book's page, such as [http://localhost:8000/books/1][book 1].
Now try an empty title for a new shelf: you are asked to sign in, because the guard runs before the validation.

Sign in as **testUser1** (the password is `testUser1!`), and everything works just as before.

## Signed-in users {#require-guest}

Sign in, then visit [http://localhost:8000/signin][signin] or [http://localhost:8000/signup][signup].
You get the forms, though you have no reason to use them.
Signing in again on top of a session would only muddle things.

We need the opposite guard, which only lets a route run when nobody is signed in.
Add `requireGuest` to **server/middleware/access.js**.

**server/middleware/access.js**
```JavaScript{hl_lines="7-13"}
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
```

`requireGuest` is the mirror image of `requireUser`.
The only difference is the `!`.

Its fallback tells the user they are already signed in.
Add `alreadySignedInRedirectController` to the end of **server/controllers/sessions.js**.

**server/controllers/sessions.js**
```JavaScript{hl_lines="37-42"}
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

export function signinRedirectController(ctx) {
    ctx.flash = "Please sign in to continue";
    return redirect("/signin", ctx);
}

export function alreadySignedInRedirectController(ctx) {
    const { user: { username } } = ctx;
    ctx.flash = `You are already signed in as '${username}'`;
    return redirect("/", ctx);
}
```

This controller only ever runs when someone is signed in, so `ctx.user` is always there, and we can take the `username` straight out of it.

Update **server/main.js** to make `mustBeSignedOut`, and put it first on the sign up and sign in routes.

**server/main.js**
```JavaScript{hl_lines="20 28 37 49-52"}
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

const validateNewShelf = withValidation(shelfSchema, shelvesController);
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
app.get("/shelves", shelvesController);
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

The guard goes on both the `GET` and the `POST` routes.
Hiding a form is not enough, because anyone can send a `POST` request without it.
Signing out stays open to anyone.

Signed in, click [http://localhost:8000/signin][signin].
You land on the home page with the message **You are already signed in as 'testUser1'**.

```plaintext {linenos=false}

============================
GET /signin
[session] {
  id: "df973b89-dea4-4032-8cd3-9006baee6fd2",
  userId: 1,
  expires: "2026-11-25T15:51:43.035Z"
}
[user] { id: 1, username: "testUser1" }
[flash sent] You are already signed in as 'testUser1'
RESPONSE 303
============================
```

> Your session id and expiry date will be different.

Sign out, and the sign up and sign in pages work again.

Notice that our two guards only ever look at `ctx.user`.
They are about *you*: whether you are signed in or not.
They don't need to know anything about the shelf or the book a route is about, so they can go first, before anything else is loaded.
In the [next exercise][owners] we will need guards that ask about a particular shelf as well.

## Conclusion

We started with an inline check in one controller, then turned it into a guard.
`requireUser` and `requireGuest` make middleware that either lets the rest of the route run, or hands over to a fallback controller.
`mustBeSignedIn` protects every route that changes a shelf, and `mustBeSignedOut` keeps signed-in users away from the sign up and sign in forms.
Each guard goes first on its routes, because it only needs to know who you are.

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

## Challenge: Guards in order

These are questions to investigate, not changes to keep.

- Move `mustBeSignedIn` after `validateNewShelf` on the `POST /shelves` route. Signed out, make a shelf with an empty title, and then with a good one. What happens each time, and why?
- Signed out, visit [http://localhost:8000/shelves/999][shelf 999], a shelf that doesn't exist. Which handler answers? If a visitor who isn't signed in sent a rename for shelf 999, would they get the sign in page or the not found page? What if `mustBeSignedIn` came after `withShelf`?
- Signed out, open the **network** panel and make a shelf. What is the status of the response, and where does the browser go next?
- Why don't we guard `POST /signout`? What happens if a visitor who isn't signed in signs out?

> If you tried any changes, undo them, so that your code matches the starting point of the next exercise.

[shelves]: http://localhost:8000/shelves
[shelf 1]: http://localhost:8000/shelves/1
[book 1]: http://localhost:8000/books/1
[signin]: http://localhost:8000/signin
[signup]: http://localhost:8000/signup
[owners]: owners-and-guards/owners/
[previous exercise]: authentication/signing-in-and-out/
[shelf 999]: http://localhost:8000/shelves/999

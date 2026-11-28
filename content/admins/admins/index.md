---
type: exercise
title: Admins
weight: 1
author: Dr Graeme Stuart
---

In this exercise we will keep our promise that a public shelf always has a public owner, and answer with `403 Forbidden` for the first time.
Then we will add an admin user, and a users page that only admins can open.

<!--more-->

> You should have private profiles and private shelves from the [previous unit][private shelves].
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

At the end of the last unit, we saw that our app breaks one of its own rules: a private user can share a shelf, and its owner link leads nowhere.
We also said that one day we would answer `403 Forbidden` instead of `404 Not Found`.
By the end of this exercise you should be able to:

- Show a button only when a predicate says yes, and tell the user what to do when it says no
- Throw a new kind of error, and answer it with its own status and page
- Choose between `404 Not Found` and `403 Forbidden`
- Keep a rule about our data in the code that changes it
- Add a flag that gives some users more power, and a guard about your status

## Only a public owner may share {#can-publish-shelf}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

The [rule from the last unit][honest data] is that a public shelf should always have a public owner.
So you can share a shelf if it's yours, *and* your own profile is public.
That's a new question.
Add `canPublishShelf` to **server/tools/permissions.js**.

**server/tools/permissions.js**
```JavaScript{hl_lines="8"}
export const isSelf = (user, targetUser) => user?.id === targetUser.id;
export const canViewUser = (user, targetUser) =>
    isSelf(user, targetUser) || Boolean(targetUser.isPublic);
export const canChangeAvatar = (user, targetUser) => isSelf(user, targetUser);
export const canPublishProfile = (user, targetUser) => isSelf(user, targetUser);
const isOwner = (user, resource) => user?.id === resource.ownerId;
export const canUseShelf = (user, shelf) => isOwner(user, shelf);
export const canPublishShelf = (user, shelf) => isOwner(user, shelf) && Boolean(user.isPublic);
export const canViewShelf = (user, shelf) => isOwner(user, shelf) || Boolean(shelf.isPublic);
```

It's built from `isOwner`, like `canUseShelf`, with the owner's own `isPublic` added.

The pages should only offer a **make public** button when `canPublishShelf` says yes.
`visibilityButton` stays as it is: it only makes the button, and the pages decide whether to show it.
Update **server/views/myShelves.js**.

**server/views/myShelves.js**
```JavaScript{hl_lines="2 9 28-30"}
import { html } from "../tools/html.js";
import { canPublishShelf } from "../tools/permissions.js";
import { cardGrid } from "./partials/cardGrid.js";
import { shelfCard } from "./partials/shelfCard.js";
import { shelfTitleForm } from "./partials/shelfTitleForm.js";
import { visibilityButton } from "./partials/visibilityButton.js";

export function myShelvesView(ctx) {
    const { user, data: { shelves, errors, values } } = ctx;
    const publicShelves = shelves.filter((shelf) => shelf.isPublic);
    const privateShelves = shelves.filter((shelf) => !shelf.isPublic);

    return html`
        <section>
            <header>
                <h1>My shelves</h1>
                <p>Make a shelf, then add books to it from any book's page.</p>
            </header>
            <h2>Public shelves</h2>
            <p>Anyone can see these shelves</p>
            ${cardGrid(
                publicShelves.map((shelf) => shelfCard(shelf, visibilityButton(shelf))),
                "No public shelves yet",
            )}
            <h2>Private shelves</h2>
            <p>Only you can see these shelves</p>
            ${cardGrid(
                privateShelves.map((shelf) =>
                    shelfCard(shelf, canPublishShelf(user, shelf) ? visibilityButton(shelf) : "")
                ),
                "No private shelves yet",
            )}
        </section>
        <aside>
            <h2>New shelf</h2>
            ${shelfTitleForm("/shelves", values?.title ?? "", errors?.title, "create")}
        </aside>
    `;
}
```

My shelves takes the signed-in user from `ctx`, and only gives a private shelf's card a button if `canPublishShelf` says yes.
A public shelf always keeps its **make private** button, because taking a shelf back never breaks the rule.

Update **server/views/myShelf.js** to do the same.

**server/views/myShelf.js**
```JavaScript{hl_lines="2 10 33"}
import { html } from "../tools/html.js";
import { canPublishShelf } from "../tools/permissions.js";
import { bookCard } from "./partials/bookCard.js";
import { cardGrid } from "./partials/cardGrid.js";
import { postButton } from "./partials/postButton.js";
import { shelfTitleForm } from "./partials/shelfTitleForm.js";
import { visibilityButton } from "./partials/visibilityButton.js";

export function myShelfView(ctx) {
    const { user, data: { shelf, books, errors, values } } = ctx;

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
            <h2>Visibility</h2>
            <p>${shelf.isPublic ? "Anyone" : "Only you"} can see this shelf</p>
            ${shelf.isPublic || canPublishShelf(user, shelf) ? visibilityButton(shelf) : ""}
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

Sign in as **testUser2** (the password is `testUser2!`).
Your profile is private, so the **make public** buttons on [My shelves][shelves] and on **sci-fi**'s page have gone.
Sign in as **testUser1** (the password is `testUser1!`), whose profile is public, and the buttons are just as before.

## Say why {#publish-your-profile}

**testUser2**'s buttons have gone, but nothing says why.
A private user needs pointing at their profile.

Update **server/views/myShelves.js**.

**server/views/myShelves.js**
```JavaScript{hl_lines="33-35"}
import { html } from "../tools/html.js";
import { canPublishShelf } from "../tools/permissions.js";
import { cardGrid } from "./partials/cardGrid.js";
import { shelfCard } from "./partials/shelfCard.js";
import { shelfTitleForm } from "./partials/shelfTitleForm.js";
import { visibilityButton } from "./partials/visibilityButton.js";

export function myShelvesView(ctx) {
    const { user, data: { shelves, errors, values } } = ctx;
    const publicShelves = shelves.filter((shelf) => shelf.isPublic);
    const privateShelves = shelves.filter((shelf) => !shelf.isPublic);

    return html`
        <section>
            <header>
                <h1>My shelves</h1>
                <p>Make a shelf, then add books to it from any book's page.</p>
            </header>
            <h2>Public shelves</h2>
            <p>Anyone can see these shelves</p>
            ${cardGrid(
                publicShelves.map((shelf) => shelfCard(shelf, visibilityButton(shelf))),
                "No public shelves yet",
            )}
            <h2>Private shelves</h2>
            <p>Only you can see these shelves</p>
            ${cardGrid(
                privateShelves.map((shelf) =>
                    shelfCard(shelf, canPublishShelf(user, shelf) ? visibilityButton(shelf) : "")
                ),
                "No private shelves yet",
            )}
            ${user.isPublic
                ? ""
                : html`<p><a href="/profile">Publish your profile</a> to share shelves</p>`}
        </section>
        <aside>
            <h2>New shelf</h2>
            ${shelfTitleForm("/shelves", values?.title ?? "", errors?.title, "create")}
        </aside>
    `;
}
```

When your profile is private, My shelves ends with a link, **Publish your profile**, then the words **to share shelves**.

The Visibility section on your shelf's page has more to decide now, so it gets a function of its own.
Update **server/views/myShelf.js**.

**server/views/myShelf.js**
```JavaScript{hl_lines="32 40-52"}
import { html } from "../tools/html.js";
import { canPublishShelf } from "../tools/permissions.js";
import { bookCard } from "./partials/bookCard.js";
import { cardGrid } from "./partials/cardGrid.js";
import { postButton } from "./partials/postButton.js";
import { shelfTitleForm } from "./partials/shelfTitleForm.js";
import { visibilityButton } from "./partials/visibilityButton.js";

export function myShelfView(ctx) {
    const { user, data: { shelf, books, errors, values } } = ctx;

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
            <h2>Visibility</h2>
            ${visibilityControls(user, shelf)}
            <h2>Delete shelf</h2>
            <p>The books stay in the catalogue</p>
            ${postButton(`/shelves/${shelf.id}/delete`, "delete shelf")}
        </aside>
    `;
}

// Who can see the shelf, then the button, or a link if the profile is private
function visibilityControls(user, shelf) {
    if (shelf.isPublic) {
        return html`<p>Anyone can see this shelf</p>
        ${visibilityButton(shelf)}`;
    }
    const control = canPublishShelf(user, shelf)
        ? visibilityButton(shelf)
        : html`<p><a href="/profile">Publish your profile</a> to share shelves</p>`;
    return html`<p>Only you can see this shelf</p>
        ${control}`;
}

function removeForm(shelf, book) {
    return postButton(`/shelves/${shelf.id}/books/${book.id}/delete`, "remove");
}
```

`visibilityControls(user, shelf)` says who can see the shelf, then shows the button, or the link to your profile if `canPublishShelf` says no.

As **testUser2**, open [My shelves][shelves] and [**sci-fi**][shelf 4]: both point you at your profile.
**testUser1** sees no link, and the buttons as before.

{{<figure src="images/publish-your-profile.webp" alt="testUser2's sci-fi shelf, with a Visibility section saying Only you can see this shelf, then a Publish your profile link followed by to share shelves" caption="A private user is told how to share a shelf.">}}

## Check the route {#may-publish-shelf}

The buttons have gone, but the route behind them hasn't changed.
We have seen [hiding is not enough][forged form] before.

> Try this on the seed data, then run `deno task db:reset` to put it back.

1. Sign in as **testUser2** and open your own **sci-fi** shelf, [http://localhost:8000/shelves/4][shelf 4].
2. There's no **make public** button to edit, so open the developer tools (**F12**) on the **elements** panel and find the rename form: `<form class="secondary" method="POST" action="/shelves/4/edit">`.
3. Double-click the `action` value, change it to `/shelves/4/publish`, and press **Enter**.
4. Press **rename**.

You land on My shelves with **Shelf 'sci-fi' is now public**, though its owner is still private.

Reset the database.

```bash {linenos=false}
deno task db:reset
```

The route needs the same question as the page.
Update **server/main.js** to make `mayPublishShelf` and put it on the publish route.

**server/main.js**
```JavaScript{hl_lines="45 62 86"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import { addBookController, removeBookController } from "./controllers/shelfBooks.js";
import {
    createShelfController,
    deleteShelfController,
    editShelfController,
    myShelvesController,
    publishShelfController,
    shelfController,
    unpublishShelfController,
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
    canPublishShelf,
    canUseShelf,
    canViewShelf,
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
const mayPublishShelf = requirePermission(canPublishShelf, "shelf");
const mayViewShelf = requirePermission(canViewShelf, "shelf");
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
app.get("/shelves/:shelfId", withShelf, mayViewShelf, shelfController);
app.post("/shelves/:shelfId/edit", mustBeSignedIn, withShelf, mayUseShelf, validateEditShelf, editShelfController);
app.post("/shelves/:shelfId/publish", mustBeSignedIn, withShelf, mayUseShelf, mayPublishShelf, publishShelfController);
app.post("/shelves/:shelfId/unpublish", mustBeSignedIn, withShelf, mayUseShelf, unpublishShelfController);
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

`mayUseShelf` still turns away anyone who doesn't own the shelf, then `mayPublishShelf` checks that the owner's profile is public.

Sign in as **testUser2** and try the devtools trick again.
You get the not found page, **sci-fi** stays private, and the terminal says why.

```plaintext {linenos=false}

============================
POST /shelves/4/publish
[session] {
  id: "c040ce73-c38a-4604-a28f-981fe0b91d9e",
  userId: 2,
  expires: "2026-11-25T16:14:10.981Z"
}
[user] { id: 2, username: "testUser2", avatarId: 2, isPublic: 0 }
[error] POST /shelves/4/publish is not permitted
RESPONSE 404
============================
```

**testUser1** can still share **to read**.

## Forbidden {#forbidden}

The not found page isn't true here.
**testUser2** can see **sci-fi**: it's their own shelf.

We set out a [rule for refusals][not found or forbidden] when shelves got owners, and used it all through [privacy][privacy pattern].

- When the answer depends on **what belongs to whom**, or the thing **can't be seen**, we answer `404 Not Found`. Someone else's shelf to change, a private profile and a private shelf are simply not there for you.
- When the answer depends on **your status**, we answer [`403 Forbidden`][403]. The thing is there, you may know about it, but you aren't allowed to do that.

**sci-fi** is yours, so `mayUseShelf` lets you through.
What stops you is your status: your profile is private.
That's a `403`, and it's the first one in our app.

Add a `ForbiddenError` to **server/tools/errors.js**.

**server/tools/errors.js**
```JavaScript{hl_lines="3-4"}
export class NotFoundError extends Error {}

export class ForbiddenError extends Error {}

export class UnexpectedError extends Error {}

export class ValidationError extends Error {}

export class SchemaValidationError extends Error {
    constructor(errors, values) {
        super("validation failed");
        this.errors = errors;
        this.values = values;
    }
}
```

Add `forbiddenController` to **server/controllers/errors.js**.

**server/controllers/errors.js**
```JavaScript{hl_lines="11-17"}
import { render } from "../tools/render.js";
import { errorView } from "../views/error.js";

export function notFoundController(ctx) {
    ctx.status = 404;
    ctx.data.title = "Not found";
    ctx.data.message = "The requested resource was not found.";
    return render(errorView, ctx);
}

export function forbiddenController(ctx) {
    ctx.status = 403;
    ctx.data.title = "Forbidden";
    ctx.data.message = "You do not have permission to take the requested action.";
    return render(errorView, ctx);
}

export function serverErrorController(ctx) {
    ctx.status = 500;
    ctx.data.title = "Server error";
    ctx.data.message = "Something went wrong on the server.";
    return render(errorView, ctx);
}
```

It renders the error page with a `403` status, the title **Forbidden**, and a message saying you don't have permission.

Update **server/middleware/errors.js** so that `withErrors` answers a `ForbiddenError` with it.

**server/middleware/errors.js**
```JavaScript{hl_lines="1-6 14"}
import { ForbiddenError, NotFoundError, UnexpectedError } from "../tools/errors.js";
import {
    forbiddenController,
    notFoundController,
    serverErrorController,
} from "../controllers/errors.js";

export async function withErrors(ctx, next) {
    try {
        return await next(ctx);
    } catch (error) {
        ctx.error = error;
        if (error instanceof NotFoundError) return notFoundController(ctx);
        if (error instanceof ForbiddenError) return forbiddenController(ctx);
        ctx.error = new UnexpectedError("Unexpected error", { cause: error });
        return serverErrorController(ctx);
    }
}
```

Just as a `NotFoundError` gets the not found page, a `ForbiddenError` now gets the forbidden page.

`requirePermission` always throws a `NotFoundError`.
It needs to throw a `ForbiddenError` for some guards.
Update **server/middleware/access.js**.

**server/middleware/access.js**
```JavaScript{hl_lines="17 21"}
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

export function requirePermission(predicate, key, DeadEnd = NotFoundError) {
    return (ctx, next) => {
        if (predicate(ctx.user, ctx.data[key])) return next(ctx);
        const { url, request } = ctx;
        throw new DeadEnd(`${request.method} ${url.pathname} is not permitted`);
    };
}
```

`requirePermission` takes a third argument, `DeadEnd`, the error to throw when the predicate says no.
Its [default][default parameters] is `NotFoundError`, so every guard we have made so far still gives a `404`.
`DeadEnd` has a capital letter, because it's a class, and we use it with `new`.

Update **server/main.js** to pass `ForbiddenError` when it makes `mayPublishShelf`.

**server/main.js**
```JavaScript{hl_lines="14 63"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import { addBookController, removeBookController } from "./controllers/shelfBooks.js";
import {
    createShelfController,
    deleteShelfController,
    editShelfController,
    myShelvesController,
    publishShelfController,
    shelfController,
    unpublishShelfController,
} from "./controllers/shelves.js";
import { staticController } from "./controllers/static.js";
import { ForbiddenError } from "./tools/errors.js";
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
    canPublishShelf,
    canUseShelf,
    canViewShelf,
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
const mayPublishShelf = requirePermission(canPublishShelf, "shelf", ForbiddenError);
const mayViewShelf = requirePermission(canViewShelf, "shelf");
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
app.get("/shelves/:shelfId", withShelf, mayViewShelf, shelfController);
app.post("/shelves/:shelfId/edit", mustBeSignedIn, withShelf, mayUseShelf, validateEditShelf, editShelfController);
app.post("/shelves/:shelfId/publish", mustBeSignedIn, withShelf, mayUseShelf, mayPublishShelf, publishShelfController);
app.post("/shelves/:shelfId/unpublish", mustBeSignedIn, withShelf, mayUseShelf, unpublishShelfController);
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

As **testUser2**, do the devtools trick again.
This time you get the forbidden page, and the address bar stays at `/shelves/4/publish`.

```plaintext {linenos=false}

============================
POST /shelves/4/publish
[session] {
  id: "b7f9da80-bf71-4d52-9f59-94f17844bea8",
  userId: 2,
  expires: "2026-11-25T16:14:13.727Z"
}
[user] { id: 2, username: "testUser2", avatarId: 2, isPublic: 0 }
[error] POST /shelves/4/publish is not permitted
RESPONSE 403
============================
```

> Point the form at **testUser1**'s shelf instead, `/shelves/1/publish`.
> You still get the not found page, because `mayUseShelf` turns you away first: **classics** isn't yours.

{{<figure src="images/forbidden.webp" alt="The Forbidden page at /shelves/4/publish, saying You do not have permission to take the requested action" caption="Our first 403, still at the URL you posted to.">}}

## Private profile, private shelves {#unpublish-on-private}

There's still one way for a public shelf to end up with a private owner.
Sign in as **testUser1** and make your profile private: **classics** and **fantasy** stay on [the home page][home], and their owner link is the not found page.

Add `unpublishShelvesByOwnerId` to **server/models/shelves.js**.

**server/models/shelves.js**
```JavaScript{hl_lines="55-60"}
import { db } from "./db.js";

export function createShelf({ title, ownerId, isPublic = 0 }) {
    return db.prepare(`
        INSERT INTO shelves(title, ownerId, isPublic)
        VALUES (:title, :ownerId, :isPublic)
        RETURNING id, title, ownerId
    `).get({ title, ownerId, isPublic });
}

export function getShelfById(id) {
    return db.prepare(
        `SELECT id, title, ownerId, isPublic FROM shelves WHERE id=:id`,
    ).get({ id });
}

export function getShelvesByOwnerId(ownerId) {
    return db.prepare(`
        SELECT id, title, ownerId, isPublic,
            (SELECT count(*) FROM shelfBooks WHERE shelfId = shelves.id) AS bookCount
        FROM shelves WHERE ownerId=:ownerId
    `).all({ ownerId });
}

// All public shelves, with the owner's username for display
export function getPublicShelves() {
    return db.prepare(`
        SELECT shelves.id, title, ownerId, username, avatarId,
            (SELECT count(*) FROM shelfBooks WHERE shelfId = shelves.id) AS bookCount
        FROM shelves JOIN users ON users.id = shelves.ownerId
        WHERE shelves.isPublic = 1
    `).all();
}

export function getPublicShelvesByOwnerId(ownerId) {
    return db.prepare(`
        SELECT id, title, ownerId,
            (SELECT count(*) FROM shelfBooks WHERE shelfId = shelves.id) AS bookCount
        FROM shelves WHERE isPublic = 1 AND ownerId=:ownerId
    `).all({ ownerId });
}

export function updateShelfTitleById(id, title) {
    return db.prepare(
        `UPDATE shelves SET title=:title WHERE id=:id RETURNING id, title, ownerId`,
    ).get({ id, title });
}

export function updateShelfIsPublicById(id, isPublic) {
    return db.prepare(
        `UPDATE shelves SET isPublic=:isPublic WHERE id=:id RETURNING id, title, ownerId, isPublic`,
    ).get({ id, isPublic });
}

export function unpublishShelvesByOwnerId(ownerId) {
    return db.prepare(
        `UPDATE shelves SET isPublic=0 WHERE ownerId=:ownerId`,
    ).run({ ownerId });
}

export function deleteShelfById(id) {
    return db.prepare(
        `DELETE FROM shelves WHERE id=:id RETURNING id, title, ownerId`,
    ).get({ id });
}
```

It makes every shelf a user owns private, in one `UPDATE`.

Update `changeProfileVisibility` in **server/controllers/users.js** to use it.

**server/controllers/users.js**
```JavaScript{hl_lines="2 56-57 61-63"}
import { createUser, updateUserAvatarById, updateUserIsPublicById } from "../models/users.js";
import { getPublicShelvesByOwnerId, unpublishShelvesByOwnerId } from "../models/shelves.js";
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
    ctx.data.publicShelves = addThumbnails(getPublicShelvesByOwnerId(targetUser.id));
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

// Unpublishing a profile also unpublishes its shelves,
// so a public shelf always has a public owner
function changeProfileVisibility(ctx, isPublic) {
    const { data: { targetUser } } = ctx;
    updateUserIsPublicById(targetUser.id, isPublic);
    if (!isPublic) {
        unpublishShelvesByOwnerId(targetUser.id);
    }
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

When a profile becomes private, so do all its shelves, and the comment says why.
Making the profile public again doesn't publish anything: you choose which shelves to share.

Run `deno task db:reset`, sign in as **testUser1**, and make your profile private.
Every shelf on My shelves is under **Private shelves**, the home page says **Nobody has shared a shelf yet**, and there's a link to publish your profile.
Publish it, and the **make public** buttons come back.

Now the rule holds wherever a shelf or a profile changes.
Like the [foreign keys][owner id] that keep every shelf's owner real, it keeps our data honest: **a public shelf always has a public owner**.

## An admin user {#admin-user}

Some users are going to be *admins*, who look after the site and the other users.
Before anything can tell an admin apart, there needs to be someone to be one.

Save {{<a href="files/adminUser.webp" download="adminUser.webp" >}}this picture{{< /a >}} as **adminUser.webp** in **db/seed-avatars**.

Update **db/seed.json** to add a third user.

**db/seed.json**
```json{hl_lines="64-81"}
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
                    "isPublic": 1,
                    "books": [
                        "Pride and Prejudice",
                        "Jane Eyre",
                        "The Great Gatsby",
                        "To Kill a Mockingbird"
                    ]
                },
                {
                    "title": "fantasy",
                    "isPublic": 1,
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
        },
        {
            "username": "adminUser",
            "password": "adminUser1!",
            "avatar": "adminUser.webp",
            "isPublic": 1,
            "shelves": [
                {
                    "title": "crime",
                    "isPublic": 1,
                    "books": [
                        "The Girl with the Dragon Tattoo",
                        "Gone Girl",
                        "The Da Vinci Code",
                        "And Then There Were None"
                    ]
                }
            ]
        }
    ]
}
```

**adminUser** has the password `adminUser1!`, a public profile and one public shelf, **crime**.

```bash {linenos=false}
deno task db:reset
```

[The home page][home] now shows **crime** alongside **classics** and **fantasy**.
**adminUser** can sign in, but for now they're an ordinary user like the others.

## An admin flag {#is-admin}

The database needs to say who is an admin.
Update **tasks/db-create.js**.

**tasks/db-create.js**
```JavaScript{hl_lines="24"}
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

db.exec(`
    CREATE TABLE users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT UNIQUE NOT NULL,
        hashedPassword TEXT NOT NULL,
        isAdmin INTEGER DEFAULT 0,
        avatarId INTEGER REFERENCES files(id) ON DELETE SET NULL,
        isPublic INTEGER NOT NULL DEFAULT 0
    )
`);

db.exec(`
    CREATE TABLE shelves (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT NOT NULL,
        ownerId INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
        isPublic INTEGER NOT NULL DEFAULT 0
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

`isAdmin INTEGER DEFAULT 0` means nobody is an admin unless they are made one.

> The users table has every column it needs now, so its placeholder has gone.

Update **server/models/users.js**.

**server/models/users.js**
```JavaScript{hl_lines="12 28 31-32 34"}
import { hashPassword } from "../tools/hash.js";
import { db } from "./db.js";

export function getUserByUsername(username) {
    return db.prepare(
        `SELECT * FROM users WHERE username=:username`,
    ).get({ username });
}

export function getUserById(id) {
    return db.prepare(
        `SELECT id, username, isAdmin, avatarId, isPublic FROM users WHERE id=:id`,
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

export async function createUser({ username, password, isAdmin = 0, isPublic = 0 }) {
    const hashedPassword = await hashPassword(password);
    return db.prepare(`
        INSERT INTO users(username, hashedPassword, isAdmin, isPublic)
        VALUES (:username, :hashedPassword, :isAdmin, :isPublic)
        RETURNING *
    `).get({ username, hashedPassword, isAdmin, isPublic });
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

- `createUser` takes `isAdmin`, with `0` as its default, so signing up never makes an admin.
- `getUserById` returns `isAdmin`, so the signed-in user has it too.

> Both placeholders have gone, because these functions are finished.

Update **db/seed.json** to make **adminUser** an admin.

**db/seed.json**
```json{hl_lines="68"}
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
                    "isPublic": 1,
                    "books": [
                        "Pride and Prejudice",
                        "Jane Eyre",
                        "The Great Gatsby",
                        "To Kill a Mockingbird"
                    ]
                },
                {
                    "title": "fantasy",
                    "isPublic": 1,
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
        },
        {
            "username": "adminUser",
            "password": "adminUser1!",
            "isAdmin": 1,
            "avatar": "adminUser.webp",
            "isPublic": 1,
            "shelves": [
                {
                    "title": "crime",
                    "isPublic": 1,
                    "books": [
                        "The Girl with the Dragon Tattoo",
                        "Gone Girl",
                        "The Da Vinci Code",
                        "And Then There Were None"
                    ]
                }
            ]
        }
    ]
}
```

An admin's profile should say so.
Update **server/views/user.js** to show a badge under the name.

**server/views/user.js**
```JavaScript{hl_lines="18"}
import { html } from "../tools/html.js";
import { canChangeAvatar, canPublishProfile, isSelf } from "../tools/permissions.js";
import { avatarImage } from "./partials/avatarImage.js";
import { cardGrid } from "./partials/cardGrid.js";
import { postButton } from "./partials/postButton.js";
import { shelfCard } from "./partials/shelfCard.js";
import { AVATAR_TYPES } from "../schema/avatar.js";

export function userView(ctx) {
    const { data: { targetUser, publicShelves, errors }, user: currentUser } = ctx;
    // Only you get controls, so only then is there an aside
    const hasControls = isSelf(currentUser, targetUser);

    return html`
        <section>
            <header>
                <h1>${targetUser.username}</h1>
                ${targetUser.isAdmin ? html`<p><span class="badge">admin</span></p>` : ""}
            </header>
            ${avatarImage(targetUser, `Avatar for '${targetUser.username}'`)}
            <h2>Public shelves</h2>
            ${cardGrid(publicShelves.map((shelf) => shelfCard(shelf)), "No public shelves")}
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
        <p>${user.isPublic ? "Anyone" : "Only you"} can see your profile and public shelves</p>
        ${postButton(`/users/${user.id}/${action}`, label)}
    `;
}
```

Badges join the flash message and the current nav link in being filled with the accent colour.
Update the rule near the top of **public/styles.css**.

**public/styles.css**
```css {linenostart="29" hl_lines="1-2"}
/* Badges, the flash message and the current page's nav link are filled with the accent */
.badge,
#flash,
nav a[aria-current="page"] {
    background: var(--highlight);
    color: contrast-color(var(--highlight)); /* black or white, whichever is easier to read */
}
```

Then add a `.badge` rule to the end of the file.

**public/styles.css**
```css {linenostart="301"}
/* A short status label, e.g. admin, public or private */
.badge {
    padding: 0 0.5em;
    border-radius: 1em;
}
```

Run `deno task db:reset` and open [http://localhost:8000/users/3][user 3].
**adminUser** has an **admin** badge, and **testUser1** and **testUser2** don't.

## A users link {#admin-links}

Admins are going to look after the other users from a page of their own, at `/users`.
They need a way to get there.

Update **server/views/layout.js**.

**server/views/layout.js**
```JavaScript{hl_lines="24 51-54"}
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
                        ${adminLinks(user, url.pathname)}
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

function adminLinks(user, path) {
    return user?.isAdmin ? navLink("/users", "users", path) : "";
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

`adminLinks` gives an admin a **users** link, after **books**.
For everyone else, signed in or not, it's nothing.

Sign in as **adminUser** (the password is `adminUser1!`), and **users** is in the nav.
There's no such page yet, so it gets the not found page.
Sign in as **testUser1**, and there's no link at all.

## A users page {#users-page}

Create **server/views/users.js**.

**server/views/users.js**
```JavaScript{hl_lines="1-11"}
import { html } from "../tools/html.js";

export function usersView() {
    return html`
        <section>
            <header>
                <h1>Users</h1>
            </header>
        </section>
    `;
}
```

Add `usersController` to **server/controllers/users.js**.

**server/controllers/users.js**
```JavaScript{hl_lines="10 26-30"}
import { createUser, updateUserAvatarById, updateUserIsPublicById } from "../models/users.js";
import { getPublicShelvesByOwnerId, unpublishShelvesByOwnerId } from "../models/shelves.js";
import { addThumbnails } from "../models/shelfBooks.js";
import { createFile, deleteFileById, getFileById } from "../models/files.js";
import { signin } from "../tools/authentication.js";
import { NotFoundError } from "../tools/errors.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { signupView } from "../views/signup.js";
import { usersView } from "../views/users.js";
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

export function usersController(ctx) {
    ctx.data.title = "Users";
    return render(usersView, ctx);
}

export function userController(ctx) {
    const { data: { targetUser } } = ctx;
    ctx.data.publicShelves = addThumbnails(getPublicShelvesByOwnerId(targetUser.id));
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

// Unpublishing a profile also unpublishes its shelves,
// so a public shelf always has a public owner
function changeProfileVisibility(ctx, isPublic) {
    const { data: { targetUser } } = ctx;
    updateUserIsPublicById(targetUser.id, isPublic);
    if (!isPublic) {
        unpublishShelvesByOwnerId(targetUser.id);
    }
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

Update **server/main.js** to add the route.

**server/main.js**
```JavaScript{hl_lines="29 93"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import { addBookController, removeBookController } from "./controllers/shelfBooks.js";
import {
    createShelfController,
    deleteShelfController,
    editShelfController,
    myShelvesController,
    publishShelfController,
    shelfController,
    unpublishShelfController,
} from "./controllers/shelves.js";
import { staticController } from "./controllers/static.js";
import { ForbiddenError } from "./tools/errors.js";
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
    usersController,
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
    canPublishShelf,
    canUseShelf,
    canViewShelf,
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
const mayPublishShelf = requirePermission(canPublishShelf, "shelf", ForbiddenError);
const mayViewShelf = requirePermission(canViewShelf, "shelf");
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
app.get("/shelves/:shelfId", withShelf, mayViewShelf, shelfController);
app.post("/shelves/:shelfId/edit", mustBeSignedIn, withShelf, mayUseShelf, validateEditShelf, editShelfController);
app.post("/shelves/:shelfId/publish", mustBeSignedIn, withShelf, mayUseShelf, mayPublishShelf, publishShelfController);
app.post("/shelves/:shelfId/unpublish", mustBeSignedIn, withShelf, mayUseShelf, unpublishShelfController);
app.post("/shelves/:shelfId/delete", mustBeSignedIn, withShelf, mayUseShelf, deleteShelfController);
app.post("/shelves/:shelfId/books/:bookId/add", mustBeSignedIn, withShelf, withBook, mayUseShelf, addBookController);
app.post("/shelves/:shelfId/books/:bookId/delete", mustBeSignedIn, withShelf, withBook, mayUseShelf, removeBookController);
app.get("/users", mustBeSignedIn, usersController);
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

`mustBeSignedIn` sends you to sign in if you're signed out, like My shelves.

As **adminUser**, follow the **users** link: the page says **Users**, and the link is marked as the current page.
It isn't only for admins yet.
Sign in as **testUser2** and type [http://localhost:8000/users][users] into the address bar, and it opens for you too.

## Every user {#user-cards}

The users page should show who uses the site.
Add `getUsers` to **server/models/users.js**.

**server/models/users.js**
```JavaScript{hl_lines="4-7"}
import { hashPassword } from "../tools/hash.js";
import { db } from "./db.js";

export function getUsers() {
    return db.prepare(`SELECT id, username, isAdmin, avatarId, isPublic FROM users;`).all();
}

export function getUserByUsername(username) {
    return db.prepare(
        `SELECT * FROM users WHERE username=:username`,
    ).get({ username });
}

export function getUserById(id) {
    return db.prepare(
        `SELECT id, username, isAdmin, avatarId, isPublic FROM users WHERE id=:id`,
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

export async function createUser({ username, password, isAdmin = 0, isPublic = 0 }) {
    const hashedPassword = await hashPassword(password);
    return db.prepare(`
        INSERT INTO users(username, hashedPassword, isAdmin, isPublic)
        VALUES (:username, :hashedPassword, :isAdmin, :isPublic)
        RETURNING *
    `).get({ username, hashedPassword, isAdmin, isPublic });
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

`getUsers` selects the same columns as `getUserById`, for every user.
It never selects the hashed password, which no page should ever need.

Update `usersController` in **server/controllers/users.js**.

**server/controllers/users.js**
```JavaScript{hl_lines="1-6 32"}
import {
    createUser,
    getUsers,
    updateUserAvatarById,
    updateUserIsPublicById,
} from "../models/users.js";
import { getPublicShelvesByOwnerId, unpublishShelvesByOwnerId } from "../models/shelves.js";
import { addThumbnails } from "../models/shelfBooks.js";
import { createFile, deleteFileById, getFileById } from "../models/files.js";
import { signin } from "../tools/authentication.js";
import { NotFoundError } from "../tools/errors.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { signupView } from "../views/signup.js";
import { usersView } from "../views/users.js";
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

export function usersController(ctx) {
    ctx.data.users = getUsers();
    ctx.data.title = "Users";
    return render(usersView, ctx);
}

export function userController(ctx) {
    const { data: { targetUser } } = ctx;
    ctx.data.publicShelves = addThumbnails(getPublicShelvesByOwnerId(targetUser.id));
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

// Unpublishing a profile also unpublishes its shelves,
// so a public shelf always has a public owner
function changeProfileVisibility(ctx, isPublic) {
    const { data: { targetUser } } = ctx;
    updateUserIsPublicById(targetUser.id, isPublic);
    if (!isPublic) {
        unpublishShelvesByOwnerId(targetUser.id);
    }
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

Update **server/views/users.js**.

**server/views/users.js**
```JavaScript{hl_lines="2-6 13 17-28"}
import { html } from "../tools/html.js";
import { avatarImage } from "./partials/avatarImage.js";
import { cardGrid } from "./partials/cardGrid.js";

export function usersView(ctx) {
    const { data: { users } } = ctx;

    return html`
        <section>
            <header>
                <h1>Users</h1>
            </header>
            ${cardGrid(users.map((user) => userCard(user)), "No users")}
        </section>
    `;
}

// A user as a card in a .grid list, with their avatar as the picture.
function userCard(user) {
    return html`
        <li class="card user">
            ${avatarImage(user)}
            <div>
                <h2><a href="/users/${user.id}">${user.username}</a></h2>
            </div>
        </li>
    `;
}
```

`userCard` shows each user's avatar, then their name linked to their profile.
It isn't exported, because only the users page uses it.

Add a `.card.user` rule to **public/styles.css**, before the `.badge` rule.

**public/styles.css**
```css {linenostart="301"}
/* A user card: the avatar on top, everything centred */
.card.user {
    grid-template-rows: auto 1fr;
    justify-items: center;
    text-align: center;
    /* The avatar is smaller on a card, with a gap above it */
    > .avatar {
        inline-size: 6rem;
        margin-block-start: 1rem;
    }
}
```

The avatar goes on top, smaller than on a profile, with everything centred under it.
The `> .avatar` rule is [nested][css nesting] inside `.card.user`, so it only applies to an avatar directly inside a user card.

As **adminUser**, open [users][users].
You see **testUser1**, **testUser2** and **adminUser**.
**testUser2**'s profile and picture are private, so their picture is a plain grey circle, and their link is the not found page, even for an admin.
The circle is grey because the rounded corners clip the browser's broken-image icon.
The page still isn't guarded, so **testUser2** can see everyone too.

## Badges {#user-badges}

An admin looking after users needs to see at a glance who's an admin, and whose profile is private.
Update **server/views/users.js**.

**server/views/users.js**
```JavaScript{hl_lines="25-28"}
import { html } from "../tools/html.js";
import { avatarImage } from "./partials/avatarImage.js";
import { cardGrid } from "./partials/cardGrid.js";

export function usersView(ctx) {
    const { data: { users } } = ctx;

    return html`
        <section>
            <header>
                <h1>Users</h1>
            </header>
            ${cardGrid(users.map((user) => userCard(user)), "No users")}
        </section>
    `;
}

// A user as a card in a .grid list, with their avatar as the picture.
function userCard(user) {
    return html`
        <li class="card user">
            ${avatarImage(user)}
            <div>
                <h2><a href="/users/${user.id}">${user.username}</a></h2>
                <p>
                    ${user.isAdmin ? html`<span class="badge">admin</span>` : ""}
                    <span class="badge">${user.isPublic ? "public" : "private"}</span>
                </p>
            </div>
        </li>
    `;
}
```

Under the name, `userCard` shows an **admin** badge for an admin, and a **public** or **private** badge for everyone.

As **adminUser**, refresh [users][users]: **testUser1** is public, **testUser2** is private, and **adminUser** is an admin and public.

## Admins only {#require-admin}

The users page is only linked from an admin's nav, but anyone signed in can type `/users` and see every user.
We need a guard.

This guard is about your *status*, not about a thing.
Whether you are an admin depends on you alone, so it doesn't need a `with` middleware or a predicate about a thing.
It's a `require` guard, like `requireUser`.

The code already asks whether a user is an admin, in `adminLinks`, and the guard will be the second place.
So the question gets a name.
Add `isAdmin` to **server/tools/permissions.js**.

**server/tools/permissions.js**
```JavaScript{hl_lines="2"}
export const isSelf = (user, targetUser) => user?.id === targetUser.id;
export const isAdmin = (user) => Boolean(user?.isAdmin);
export const canViewUser = (user, targetUser) =>
    isSelf(user, targetUser) || Boolean(targetUser.isPublic);
export const canChangeAvatar = (user, targetUser) => isSelf(user, targetUser);
export const canPublishProfile = (user, targetUser) => isSelf(user, targetUser);
const isOwner = (user, resource) => user?.id === resource.ownerId;
export const canUseShelf = (user, shelf) => isOwner(user, shelf);
export const canPublishShelf = (user, shelf) => isOwner(user, shelf) && Boolean(user.isPublic);
export const canViewShelf = (user, shelf) => isOwner(user, shelf) || Boolean(shelf.isPublic);
```

`isAdmin` only takes the user, because it's only about them.
`user?.isAdmin` means a visitor who isn't signed in is never an admin.

Add `requireAdmin` to **server/middleware/access.js**.

**server/middleware/access.js**
```JavaScript{hl_lines="1-2 18-23"}
import { ForbiddenError, NotFoundError } from "../tools/errors.js";
import { isAdmin } from "../tools/permissions.js";

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

export function requireAdmin(ctx, next) {
    if (isAdmin(ctx.user)) return next(ctx);
    const { url, request } = ctx;
    throw new ForbiddenError(`${request.method} ${url.pathname} is for admins only`);
}

export function requirePermission(predicate, key, DeadEnd = NotFoundError) {
    return (ctx, next) => {
        if (predicate(ctx.user, ctx.data[key])) return next(ctx);
        const { url, request } = ctx;
        throw new DeadEnd(`${request.method} ${url.pathname} is not permitted`);
    };
}
```

`requireAdmin` is a plain middleware, not a factory.
If you are an admin, the rest of the route runs.
If not, it throws a `ForbiddenError`.

Everyone knows there are users, so the page isn't a secret.
You are signed in, and the only thing wrong is your status, so the honest answer is `403 Forbidden`, not `404 Not Found`.

`requireUser` and `requireGuest` send you somewhere sensible next, because a visitor can sign in, and a signed-in user can go home.
There's nowhere sensible to send someone who isn't an admin, so `requireAdmin` refuses.
All three are about [you alone][require user], so they go before anything that loads a thing.

Update `adminLinks` in **server/views/layout.js** to ask `isAdmin` too.

**server/views/layout.js**
```JavaScript{hl_lines="2 53"}
import { html } from "../tools/html.js";
import { isAdmin } from "../tools/permissions.js";
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
                        ${adminLinks(user, url.pathname)}
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

function adminLinks(user, path) {
    return isAdmin(user) ? navLink("/users", "users", path) : "";
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

The nav link and the guard now ask the same question, so they can't disagree.

Update **server/main.js** to put `requireAdmin` on the route, after `mustBeSignedIn`.

**server/main.js**
```JavaScript{hl_lines="42 93"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import { addBookController, removeBookController } from "./controllers/shelfBooks.js";
import {
    createShelfController,
    deleteShelfController,
    editShelfController,
    myShelvesController,
    publishShelfController,
    shelfController,
    unpublishShelfController,
} from "./controllers/shelves.js";
import { staticController } from "./controllers/static.js";
import { ForbiddenError } from "./tools/errors.js";
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
    usersController,
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
import { requireAdmin, requireGuest, requirePermission, requireUser } from "./middleware/access.js";
import { withBook, withShelf, withTargetUser } from "./middleware/resources.js";
import {
    canChangeAvatar,
    canPublishProfile,
    canPublishShelf,
    canUseShelf,
    canViewShelf,
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
const mayPublishShelf = requirePermission(canPublishShelf, "shelf", ForbiddenError);
const mayViewShelf = requirePermission(canViewShelf, "shelf");
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
app.get("/shelves/:shelfId", withShelf, mayViewShelf, shelfController);
app.post("/shelves/:shelfId/edit", mustBeSignedIn, withShelf, mayUseShelf, validateEditShelf, editShelfController);
app.post("/shelves/:shelfId/publish", mustBeSignedIn, withShelf, mayUseShelf, mayPublishShelf, publishShelfController);
app.post("/shelves/:shelfId/unpublish", mustBeSignedIn, withShelf, mayUseShelf, unpublishShelfController);
app.post("/shelves/:shelfId/delete", mustBeSignedIn, withShelf, mayUseShelf, deleteShelfController);
app.post("/shelves/:shelfId/books/:bookId/add", mustBeSignedIn, withShelf, withBook, mayUseShelf, addBookController);
app.post("/shelves/:shelfId/books/:bookId/delete", mustBeSignedIn, withShelf, withBook, mayUseShelf, removeBookController);
app.get("/users", mustBeSignedIn, requireAdmin, usersController);
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

Signed out, you are still sent to sign in.
Sign in as **testUser2** and open [http://localhost:8000/users][users]: you get the forbidden page.

```plaintext {linenos=false}

============================
GET /users
[session] {
  id: "033e85f6-8a2f-4528-b9da-0dc2f2d69d45",
  userId: 2,
  expires: "2026-11-25T16:14:16.561Z"
}
[user] { id: 2, username: "testUser2", isAdmin: 0, avatarId: 2, isPublic: 0 }
[error] GET /users is for admins only
RESPONSE 403
============================
```

As **adminUser**, it opens as before.

{{<figure src="images/users-page.webp" alt="The users page for adminUser, with cards for testUser1 (public), testUser2 (a grey circle and a private badge) and adminUser (admin and public)" caption="Only admins can open the users page.">}}

## Conclusion

A public shelf now always has a public owner: only a public user can share a shelf, and making your profile private makes your shelves private.
We answered `403 Forbidden` for the first time, when the thing is there and the only problem is your status.
**adminUser** is an admin, with a badge, a **users** link and a users page that `requireAdmin` keeps for admins only.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="6 81"}
my-app/
├── .vscode/
│   └── settings.json
├── db/
│   ├── seed-avatars/
│   │   ├── adminUser.webp
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
│   │   │   ├── shelfTitleForm.js
│   │   │   └── visibilityButton.js
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
│   │   ├── user.js
│   │   └── users.js
│   └── main.js
├── tasks/
│   ├── db-create.js
│   ├── db-seed.js
│   └── server.js
├── .gitignore
├── deno.json
└── deno.lock
```

## Challenge: Not found or forbidden?

These are questions to investigate, not changes to keep.

- For each of these, decide whether the answer should be `404` or `403`, then try it: **testUser2** opening **to read**, **testUser2** publishing **sci-fi**, **testUser1** opening `/users`, and **testUser2** publishing **classics**.
- Sign up a new user. Can they share a shelf? What do they need to do first?
- Make **testUser1**'s profile private, then public again. Which shelves are public now, and why doesn't the app republish them?
- Why does `requireAdmin` go after `mustBeSignedIn` on the users route? What would a visitor who isn't signed in get if it went first?

<details>
<summary>Answers</summary>

- **to read** is private, so for **testUser2** it can't be seen: `404`.
- **sci-fi** is **testUser2**'s, and their profile is private, so it's their status: `403`.
- `/users` is about **testUser1**'s status: `403`.
- **classics** isn't **testUser2**'s, so `mayUseShelf` says it's not found, before anyone asks about their status: `404`.

With `requireAdmin` first, a visitor who isn't signed in would get the forbidden page, rather than being asked to sign in.
`mustBeSignedIn` first gives them something sensible to do.

</details>

> If you tried any changes, undo them, and run `deno task db:reset`, so that your code and data match the starting point of the next exercise.

[private shelves]: privacy/private-shelves/
[honest data]: privacy/private-shelves/#honest-data
[privacy pattern]: privacy/private-shelves/#pattern
[not found or forbidden]: owners-and-guards/owners/#not-found-or-forbidden
[forged form]: owners-and-guards/owners/#forged-form
[owner id]: owners-and-guards/owners/#owner-id
[require user]: owners-and-guards/guards/#require-user
[home]: http://localhost:8000/
[shelves]: http://localhost:8000/shelves
[shelf 4]: http://localhost:8000/shelves/4
[user 3]: http://localhost:8000/users/3
[users]: http://localhost:8000/users
[403]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/403
[default parameters]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Default_parameters
[css nesting]: https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_nesting/Using_CSS_nesting

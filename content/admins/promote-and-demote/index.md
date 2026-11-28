---
type: exercise
title: Promote and demote
weight: 2
author: Dr Graeme Stuart
---

In this exercise we will let admins see every profile, make other users admins, and take it away again, without ever removing themselves.
This finishes the bookshelves app.

<!--more-->

> You should have the users page from the [previous exercise][admins].
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

An admin can list every user, but can't do anything about them.
By the end of this exercise you should be able to:

- Give a predicate another way to say yes
- Guard a route with your status first, then with a question about you and a thing
- Share the work of two similar controllers in one function
- Use the whole permissions pattern, from loading a thing to keeping the data honest

## Admins see every profile {#admin-view-user}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Sign in as **adminUser** (the password is `adminUser1!`) and open [users][users].
**testUser2**'s picture is a plain grey circle, and their link is the not found page.
An admin looking after users needs to see them.

Update `canViewUser` in **server/tools/permissions.js**.

**server/tools/permissions.js**
```JavaScript{hl_lines="4"}
export const isSelf = (user, targetUser) => user?.id === targetUser.id;
export const isAdmin = (user) => Boolean(user?.isAdmin);
export const canViewUser = (user, targetUser) =>
    isSelf(user, targetUser) || isAdmin(user) || Boolean(targetUser.isPublic);
export const canChangeAvatar = (user, targetUser) => isSelf(user, targetUser);
export const canPublishProfile = (user, targetUser) => isSelf(user, targetUser);
const isOwner = (user, resource) => user?.id === resource.ownerId;
export const canUseShelf = (user, shelf) => isOwner(user, shelf);
export const canPublishShelf = (user, shelf) => isOwner(user, shelf) && Boolean(user.isPublic);
export const canViewShelf = (user, shelf) => isOwner(user, shelf) || Boolean(shelf.isPublic);
```

You can see a profile if it's yours, if you're an admin, or if it's public.
[`mayViewUser`][may view user] is made from `canViewUser`, so the profile route and the [avatar route][private avatar] both let admins through, with no other changes.

As **adminUser**, refresh [users][users]: **testUser2**'s picture is there, and their profile opens.
Signed out, or as **testUser1**, it's still not found.

## Admin buttons {#admin-buttons}

Each card should have a button to change that user's admin status.
Update **server/views/users.js**.

**server/views/users.js**
```JavaScript{hl_lines="4 14 20-21 31 36-41"}
import { html } from "../tools/html.js";
import { avatarImage } from "./partials/avatarImage.js";
import { cardGrid } from "./partials/cardGrid.js";
import { postButton } from "./partials/postButton.js";

export function usersView(ctx) {
    const { data: { users } } = ctx;

    return html`
        <section>
            <header>
                <h1>Users</h1>
            </header>
            ${cardGrid(users.map((user) => userCard(user, adminButton(user))), "No users")}
        </section>
    `;
}

// A user as a card in a .grid list, with their avatar as the picture.
// Pass extra content (e.g. the admin button) to show it in the card footer
function userCard(user, footer) {
    return html`
        <li class="card user">
            ${avatarImage(user)}
            <div>
                <h2><a href="/users/${user.id}">${user.username}</a></h2>
                <p>
                    ${user.isAdmin ? html`<span class="badge">admin</span>` : ""}
                    <span class="badge">${user.isPublic ? "public" : "private"}</span>
                </p>
                ${footer ? html`<footer>${footer}</footer>` : ""}
            </div>
        </li>
    `;
}

function adminButton(user) {
    const action = user.isAdmin ? "demote" : "promote";
    const label = user.isAdmin ? "remove admin" : "make admin";
    return postButton(`/users/${user.id}/${action}`, label);
}
```

- `userCard` takes a second argument, `footer`, and shows it under the badges if there is one, like `shelfCard`.
- `adminButton` makes the button with `postButton`: **make admin** for an ordinary user, posting to `/users/:userId/promote`, and **remove admin** for an admin, posting to `/users/:userId/demote`.

There are no such routes yet, so the buttons get the not found page.
**adminUser** has a **remove admin** button on their own card too.

## Promote {#promote}

Add `updateUserIsAdminById` to **server/models/users.js**.

**server/models/users.js**
```JavaScript{hl_lines="20-25"}
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

export function updateUserIsAdminById(id, isAdmin) {
    return db.prepare(
        `UPDATE users SET isAdmin=:isAdmin WHERE id=:id`,
    ).run({ id, isAdmin });
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

Add `promoteController` to **server/controllers/users.js**.

**server/controllers/users.js**
```JavaScript{hl_lines="5 69-75"}
import {
    createUser,
    getUsers,
    updateUserAvatarById,
    updateUserIsAdminById,
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

export function promoteController(ctx) {
    const { data: { targetUser } } = ctx;
    updateUserIsAdminById(targetUser.id, 1);
    ctx.flash = `'${targetUser.username}' is now an admin`;
    return redirect("/users", ctx);
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

It sets the target user's `isAdmin` to `1`, flashes a message and redirects back to the users page.

Update **server/main.js** to add the route.

**server/main.js**
```JavaScript{hl_lines="24 100"}
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
    promoteController,
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
app.post("/users/:userId/promote", mustBeSignedIn, requireAdmin, withTargetUser, promoteController);

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

The guards go in the same order as always: first the ones about you alone, then load the thing.

1. `mustBeSignedIn` sends you to sign in if you aren't.
2. `requireAdmin` turns away anyone who isn't an admin, with `403 Forbidden`.
3. `withTargetUser` loads the user, or answers `404 Not Found` if there isn't one.
4. `promoteController` does the work.

Sign in as **adminUser** and make **testUser2** an admin.
The flash says so, and **testUser2**'s card has an **admin** badge and a **remove admin** button.
Sign in as **testUser2**, and **users** is in your nav.

Sign in as **testUser1**, who isn't an admin, and forge a promote with the devtools: on [your profile][profile], change the `action` of the **make profile private** form to `/users/2/promote`, and press it.
You get the forbidden page.

```plaintext {linenos=false}

============================
POST /users/2/promote
[session] {
  id: "e1b592d6-1d9f-4f86-b28f-fc7ccfb8b748",
  userId: 1,
  expires: "2026-11-25T16:14:22.157Z"
}
[user] { id: 1, username: "testUser1", isAdmin: 0, avatarId: 1, isPublic: 1 }
[error] POST /users/2/promote is for admins only
RESPONSE 403
============================
```

`requireAdmin` comes before `withTargetUser`, so a non-admin gets `403` even for a user who doesn't exist.
That's our rule: it's your status that stops you, before anyone looks for the user.

There's no way back yet.
**remove admin** is still the not found page.

## Demote {#demote}

`demoteController` would be `promoteController` with a `0` instead of a `1`, and a different flash.
So the work goes in one function, like [`changeProfileVisibility`][publish profile].
Update **server/controllers/users.js**.

**server/controllers/users.js**
```JavaScript{hl_lines="69 71-74 78-85"}
import {
    createUser,
    getUsers,
    updateUserAvatarById,
    updateUserIsAdminById,
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

function changeAdminStatus(ctx, isAdmin) {
    const { data: { targetUser } } = ctx;
    updateUserIsAdminById(targetUser.id, isAdmin);
    ctx.flash = isAdmin
        ? `'${targetUser.username}' is now an admin`
        : `'${targetUser.username}' is no longer an admin`;
    return redirect("/users", ctx);
}

export function promoteController(ctx) {
    return changeAdminStatus(ctx, 1);
}

export function demoteController(ctx) {
    return changeAdminStatus(ctx, 0);
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

`changeAdminStatus(ctx, isAdmin)` updates the user, flashes **is now an admin** or **is no longer an admin**, and redirects to the users page.
`promoteController` and `demoteController` each call it, with `1` or `0`.

Update **server/main.js** to add the route, guarded just like promote.

**server/main.js**
```JavaScript{hl_lines="23 102"}
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
    demoteController,
    profileController,
    promoteController,
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
app.post("/users/:userId/promote", mustBeSignedIn, requireAdmin, withTargetUser, promoteController);
app.post("/users/:userId/demote", mustBeSignedIn, requireAdmin, withTargetUser, demoteController);

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

Sign in as **adminUser**, make **testUser2** an admin, then remove them again: their badge comes and goes.

You can remove yourself too.
Press **remove admin** on your own card, and you're no longer an admin, so the users page you're sent back to is forbidden.

Reset the database.

```bash {linenos=false}
deno task db:reset
```

## Not your own card {#can-change-admin}

One press of a button and an admin has removed themselves.
If the last admin does it, there's nobody left to put things right.

Add `canChangeAdmin` to **server/tools/permissions.js**.

**server/tools/permissions.js**
```JavaScript{hl_lines="5"}
export const isSelf = (user, targetUser) => user?.id === targetUser.id;
export const isAdmin = (user) => Boolean(user?.isAdmin);
export const canViewUser = (user, targetUser) =>
    isSelf(user, targetUser) || isAdmin(user) || Boolean(targetUser.isPublic);
export const canChangeAdmin = (user, targetUser) => isAdmin(user) && !isSelf(user, targetUser);
export const canChangeAvatar = (user, targetUser) => isSelf(user, targetUser);
export const canPublishProfile = (user, targetUser) => isSelf(user, targetUser);
const isOwner = (user, resource) => user?.id === resource.ownerId;
export const canUseShelf = (user, shelf) => isOwner(user, shelf);
export const canPublishShelf = (user, shelf) => isOwner(user, shelf) && Boolean(user.isPublic);
export const canViewShelf = (user, shelf) => isOwner(user, shelf) || Boolean(shelf.isPublic);
```

You can change a user's admin status if you're an admin and they aren't you.

Update **server/views/users.js** to only give a card its button when `canChangeAdmin` says yes.

**server/views/users.js**
```JavaScript{hl_lines="2 8 15-20"}
import { html } from "../tools/html.js";
import { canChangeAdmin } from "../tools/permissions.js";
import { avatarImage } from "./partials/avatarImage.js";
import { cardGrid } from "./partials/cardGrid.js";
import { postButton } from "./partials/postButton.js";

export function usersView(ctx) {
    const { data: { users }, user: currentUser } = ctx;

    return html`
        <section>
            <header>
                <h1>Users</h1>
            </header>
            ${cardGrid(
                users.map((user) =>
                    userCard(user, canChangeAdmin(currentUser, user) ? adminButton(user) : "")
                ),
                "No users",
            )}
        </section>
    `;
}

// A user as a card in a .grid list, with their avatar as the picture.
// Pass extra content (e.g. the admin button) to show it in the card footer
function userCard(user, footer) {
    return html`
        <li class="card user">
            ${avatarImage(user)}
            <div>
                <h2><a href="/users/${user.id}">${user.username}</a></h2>
                <p>
                    ${user.isAdmin ? html`<span class="badge">admin</span>` : ""}
                    <span class="badge">${user.isPublic ? "public" : "private"}</span>
                </p>
                ${footer ? html`<footer>${footer}</footer>` : ""}
            </div>
        </li>
    `;
}

function adminButton(user) {
    const action = user.isAdmin ? "demote" : "promote";
    const label = user.isAdmin ? "remove admin" : "make admin";
    return postButton(`/users/${user.id}/${action}`, label);
}
```

`usersView` takes the signed-in user from `ctx` as `currentUser`, because `user` already means the user on each card.

As **adminUser**, open [users][users]: **testUser1** and **testUser2** have their buttons, and your own card has none.

## Check the routes {#may-change-admin}

Your own card has no button, but the route behind it still takes a demote from you.

> Try this on the seed data, then run `deno task db:reset` to put it back.

1. Sign in as **adminUser** and open [http://localhost:8000/users][users].
2. Open the developer tools (**F12**) on the **elements** panel and find **testUser1**'s **make admin** form: `<form method="POST" action="/users/1/promote">`.
3. Double-click the `action` value, change it to `/users/3/demote`, and press **Enter**.
4. Press **make admin**.

The flash says **'adminUser' is no longer an admin**, and the users page is forbidden.

Reset the database.

```bash {linenos=false}
deno task db:reset
```

The route needs the same question as the page.
Update **server/main.js**.

**server/main.js**
```JavaScript{hl_lines="47 65 103-104"}
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
    demoteController,
    profileController,
    promoteController,
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
    canChangeAdmin,
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
const mayChangeAdmin = requirePermission(canChangeAdmin, "targetUser", ForbiddenError);
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
app.post("/users/:userId/promote", mustBeSignedIn, requireAdmin, withTargetUser, mayChangeAdmin, promoteController);
app.post("/users/:userId/demote", mustBeSignedIn, requireAdmin, withTargetUser, mayChangeAdmin, demoteController);

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

`mayChangeAdmin` is made from `canChangeAdmin`, like the [other `may` guards][require permission].
It goes after `withTargetUser`, because it needs the user that loads.

It passes `ForbiddenError`, so the answer is `403`.
An admin can see every user, so **adminUser** isn't being kept from something they can't see, and a `404` wouldn't be true.
`canChangeAdmin` starts from your status, `isAdmin`, and here the refusal is about the action, not the user.

Sign in as **adminUser** and try the devtools trick again.
You get the forbidden page, and you stay an admin.

```plaintext {linenos=false}

============================
POST /users/3/demote
[session] {
  id: "319dc745-0ed5-4d8c-a5bb-0916bae7502b",
  userId: 3,
  expires: "2026-11-25T16:14:19.382Z"
}
[user] { id: 3, username: "adminUser", isAdmin: 1, avatarId: 3, isPublic: 1 }
[error] POST /users/3/demote is not permitted
RESPONSE 403
============================
```

Promoting and demoting other users works as before.

## An admin button partial {#admin-button}

The profile page is going to show the same button as the users page.
Create **server/views/partials/adminButton.js**, moving `adminButton` out of **server/views/users.js** unchanged.

**server/views/partials/adminButton.js**
```JavaScript{hl_lines="1-7"}
import { postButton } from "./postButton.js";

export function adminButton(user) {
    const action = user.isAdmin ? "demote" : "promote";
    const label = user.isAdmin ? "remove admin" : "make admin";
    return postButton(`/users/${user.id}/${action}`, label);
}
```

Update **server/views/users.js** to import it, and remove the `postButton` import it no longer needs.

**server/views/users.js**
```JavaScript{hl_lines="3"}
import { html } from "../tools/html.js";
import { canChangeAdmin } from "../tools/permissions.js";
import { adminButton } from "./partials/adminButton.js";
import { avatarImage } from "./partials/avatarImage.js";
import { cardGrid } from "./partials/cardGrid.js";

export function usersView(ctx) {
    const { data: { users }, user: currentUser } = ctx;

    return html`
        <section>
            <header>
                <h1>Users</h1>
            </header>
            ${cardGrid(
                users.map((user) =>
                    userCard(user, canChangeAdmin(currentUser, user) ? adminButton(user) : "")
                ),
                "No users",
            )}
        </section>
    `;
}

// A user as a card in a .grid list, with their avatar as the picture.
// Pass extra content (e.g. the admin button) to show it in the card footer
function userCard(user, footer) {
    return html`
        <li class="card user">
            ${avatarImage(user)}
            <div>
                <h2><a href="/users/${user.id}">${user.username}</a></h2>
                <p>
                    ${user.isAdmin ? html`<span class="badge">admin</span>` : ""}
                    <span class="badge">${user.isPublic ? "public" : "private"}</span>
                </p>
                ${footer ? html`<footer>${footer}</footer>` : ""}
            </div>
        </li>
    `;
}
```

Nothing changes in the browser.

## Admin status on profiles {#profile-admin}

An admin can promote and demote from the users page, but not from the profile they're looking at.
Update **server/views/user.js**.

**server/views/user.js**
```JavaScript{hl_lines="2-8 17-18 36-38"}
import { html } from "../tools/html.js";
import {
    canChangeAdmin,
    canChangeAvatar,
    canPublishProfile,
    isSelf,
} from "../tools/permissions.js";
import { adminButton } from "./partials/adminButton.js";
import { avatarImage } from "./partials/avatarImage.js";
import { cardGrid } from "./partials/cardGrid.js";
import { postButton } from "./partials/postButton.js";
import { shelfCard } from "./partials/shelfCard.js";
import { AVATAR_TYPES } from "../schema/avatar.js";

export function userView(ctx) {
    const { data: { targetUser, publicShelves, errors }, user: currentUser } = ctx;
    // Only you (or an admin) get controls, so only then is there an aside
    const hasControls = isSelf(currentUser, targetUser) || canChangeAdmin(currentUser, targetUser);

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
                    ${canChangeAdmin(currentUser, targetUser)
                        ? html`<h2>Admin status</h2>${adminButton(targetUser)}`
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

Until now, only you got an aside, on your own profile.
Now an admin gets one on other people's profiles too, so `hasControls` is `isSelf` *or* `canChangeAdmin`.
On someone else's profile, an admin's aside has just the **Admin status** section.
The avatar form and the visibility button are still yours alone, because `canChangeAvatar` and `canPublishProfile` haven't changed.

Sign in as **adminUser** and open [**testUser1**'s profile][user 1]: **Admin status**, with **make admin**.
**Public shelves** lists **classics** and **fantasy**; **to read** is private, so it isn't there.
Press it, and you're back on the users page with **testUser1** an admin.
Your own profile has no admin section, because you can't change your own status.

{{<figure src="images/admin-status.webp" alt="testUser1's profile viewed by adminUser, with Public shelves listing classics and fantasy, and an aside that has only Admin status and a make admin button" caption="An admin can change another user's status from their profile.">}}

## The pattern, complete {#pattern}

Every rule about who may do what in our app now follows the same pattern, which we started with [owners][owners pattern] and named in [privacy][named pattern].

1. **Load** the thing with a `with` middleware, such as `withShelf` or `withTargetUser`, which answers `404` if there isn't one.
2. **Ask** a `can` question in **server/tools/permissions.js**, such as `canPublishShelf` or `canChangeAdmin`, built from small questions like `isOwner`, `isSelf` and `isAdmin`.
3. **Use it in the route**, as a `may` guard made with [`requirePermission`][require permission], and **in the view**, to only show the button when the answer is yes.
4. **Filter lists in SQL**, so that a list like [public shelves][public shelves] never contains something you can't see.
5. **Keep the data honest**, so that a rule like [a public shelf has a public owner][honest data] holds whatever order things happen in.

Guards about you alone, `mustBeSignedIn`, `mustBeSignedOut` and `requireAdmin`, come before anything is loaded.
When the answer is no, it's [`404`][not found or forbidden] when it's about what belongs to whom or what you can see, and [`403`][forbidden] when it's about your status.

## The end of bookshelves {#the-end}

This is the last exercise in the bookshelves app.
Over these units you have built an app with a database, validated forms, sessions, users who own their shelves, avatars, privacy, and admins.

Your code should now match the [finished app][final code].
Compare your work against it, and look for any differences.

## Conclusion

Admins can see every profile, and make other users admins or take it away, from the users page or from a profile.
An admin can't remove themselves, either with the button or with a forged form.
We have completed the permissions pattern: load, ask, use the answer in the route and the view, filter lists in SQL, and keep the data honest.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="62"}
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
│   │   │   ├── adminButton.js
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

## Challenge: The last admin

These are questions to investigate, not changes to keep.

- Make **testUser1** an admin, then sign in as **testUser1** and remove **adminUser**. Sign back in as **adminUser**: what can you see now?
- Can the app ever be left with no admins at all? Try it, then explain why.
- Sign up a new user. Can they ever become an admin without an admin's help?

<details>
<summary>Answers</summary>

**adminUser** has no badge, no **users** link and no buttons, and `/users` is forbidden.

No.
Only an admin can remove an admin, and `canChangeAdmin` stops you removing yourself, on the page and on the route.
So whoever removes the second-to-last admin is always left as an admin.

No.
`createUser` gives every new user `isAdmin` `0`, and only `promoteController` sets it to `1`, behind `requireAdmin`.
The only other way is in the database itself, such as **adminUser** in **db/seed.json**.

</details>

> If you tried any changes, undo them, and run `deno task db:reset`.

[admins]: admins/admins/
[forbidden]: admins/admins/#forbidden
[honest data]: admins/admins/#unpublish-on-private
[may view user]: privacy/private-profiles/#may-view-user
[private avatar]: privacy/private-profiles/#private-avatar
[publish profile]: privacy/private-profiles/#publish-profile
[require permission]: privacy/private-profiles/#require-permission
[named pattern]: privacy/private-profiles/#named-pattern
[public shelves]: privacy/private-shelves/#public-shelves
[owners pattern]: owners-and-guards/owners/#pattern
[not found or forbidden]: owners-and-guards/owners/#not-found-or-forbidden
[final code]: https://github.com/ggstuart/deno-exercises/tree/bookshelves
[users]: http://localhost:8000/users
[profile]: http://localhost:8000/profile
[user 1]: http://localhost:8000/users/1

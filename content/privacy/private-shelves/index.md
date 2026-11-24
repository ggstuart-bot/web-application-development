---
type: exercise
title: Private shelves
weight: 2
author: Dr Graeme Stuart
---

In this exercise we will give every shelf its own public or private setting.
Only public shelves will be listed, a private shelf will be not found for everyone but its owner, and you will be able to share a shelf, and take it back.

<!--more-->

> You should have private profiles, **tools/permissions.js** and `requirePermission` from the [previous exercise][previous exercise].
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

Profiles can be private now, but every shelf is still on show to everyone, on the home page and on its owner's profile.
In this exercise, shelves get the same treatment as profiles, and our permissions pattern is complete.
By the end of this exercise you should be able to:

- Filter a list in SQL so that it only ever fetches the rows anyone may see
- Combine two predicates into a third, and keep a helper private to its module
- Reuse an existing `may` guard on a new route that changes the same thing
- Move a button that is used twice into a partial
- Explain the whole permissions pattern, and why a public shelf should always have a public owner

## Public or private shelves {#shelf-is-public}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Each shelf is going to be public or private, on its own.
Update **tasks/db-create.js** to give the shelves table an `isPublic` column.

**tasks/db-create.js**
```JavaScript{hl_lines="34-35"}
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

Just like a profile, a shelf is private until its owner shares it.

> The shelves table has lost its placeholder, because it is finished.
> The users table keeps its placeholder.

Update **server/models/shelves.js**.

**server/models/shelves.js**
```JavaScript{hl_lines="3 5-6 8 12-14 19"}
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

- `createShelf` takes `isPublic`, with `0` as its default, so making a shelf on My shelves still makes a private one.
- `getShelfById` and `getShelvesByOwnerId` return `isPublic`.

> `createShelf`, `getShelfById` and `getShelvesByOwnerId` have lost their placeholders, because they are finished.
> `getShelves` still has its placeholder.

The seed data need some public shelves.
Update **db/seed.json** so that **testUser1**'s **classics** and **fantasy** are public.

**db/seed.json**
```json{hl_lines="11 21"}
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
        }
    ]
}
```

Update **tasks/db-seed.js** to pass each shelf's `isPublic` to `createShelf`.

**tasks/db-seed.js**
```JavaScript{hl_lines="22-26"}
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
        const { id: shelfId } = createShelf({
            title: shelf.title,
            ownerId: userId,
            isPublic: shelf.isPublic,
        });
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

Every other shelf has no `isPublic`, so it's private.

My shelves should say which of your shelves are which.
Update **server/views/myShelves.js**.

**server/views/myShelves.js**
```JavaScript{hl_lines="8-9 17-22"}
import { html } from "../tools/html.js";
import { cardGrid } from "./partials/cardGrid.js";
import { shelfCard } from "./partials/shelfCard.js";
import { shelfTitleForm } from "./partials/shelfTitleForm.js";

export function myShelvesView(ctx) {
    const { data: { shelves, errors, values } } = ctx;
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
            ${cardGrid(publicShelves.map((shelf) => shelfCard(shelf)), "No public shelves yet")}
            <h2>Private shelves</h2>
            <p>Only you can see these shelves</p>
            ${cardGrid(privateShelves.map((shelf) => shelfCard(shelf)), "No private shelves yet")}
        </section>
        <aside>
            <h2>New shelf</h2>
            ${shelfTitleForm("/shelves", values?.title ?? "", errors?.title, "create")}
        </aside>
    `;
}
```

My shelves now has two sections, **Public shelves** and **Private shelves**, each saying who can see them.

We said that [a list is filtered in SQL][my shelves], so why use [`filter`][filter] here?
The SQL has already decided which rows you may see: `getShelvesByOwnerId` only fetches your own shelves.
Every one of them is yours to see, so splitting them into two groups for the page is just presentation, not permission.

Rebuild the database.

```bash {linenos=false}
deno task db:reset
```

Sign in as **testUser1** (the password is `testUser1!`) and open [My shelves][shelves].
**classics** and **fantasy** are under **Public shelves**, and **to read** is under **Private shelves**.

That isn't true anywhere else yet: the home page and the profiles still show every shelf.

## Public shelves on the home page {#public-shelves}

The home page lists every shelf, so a private shelf is there for anyone to see, and **testUser2**'s private profile is linked from it.
In the [profile pages exercise][everyones shelves], `getShelves` had a placeholder saying it would change when shelves can be public.
Now it does.

Rename `getShelves` to `getPublicShelves` in **server/models/shelves.js**, and add a `WHERE` clause.

**server/models/shelves.js**
```JavaScript{hl_lines="25-26 31"}
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

`WHERE shelves.isPublic = 1` means the database only ever gives us public shelves.
Nothing private reaches our JavaScript, so nothing needs to be hidden.
The new name says which shelves it gets.

> The placeholder has gone, because `getPublicShelves` is finished.

Update **server/controllers/home.js**.

**server/controllers/home.js**
```JavaScript{hl_lines="1 7"}
import { getPublicShelves } from "../models/shelves.js";
import { addThumbnails } from "../models/shelfBooks.js";
import { render } from "../tools/render.js";
import { homeView } from "../views/home.js";

export function homeController(ctx) {
    ctx.data.publicShelves = addThumbnails(getPublicShelves());
    return render(homeView, ctx);
}
```

The shelves go on `ctx.data` as `publicShelves`, so the name says what they are wherever they are used.

Update **server/views/home.js**.

**server/views/home.js**
```JavaScript{hl_lines="8 18 20-21"}
import { html } from "../tools/html.js";
import { cardGrid } from "./partials/cardGrid.js";
import { avatarImage } from "./partials/avatarImage.js";
import { shelfCard } from "./partials/shelfCard.js";
import { searchForm } from "./partials/searchForm.js";

export function homeView(ctx) {
    const { data: { publicShelves } } = ctx;

    return html`
        <section>
            <header>
                <h1>Welcome to Bookshelves</h1>
                <p>Organise your books with bookshelves.</p>
            </header>
            <h2>Find a book</h2>
            ${searchForm()}
            <h2>Public shelves</h2>
            ${cardGrid(
                publicShelves.map((shelf) => shelfCard(shelf, ownerLink(shelf))),
                "Nobody has shared a shelf yet",
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

The heading says **Public shelves**, and if there are none, the page says **Nobody has shared a shelf yet**.

Open [the home page][home].
You see **testUser1**'s **classics** and **fantasy**, and nothing else.
Every owner link goes to a public profile, so the broken links and pictures have gone.

## Public shelves on profiles {#profile-public-shelves}

A profile still lists all of its owner's shelves.
Sign out and open [http://localhost:8000/users/1][user 1]: **testUser1**'s private **to read** shelf is there, though My shelves says only they can see it.

Add `getPublicShelvesByOwnerId` to **server/models/shelves.js**.

**server/models/shelves.js**
```JavaScript{hl_lines="35-42"}
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

export function deleteShelfById(id) {
    return db.prepare(
        `DELETE FROM shelves WHERE id=:id RETURNING id, title, ownerId`,
    ).get({ id });
}
```

It's `getShelvesByOwnerId` with `isPublic = 1` in its `WHERE` clause.
`getShelvesByOwnerId` stays as it is, for My shelves and a book's page, which are only ever about your own shelves.

Update `userController` in **server/controllers/users.js** to use it.

**server/controllers/users.js**
```JavaScript{hl_lines="2 27"}
import { createUser, updateUserAvatarById, updateUserIsPublicById } from "../models/users.js";
import { getPublicShelvesByOwnerId } from "../models/shelves.js";
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

Update **server/views/user.js**.

**server/views/user.js**
```JavaScript{hl_lines="10 20-21 55"}
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

The heading says **Public shelves**.
The aside on your own profile now says who can see your profile *and public shelves*, because a private profile hides its public shelves too.

Refresh [http://localhost:8000/users/1][user 1].
It shows **classics** and **fantasy**, whoever is looking, even **testUser1**.
My shelves is where you see all of your own.

## A private shelf {#may-view-shelf}

A private shelf is no longer listed anywhere but My shelves, but it still opens for anyone who has its address.
Sign out and open [http://localhost:8000/shelves/3][shelf 3]: **testUser1**'s **to read** shelf is all there.

We need a new question: *can this user see this shelf?*
You can if it's yours, or if it's public.
Update **server/tools/permissions.js**.

**server/tools/permissions.js**
```JavaScript{hl_lines="6-8"}
export const isSelf = (user, targetUser) => user?.id === targetUser.id;
export const canViewUser = (user, targetUser) =>
    isSelf(user, targetUser) || Boolean(targetUser.isPublic);
export const canChangeAvatar = (user, targetUser) => isSelf(user, targetUser);
export const canPublishProfile = (user, targetUser) => isSelf(user, targetUser);
const isOwner = (user, resource) => user?.id === resource.ownerId;
export const canUseShelf = (user, shelf) => isOwner(user, shelf);
export const canViewShelf = (user, shelf) => isOwner(user, shelf) || Boolean(shelf.isPublic);
```

- This is the second time we need the owner check, so it gets a name of its own, `isOwner`. `canUseShelf` and `canViewShelf` both use it.
- `isOwner` isn't exported. It's only a piece of the rules, not a question the rest of the app asks. Routes and views ask `can` questions.
- `isOwner` takes any `resource` with an `ownerId`, not only a shelf.

Update **server/main.js** to make `mayViewShelf`, and put it on the shelf's page.

**server/main.js**
```JavaScript{hl_lines="44 59 80"}
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

`mayViewShelf` goes straight after `withShelf`, which loads the shelf it checks.

Now that a public shelf is the only kind other people can open, the shelf's page can say so.
Update **server/views/publicShelf.js**.

**server/views/publicShelf.js**
```JavaScript{hl_lines="13"}
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
                <p>A public shelf by <a href="/users/${owner.id}">${avatarImage(owner)}${owner
                    .username}</a></p>
            </header>
            ${cardGrid(books.map((book) => bookCard(book)), "No books on this shelf yet")}
            <p><a href="/">Back to home</a></p>
        </section>
    `;
}
```

Sign out, or sign in as **testUser2**, and open [http://localhost:8000/shelves/3][shelf 3].
You get the not found page, and the terminal says why.

```plaintext {linenos=false}

============================
GET /shelves/3
[session] {
  id: "c03268c7-2bf4-4206-9b56-688582121e8c",
  userId: 2,
  expires: "2026-11-25T16:08:20.291Z"
}
[user] { id: 2, username: "testUser2", avatarId: 2, isPublic: 0 }
[error] GET /shelves/3 is not permitted
RESPONSE 404
============================
```

A private shelf, like a private profile, is something you can't see, so it's not found.
**testUser1** can still open it, and [http://localhost:8000/shelves/1][shelf 1] opens for everyone, with **A public shelf by testUser1**.

## A make public button {#shelf-publish-button}

There's no way to share a shelf yet.
Update **server/views/myShelves.js**.

**server/views/myShelves.js**
```JavaScript{hl_lines="3 23-28"}
import { html } from "../tools/html.js";
import { cardGrid } from "./partials/cardGrid.js";
import { postButton } from "./partials/postButton.js";
import { shelfCard } from "./partials/shelfCard.js";
import { shelfTitleForm } from "./partials/shelfTitleForm.js";

export function myShelvesView(ctx) {
    const { data: { shelves, errors, values } } = ctx;
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
            ${cardGrid(publicShelves.map((shelf) => shelfCard(shelf)), "No public shelves yet")}
            <h2>Private shelves</h2>
            <p>Only you can see these shelves</p>
            ${cardGrid(
                privateShelves.map((shelf) =>
                    shelfCard(shelf, postButton(`/shelves/${shelf.id}/publish`, "make public"))
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

Each card under **Private shelves** now has a **make public** button in its footer, made with `postButton`, posting to `/shelves/:shelfId/publish`.
We pass the button as the card's footer, just as the home page passes the owner.

My shelves doesn't ask `canUseShelf` before showing the button.
It doesn't need to: the list comes from `getShelvesByOwnerId`, so every shelf on it is yours.

As **testUser1**, press **make public** on **to read**.
There's no such route yet, so you get the not found page.

## Share a shelf {#publish-shelf}

Sharing a shelf is a change to it, so the route is guarded just like renaming or deleting it, with [`mayUseShelf`][may use shelf].
We don't need a new predicate, because the same people can do it.

Add `updateShelfIsPublicById` to **server/models/shelves.js**.

**server/models/shelves.js**
```JavaScript{hl_lines="49-54"}
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

export function deleteShelfById(id) {
    return db.prepare(
        `DELETE FROM shelves WHERE id=:id RETURNING id, title, ownerId`,
    ).get({ id });
}
```

It sets a shelf's `isPublic`, and returns the changed shelf.

Add `publishShelfController` to **server/controllers/shelves.js**.

**server/controllers/shelves.js**
```JavaScript{hl_lines="5 49-56"}
import {
    createShelf,
    deleteShelfById,
    getShelvesByOwnerId,
    updateShelfIsPublicById,
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

export function publishShelfController(ctx) {
    const { data: { shelf: { id } } } = ctx;
    const shelf = updateShelfIsPublicById(id, 1);
    ctx.flash = `Shelf '${shelf.title}' is now public`;
    // Back to My shelves, where the shelf has moved to its new section
    return redirect("/shelves", ctx);
}

export function deleteShelfController(ctx) {
    const { data: { shelf: { id } } } = ctx;
    const shelf = deleteShelfById(id);
    ctx.flash = `Shelf '${shelf.title}' deleted`;
    return redirect("/shelves", ctx);
}
```

It makes the shelf public, flashes a message with its title, and redirects to My shelves, where the shelf has moved to the public section.

Update **server/main.js** to add the route.

**server/main.js**
```JavaScript{hl_lines="9 83"}
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
app.post("/shelves/:shelfId/publish", mustBeSignedIn, withShelf, mayUseShelf, publishShelfController);
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

As **testUser1**, press **make public** on **to read**.
You get **Shelf 'to read' is now public**, and it's on [the home page][home] and on your profile.
Sign out, and [http://localhost:8000/shelves/3][shelf 3] opens.

> Try the [devtools trick][forged form] here too.
> As **testUser1**, change a **make public** button's form `action` to `/shelves/4/publish`, to share **testUser2**'s **sci-fi**.
> You get the not found page, because `mayUseShelf` says it isn't yours.

There's no way back to private yet.

## And private again {#unpublish-shelf}

Once a shelf is public, there's no way to take it back.
Update **server/views/myShelves.js** to give each public shelf a **make private** button.

**server/views/myShelves.js**
```JavaScript{hl_lines="20-25"}
import { html } from "../tools/html.js";
import { cardGrid } from "./partials/cardGrid.js";
import { postButton } from "./partials/postButton.js";
import { shelfCard } from "./partials/shelfCard.js";
import { shelfTitleForm } from "./partials/shelfTitleForm.js";

export function myShelvesView(ctx) {
    const { data: { shelves, errors, values } } = ctx;
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
                publicShelves.map((shelf) =>
                    shelfCard(shelf, postButton(`/shelves/${shelf.id}/unpublish`, "make private"))
                ),
                "No public shelves yet",
            )}
            <h2>Private shelves</h2>
            <p>Only you can see these shelves</p>
            ${cardGrid(
                privateShelves.map((shelf) =>
                    shelfCard(shelf, postButton(`/shelves/${shelf.id}/publish`, "make public"))
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

The two controllers would be the same apart from one number and the flash message.
Update **server/controllers/shelves.js**.

**server/controllers/shelves.js**
```JavaScript{hl_lines="49 51-54 59-66"}
import {
    createShelf,
    deleteShelfById,
    getShelvesByOwnerId,
    updateShelfIsPublicById,
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

function changeShelfVisibility(ctx, isPublic) {
    const { data: { shelf: { id } } } = ctx;
    const shelf = updateShelfIsPublicById(id, isPublic);
    ctx.flash = isPublic
        ? `Shelf '${shelf.title}' is now public`
        : `Shelf '${shelf.title}' is now private`;
    // Back to My shelves, where the shelf has moved to its new section
    return redirect("/shelves", ctx);
}

export function publishShelfController(ctx) {
    return changeShelfVisibility(ctx, 1);
}

export function unpublishShelfController(ctx) {
    return changeShelfVisibility(ctx, 0);
}

export function deleteShelfController(ctx) {
    const { data: { shelf: { id } } } = ctx;
    const shelf = deleteShelfById(id);
    ctx.flash = `Shelf '${shelf.title}' deleted`;
    return redirect("/shelves", ctx);
}
```

`changeShelfVisibility(ctx, isPublic)` does the work for both, like `changeProfileVisibility` does for profiles.

Update **server/main.js** to add the route, guarded just like the publish route.

**server/main.js**
```JavaScript{hl_lines="11 85"}
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
app.post("/shelves/:shelfId/publish", mustBeSignedIn, withShelf, mayUseShelf, publishShelfController);
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

As **testUser1**, press **make private** on **classics**.
It moves to the private section, and leaves the home page and your profile.
Sign out, and [http://localhost:8000/shelves/1][shelf 1] is not found.
Make it public again, and it's back.

## A visibility button {#visibility-button}

The two sections of My shelves each build a button for their cards.
They are the same button, pointing the other way, and a shelf's own page is going to need it too.

Create **server/views/partials/visibilityButton.js**.

**server/views/partials/visibilityButton.js**
```JavaScript{hl_lines="1-8"}
import { postButton } from "./postButton.js";

// The publish or unpublish button for a shelf
export function visibilityButton(shelf) {
    const action = shelf.isPublic ? "unpublish" : "publish";
    const label = shelf.isPublic ? "make private" : "make public";
    return postButton(`/shelves/${shelf.id}/${action}`, label);
}
```

Like the button on your profile, `visibilityButton` picks the action and the label from the shelf's `isPublic`.

Update **server/views/myShelves.js** to use it.

**server/views/myShelves.js**
```JavaScript{hl_lines="5 21 27"}
import { html } from "../tools/html.js";
import { cardGrid } from "./partials/cardGrid.js";
import { shelfCard } from "./partials/shelfCard.js";
import { shelfTitleForm } from "./partials/shelfTitleForm.js";
import { visibilityButton } from "./partials/visibilityButton.js";

export function myShelvesView(ctx) {
    const { data: { shelves, errors, values } } = ctx;
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
                privateShelves.map((shelf) => shelfCard(shelf, visibilityButton(shelf))),
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

Both sections use `visibilityButton`, and My shelves no longer needs `postButton` itself.
Nothing changes in the browser.

## Visibility on the shelf's page {#shelf-visibility}

To share a shelf you are looking at, you have to go back to My shelves to find its button.
Update **server/views/myShelf.js**.

**server/views/myShelf.js**
```JavaScript{hl_lines="6 30-32"}
import { html } from "../tools/html.js";
import { bookCard } from "./partials/bookCard.js";
import { cardGrid } from "./partials/cardGrid.js";
import { postButton } from "./partials/postButton.js";
import { shelfTitleForm } from "./partials/shelfTitleForm.js";
import { visibilityButton } from "./partials/visibilityButton.js";

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
            <h2>Visibility</h2>
            <p>${shelf.isPublic ? "Anyone" : "Only you"} can see this shelf</p>
            ${visibilityButton(shelf)}
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

Your own shelf's page has a **Visibility** section in the aside, between renaming and deleting it.
It says who can see the shelf, and has `visibilityButton`, its second use.

`myShelfView` doesn't ask a question before showing it, because `shelfController` has already asked `canUseShelf` to choose this view.

As **testUser1**, open [**to read**][shelf 3].
It says **Only you can see this shelf**, with a **make public** button.

{{<figure src="images/shelf-visibility.webp" alt="testUser1's to read shelf, with an aside showing the rename form, a Visibility section saying Only you can see this shelf with a make public button, and the delete shelf button" caption="Your own shelf's page says who can see it.">}}

Press it, and you land on My shelves, with **to read** among your public shelves.

## The permissions pattern {#pattern}

Our [permissions pattern][owners pattern] is complete.
Every route that touches something follows it.

1. **Load the thing.** A `with` middleware loads what the route is about: `withShelf`, `withBook`, `withTargetUser`. If it doesn't exist, the route is not found.
2. **Ask a `can` predicate.** Every question about permissions lives in **tools/permissions.js**, as a small function that takes the signed-in user and the thing: `canViewUser`, `canChangeAvatar`, `canPublishProfile`, `canUseShelf`, `canViewShelf`. Smaller pieces, such as `isOwner`, stay private to the module.
3. **Use it twice.** In the route, a `may` guard made by `requirePermission` turns away anyone the answer is no for. In the view, or the controller that picks the view, the same predicate decides whether to show a button. The route and the page can't disagree, because they ask the same function.
4. **Filter lists in SQL.** The home page asks `getPublicShelves`, and a profile asks `getPublicShelvesByOwnerId`, so a list never fetches a row that its reader can't see. Splitting rows you may already see, as My shelves does, is presentation, not permission.

Two rules go with it.

- **A `mustBe` guard is about you alone.** `mustBeSignedIn` and `mustBeSignedOut` only look at `ctx.user`, so they go first, and they send you somewhere sensible next, rather than answering with an error.
- **What belongs to whom, or can't be seen, is not found.** A private profile, a private shelf, or someone else's shelf to change all give `404 Not Found`, exactly as if they weren't there. When the answer depends only on your *status*, we will answer `403 Forbidden` instead. We meet that in the next unit, with admins.

### Keep the data honest {#honest-data}

When we gave every shelf an owner, the [database made sure][owner id] that every shelf has one, and that the owner exists.
Our privacy settings need a rule like that too: **a public shelf should always have a public owner**.
A public shelf links to its owner's profile, and shows their avatar.
If the owner is private, that link and that picture lead nowhere.

Our app doesn't keep that rule yet.

1. Sign in as **testUser2**. Your profile is private.
2. On [My shelves][shelves], press **make public** on **sci-fi**.
The form posts to `/shelves/4/publish`, and the page flashes **Shelf 'sci-fi' is now public**.
3. Sign out, and open [the home page][home]. **sci-fi** is there under Public shelves, with a broken picture beside **testUser2**'s name.
The picture is broken because their profile is still private, so `/users/2/avatar` and `/users/2` both return `404 Not Found` for a visitor.
4. Follow the link to **testUser2**. It's not found.

Run `deno task db:reset` to put it right.
In the next unit, we will keep this rule: only a public user may share a shelf, and making your profile private makes your shelves private too.

{{<figure src="images/private-owner.webp" alt="The home page signed out, listing testUser2's sci-fi shelf with a broken avatar beside testUser2's name" caption="A public shelf with a private owner leads nowhere.">}}

## Conclusion

Every shelf now has a public or private setting, and new shelves are private.
The home page and profiles only list public shelves, filtered in SQL.
A private shelf is not found for everyone but its owner, and you can share a shelf, and take it back, from My shelves or from the shelf's own page.
Every permission question lives in **tools/permissions.js**, and is asked in the route and in the page, and we have seen one rule about our data that our app doesn't keep yet.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="68"}
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

## Challenge: Shared or not?

These are questions to investigate, not changes to keep.

- Make a new shelf on My shelves. Which section is it in, and why?
- As **testUser2**, open [a book's page][book 1]. Which shelves are you offered, and why doesn't the page need to ask `canUseShelf` about each one?
- In **tools/permissions.js**, try to `import { isOwner }` in **server/main.js**. What happens, and why is that a good thing?
- Write down a predicate that would answer *can this user share this shelf?*, so that the honest data rule holds. Don't add it to your code: it comes in the next unit.

<details>
<summary>A predicate for sharing</summary>

You can share a shelf if it's yours, and your own profile is public.

```JavaScript {linenos=false}
const canPublishShelf = (user, shelf) => isOwner(user, shelf) && Boolean(user.isPublic);
```

It would go in **tools/permissions.js**, next to `isOwner`, and it would be asked by a `may` guard on the publish route and by My shelves, before showing a **make public** button.

The book's page offers only your own shelves, because they come from `getShelvesByOwnerId`, filtered in SQL.
Every shelf on the list is yours, so there's nothing to check.

`isOwner` isn't exported, so importing it fails, and the server stops with an error.
The rest of the app can only ask the questions that **tools/permissions.js** chooses to answer.

</details>

> If you tried any changes, undo them, and run `deno task db:reset`, so that your code and data match the starting point of the next unit.

[previous exercise]: privacy/private-profiles/
[owners pattern]: owners-and-guards/owners/#pattern
[owner id]: owners-and-guards/owners/#owner-id
[my shelves]: owners-and-guards/owners/#my-shelves
[may use shelf]: owners-and-guards/owners/#may-use-shelf
[forged form]: owners-and-guards/owners/#forged-form
[everyones shelves]: profiles-and-avatars/profile-pages/#everyones-shelves
[home]: http://localhost:8000/
[shelves]: http://localhost:8000/shelves
[user 1]: http://localhost:8000/users/1
[shelf 1]: http://localhost:8000/shelves/1
[shelf 3]: http://localhost:8000/shelves/3
[book 1]: http://localhost:8000/books/1
[filter]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/filter

---
type: exercise
title: Signing up
weight: 1
author: Dr Graeme Stuart
---

In this exercise we will store users, so the sign up form can create a real account.
A password will be hashed before it reaches the database, and a username that's already taken will be rejected with a message, not a server error.

<!--more-->

> You should have the sign up and sign in forms, still stubs, from the [previous unit][validation schemas].
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

At the end of the last unit, every form was checked against a schema, but signing up still flashed that it ignored the registration, and signing in did the same.
By the end of this exercise you should be able to:

- Add a users table with a unique username and a hashed password
- Hash a password with PBKDF2 and a random salt, so the typed password never reaches the database
- Create a user from the validated sign up data, and redirect them to sign in
- Seed test users, awaiting each create because hashing takes a moment
- Reject a username that's already taken with a `400` and a message under the field

## A users table {#users-table}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Before the sign up form can do anything real, the database needs somewhere to keep users.
Update **tasks/db-create.js**.

**tasks/db-create.js**
```JavaScript{hl_lines="6-15"}
import { db } from "../server/models/db.js";

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
```

> The users table has a placeholder, because users will get more columns later.
> The shelves placeholder is still there too.

The username is `UNIQUE NOT NULL`, so SQLite itself refuses a second user with the same name.
The column is called `hashedPassword` because we will never store the password that was typed, only a hash of it.

Rebuild the database, so the new table is created.

```bash {linenos=false}
deno task db:reset
```

Nothing changes in the browser.
Sign up still flashes that it ignored the registration, because nothing writes to the table yet.

## Hash passwords {#hash-password}

A password must never be stored as it was typed.
Anyone who sees the database would see every password.
Create **server/tools/hash.js**.

**server/tools/hash.js**
```JavaScript{hl_lines="1-25"}
import { encodeHex } from "@std/encoding/hex";

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
```

`hashPassword` turns a password into a string that's safe to keep.

A private `derive` function runs [PBKDF2][pbkdf2] from the [Web Crypto API][web crypto], with SHA-256 and 600,000 iterations, so each guess is slow for an attacker.
`hashPassword` makes a random 16-byte salt for every password, so two users with the same password still get different hashes.
It returns the salt and the hash as hex, joined by a colon, ready for the `hashedPassword` column.

`derive` and `hashPassword` are [`async`][async], because the work takes a moment, and we wait for the result with `await`.
That idea is the same one we used when middleware had to wait for the rest of the chain.

`encodeHex` comes from `@std/encoding`.
Add it to **deno.json**.

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
        "@std/encoding": "jsr:@std/encoding@^1.0.11",
        "@std/html": "jsr:@std/html@^1.0.7",
        "@std/http": "jsr:@std/http@^1.1.4"
    }
}
```

`deno.lock` is updated too, the first time the new import is used.

Nothing calls `hashPassword` yet.
The users model will use it next.

## A users model {#create-user}

Create **server/models/users.js**.

**server/models/users.js**
```JavaScript{hl_lines="1-12"}
import { hashPassword } from "../tools/hash.js";
import { db } from "./db.js";

// Placeholder: this will need to change when users get more columns
export async function createUser({ username, password }) {
    const hashedPassword = await hashPassword(password);
    return db.prepare(`
        INSERT INTO users(username, hashedPassword)
        VALUES (:username, :hashedPassword)
        RETURNING *
    `).get({ username, hashedPassword });
}
```

> The placeholder is there because `createUser` will need to change when users get more columns.

`createUser` takes a username and a password.
It hashes the password with `hashPassword` first, so the plain password never reaches the database, then inserts the username and the hash into the users table.

Hashing takes a moment, so `createUser` is `async` and has to be awaited.
`RETURNING *` gives back the new row, including its id.
The sign up controller doesn't call `createUser` yet.

## Create the user {#register}

`createUserController` still ignores the registration.
Update **server/controllers/users.js**.

**server/controllers/users.js**
```JavaScript{hl_lines="1 11 13-15"}
import { createUser } from "../models/users.js";
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
    ctx.flash = `User '${user.username}' created`;
    return redirect("/signin", ctx);
}
```

The controller passes the validated username and password to `createUser` and waits for the new row, so the controller is `async` now.
It flashes that the user was created and redirects to the sign in page, ready for them to sign in.

Signing in is still a stub, so it doesn't check the password yet.
But the users table now has a row for each new user, with a hashed password and never the one they typed.

Open [sign up][signup], choose a new username, **alice**, and a password that meets the rules, `Alice123!`.
You land on the sign in page, with the message **User 'alice' created**.

{{<figure src="images/signup-created.webp" alt="The sign in page with the message User 'alice' created" caption="A new account is created, and you are sent to sign in.">}}

```plaintext {linenos=false}

============================
POST /signup
[flash sent] User 'alice' created
RESPONSE 303
============================

============================
GET /signin
[flash received] User 'alice' created
RESPONSE 200
============================
```

## Seed two test users {#seed-users}

Testing sign up and, soon, sign in means making the same users again after every `db:reset`.
Update **db/seed.json** to add a users list.

**db/seed.json**
```json{hl_lines="2-11"}
{
    "users": [
        {
            "username": "testUser1",
            "password": "testUser1!"
        },
        {
            "username": "testUser2",
            "password": "testUser2!"
        }
    ],
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
}
```

Each test user has a username and a password.
The passwords are in `seed.json` for testing only.
The database itself only ever holds their hashes.

Update **tasks/db-seed.js** to create each one with `createUser`.

**tasks/db-seed.js**
```JavaScript{hl_lines="3 14-17"}
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
    await createUser(user);
}

// Temporary: these shelves will belong to users once there are users
for (const shelf of seed.shelves) {
    const { id: shelfId } = createShelf({ title: shelf.title });
    for (const title of shelf.books) {
        const bookId = bookIds[title];
        if (!bookId) {
            throw new Error(`Book '${title}' not found in books.json`);
        }
        addBookToShelf(shelfId, bookId);
    }
}
```

`createUser` hashes the password, which takes a moment, so the loop awaits each one.
`db-seed.js` is a module, so it can use `await` at the top level.

The shelves are still listed on their own, without owners, until shelves belong to users.
The temporary comment stays.

Rebuild the database.

```bash {linenos=false}
deno task db:reset
```

The app looks the same, but the users table now has **testUser1** and **testUser2** every time you reset.

## Already taken {#unique-username}

Open [sign up][signup] and try to register as **testUser1**, with the password `testUser1!`.
SQLite refuses the insert, which is right, but the error reaches `withErrors` and you get a server error page with a `500`.

```plaintext {linenos=false}

============================
POST /signup
[error] UnexpectedError: Unexpected error
    at withErrors (file:///…/server/middleware/errors.js:10:21)
    at async withLogs (file:///…/server/middleware/logging.js:8:22) {
  [cause]: Error: UNIQUE constraint failed: users.username
      at unwrap (https://jsr.io/@db/sqlite/0.12.0/src/util.ts:37:11)
      at Statement.get (https://jsr.io/@db/sqlite/0.12.0/src/statement.ts:662:7)
      at createUser (file:///…/server/models/users.js:11:8)
      at async createUserController (file:///…/server/controllers/users.js:13:18)
      at async withErrors (file:///…/server/middleware/errors.js:6:16)
      at async withLogs (file:///…/server/middleware/logging.js:8:22)
}
RESPONSE 500
============================
```

The user should get a message under the username field, like any other bad input.
Add a `unique` factory to **server/tools/validators.js**.

**server/tools/validators.js**
```JavaScript{hl_lines="40-47"}
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
```

`unique(exists)` takes a function that says whether a value is already in use, and throws a `ValidationError` saying the field is already taken if it is.

Add `usernameExists` to **server/models/users.js**.

**server/models/users.js**
```JavaScript{hl_lines="13-21"}
import { hashPassword } from "../tools/hash.js";
import { db } from "./db.js";

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

`usernameExists` looks for a row with that username, and returns `true` or `false`.

Update **server/schema/user.js** to add `unique(usernameExists)` to the username's validators.

**server/schema/user.js**
```JavaScript{hl_lines="1-9 14"}
import { usernameExists } from "../models/users.js";
import {
    hasDigit,
    hasSymbol,
    maxLength,
    minLength,
    required,
    unique,
} from "../tools/validators.js";

export const userSchema = {
    username: {
        displayName: "Username",
        validators: [required, minLength(3), maxLength(50), unique(usernameExists)],
    },
    password: {
        displayName: "Password",
        validators: [required, minLength(8), hasDigit, hasSymbol],
    },
};
```

A taken name now gives the form back with a `400` and the message, like any other bad input.
The `UNIQUE` constraint stays on the column, as a last line of defence.

Try signing up as **testUser1** again.
The form comes back with **Username already taken.** under the username, and the username is kept.

{{<figure src="images/username-taken.webp" alt="The sign up page with testUser1 in the username field and Username already taken. under it" caption="A taken username is rejected with a message, not a server error.">}}

```plaintext {linenos=false}

============================
POST /signup
RESPONSE 400
============================
```

## Conclusion

We now have a users table, and a password is hashed with PBKDF2 and a random salt before it is stored, so the typed password never reaches the database.
The sign up form creates a real account and sends you to sign in, and two test users are seeded on every reset.
A username that's already taken is rejected with a `400` and a message under the field, while the `UNIQUE` constraint stays as a last line of defence.

Signing in is still a stub.
In the next exercise we will check the password, and remember who signed in.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="33 41"}
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
│   │   └── validation.js
│   ├── models/
│   │   ├── books.js
│   │   ├── db.js
│   │   ├── shelfBooks.js
│   │   ├── shelves.js
│   │   └── users.js
│   ├── schema/
│   │   ├── shelf.js
│   │   ├── signin.js
│   │   └── user.js
│   ├── tools/
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

## Challenge: Already taken, or a 500?

These are questions to investigate, not changes to keep.

- In **server/schema/user.js**, remove `unique(usernameExists)` and try signing up as **testUser1** again. What happens, and why is the `UNIQUE` constraint still useful?
- If `hashPassword` reused the same salt for every password, what would an attacker who stole the database be able to notice?
- Why does `createUser` have to be `async`?

<details>
<summary>The constraint, the salt, and the wait</summary>

Without `unique`, SQLite still refuses a second **testUser1**, but the error reaches `withErrors`, and the user gets the server error page with a `500`.
The `UNIQUE` constraint is the last line of defence: it still protects the table if a bug lets a taken name past the schema.

With the same salt every time, two users who chose the same password would have the same hash.
An attacker could see the match, and could hash common passwords once and compare them to every row.

`createUser` awaits `hashPassword`, which awaits the work in `derive`.
Only an `async` function can use `await`.

</details>

> If you tried any changes, undo them, and run `deno task db:reset`, so that your code and data match the starting point of the next exercise.

[validation schemas]: validation-schemas/
[async]: router-class/middleware/#async
[pbkdf2]: https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto/deriveBits#pbkdf2
[web crypto]: https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto
[signup]: http://localhost:8000/signup
[signin]: http://localhost:8000/signin

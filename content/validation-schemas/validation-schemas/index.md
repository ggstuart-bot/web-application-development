---
type: exercise
title: Validation schemas
weight: 1
author: Dr Graeme Stuart
---

In this exercise we will check every form against a schema, with one middleware that does the checking for every route.
Then we will add a sign in page, and let users rename their shelves.

<!--more-->

> You should have the sign up form and its schema from the [previous unit][validating input].
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

At the end of the last unit, the sign up form was checked against a schema, but the new shelf form still checked its title by hand.
Both controllers also had the same `try...catch`, which has nothing to do with shelves or users.
By the end of this exercise you should be able to:

- Describe a form's rules as a schema, and check it with `validateSchema`
- Write a function that makes a middleware, and use it to validate a form before its controller runs
- Give a route a fallback controller that shows the form again with its messages
- Reuse a schema and a form partial for a second form

## A schema for shelves {#shelf-schema}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

The new shelf form checks its title with its own list of validators, in the controller.
The sign up form already has a better way: a schema, checked with `validateSchema`.

Create **server/schema/shelf.js**.

**server/schema/shelf.js**
```JavaScript{hl_lines="1-8"}
import { maxLength, minLength, required } from "../tools/validators.js";

export const shelfSchema = {
    title: {
        displayName: "Title",
        validators: [required, minLength(3), maxLength(25)],
    },
};
```

`shelfSchema` has a single field, `title`, with the same validators as before.
Its display name is **Title**, so the messages read just as they did.

Update `createShelfController` in **server/controllers/shelves.js**.

**server/controllers/shelves.js**
```JavaScript{hl_lines="3 6-7 21-24 26-28"}
import { createShelf, deleteShelfById, getShelves } from "../models/shelves.js";
import { addThumbnails, getBooksByShelfId } from "../models/shelfBooks.js";
import { SchemaValidationError } from "../tools/errors.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { validateSchema } from "../tools/validation.js";
import { shelfSchema } from "../schema/shelf.js";
import { shelvesView } from "../views/shelves.js";
import { shelfView } from "../views/shelf.js";

export function shelvesController(ctx) {
    ctx.data.shelves = addThumbnails(getShelves());
    ctx.data.title = "Shelves";
    return render(shelvesView, ctx);
}

export async function createShelfController(ctx) {
    const { request } = ctx;
    const formData = await request.formData();
    try {
        const { title } = validateSchema(formData, shelfSchema);
        const shelf = createShelf({ title });
        ctx.flash = `Shelf '${shelf.title}' created`;
        return redirect(`/shelves/${shelf.id}`, ctx);
    } catch (error) {
        if (!(error instanceof SchemaValidationError)) throw error;
        ctx.data.errors = error.errors;
        ctx.data.values = error.values;
        ctx.status = 400;
        return shelvesController(ctx);
    }
}

export function shelfController(ctx) {
    const { data: { shelf } } = ctx;
    ctx.data.title = shelf.title;
    ctx.data.books = getBooksByShelfId(shelf.id);
    return render(shelfView, ctx);
}

export function deleteShelfController(ctx) {
    const { data: { shelf: { id } } } = ctx;
    const shelf = deleteShelfById(id);
    ctx.flash = `Shelf '${shelf.title}' deleted`;
    return redirect("/shelves", ctx);
}
```

The controller passes the form data and `shelfSchema` to `validateSchema`, and uses the `title` it returns.
A rejected title throws a `SchemaValidationError`, so the controller takes its `errors` and `values` for the view, exactly as the sign up controller does.

Nothing outside **server/tools/validation.js** calls `validateField` now.
Update **server/tools/validation.js** to remove its `export`.

**server/tools/validation.js**
```JavaScript{hl_lines="3"}
import { SchemaValidationError, ValidationError } from "./errors.js";

function validateField(name, value, validators) {
    for (const validate of validators) {
        validate(name, value);
    }
}

export function validateSchema(formData, schema) {
    const errors = {};
    const validated = {};
    const values = {};
    for (const [key, { validators, displayName }] of Object.entries(schema)) {
        const value = formData.get(key) ?? "";
        values[key] = value;
        try {
            validateField(displayName || key, value, validators);
            validated[key] = value;
        } catch (error) {
            if (!(error instanceof ValidationError)) throw error;
            errors[key] = error.message;
        }
    }
    if (Object.keys(errors).length) {
        throw new SchemaValidationError(errors, values);
    }
    return validated;
}
```

`validateField` is now a private helper for `validateSchema`.

Open [shelves][shelves] and try to create a shelf called **ab**.
The message and the typed title come back, just as before.

## Validation middleware {#with-validation}

Look at the `try...catch` in `createShelfController`.
None of it is about shelves.
Every form needs the same steps: read the form data, check it against a schema, and either carry on, or show the form again with the messages.

That sounds like a job for [middleware][middleware].
But each route has a different schema, and a different page to show when the form is rejected.
So we need a function that *makes* a middleware for each route.

Create **server/middleware/validation.js**.

**server/middleware/validation.js**
```JavaScript{hl_lines="1-19"}
import { SchemaValidationError } from "../tools/errors.js";
import { validateSchema } from "../tools/validation.js";

export function withValidation(schema, fallback) {
    return async (ctx, next) => {
        const { request } = ctx;
        const formData = await request.formData();
        try {
            ctx.validated = validateSchema(formData, schema);
            return next(ctx);
        } catch (error) {
            if (!(error instanceof SchemaValidationError)) throw error;
            ctx.data.errors = error.errors;
            ctx.data.values = error.values;
            ctx.status = 400;
            return fallback(ctx);
        }
    };
}
```

`withValidation(schema, fallback)` returns a new middleware, which remembers the `schema` and the `fallback` it was made with.
The middleware is [`async`][async], because reading the form data takes time.

- Good data goes on `ctx.validated`, and the next handler runs.
- A `SchemaValidationError` puts the errors and the values on `ctx.data`, sets a `400` status, and calls the `fallback` controller instead.
- Any other error is thrown again, for `withErrors` to deal with.

Update **server/main.js** to make `validateNewShelf` and put it in front of `createShelfController`.

**server/main.js**
```JavaScript{hl_lines="15-16 19-20 33"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import { addBookController, removeBookController } from "./controllers/shelfBooks.js";
import {
    createShelfController,
    deleteShelfController,
    shelfController,
    shelvesController,
} from "./controllers/shelves.js";
import { staticController } from "./controllers/static.js";
import { createUserController, signupFormController } from "./controllers/users.js";
import { Router } from "./tools/router.js";
import { withLogs } from "./middleware/logging.js";
import { withErrors } from "./middleware/errors.js";
import { withValidation } from "./middleware/validation.js";
import { shelfSchema } from "./schema/shelf.js";
import { withBook, withShelf } from "./middleware/resources.js";

const validateNewShelf = withValidation(shelfSchema, shelvesController);

const app = new Router();

app.use(withLogs);
app.use(withErrors);

app.get("/assets/*", staticController);
app.get("/", homeController);
app.get("/books", booksController);
app.get("/books/:bookId", withBook, bookController);
app.get("/signup", signupFormController);
app.post("/signup", createUserController);
app.get("/shelves", shelvesController);
app.post("/shelves", validateNewShelf, createShelfController);
app.get("/shelves/:shelfId", withShelf, shelfController);
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

`validateNewShelf` is made from `shelfSchema`, with `shelvesController` as the fallback, so a rejected title shows the shelves page again.

Now `createShelfController` only needs to create the shelf.
Update **server/controllers/shelves.js**.

**server/controllers/shelves.js**
```JavaScript{hl_lines="14-18"}
import { createShelf, deleteShelfById, getShelves } from "../models/shelves.js";
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

export function deleteShelfController(ctx) {
    const { data: { shelf: { id } } } = ctx;
    const shelf = deleteShelfById(id);
    ctx.flash = `Shelf '${shelf.title}' deleted`;
    return redirect("/shelves", ctx);
}
```

The controller reads the title from `ctx.validated`.
It doesn't read the form data itself any more, so it doesn't need to be `async`.

Try **ab** again: it's rejected just as before.
Create a shelf called **favourites**, and you go to its page with **Shelf 'favourites' created**.

## Sign up, with withValidation too {#validate-new-user}

`createUserController` has the same `try...catch` that we just took out of the new shelf controller.
Update **server/main.js** to make `validateNewUser`.

**server/main.js**
```JavaScript{hl_lines="16-17 21 33"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import { addBookController, removeBookController } from "./controllers/shelfBooks.js";
import {
    createShelfController,
    deleteShelfController,
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
import { withBook, withShelf } from "./middleware/resources.js";

const validateNewShelf = withValidation(shelfSchema, shelvesController);
const validateNewUser = withValidation(userSchema, signupFormController);

const app = new Router();

app.use(withLogs);
app.use(withErrors);

app.get("/assets/*", staticController);
app.get("/", homeController);
app.get("/books", booksController);
app.get("/books/:bookId", withBook, bookController);
app.get("/signup", signupFormController);
app.post("/signup", validateNewUser, createUserController);
app.get("/shelves", shelvesController);
app.post("/shelves", validateNewShelf, createShelfController);
app.get("/shelves/:shelfId", withShelf, shelfController);
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

`validateNewUser` is made from `userSchema`, with `signupFormController` as the fallback, and goes in front of `createUserController`.

Update **server/controllers/users.js**.

**server/controllers/users.js**
```JavaScript{hl_lines="10-13"}
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { signupView } from "../views/signup.js";

export function signupFormController(ctx) {
    ctx.data.title = "Sign up";
    return render(signupView, ctx);
}

export function createUserController(ctx) {
    const { validated } = ctx;
    ctx.flash = `Ignored registration for '${validated.username}'`;
    return redirect("/signup", ctx);
}
```

The controller loses its `try...catch` and reads the username from `ctx.validated`.
There are no users to create yet, so it still flashes that it ignored the registration and redirects back to the form.

`withValidation` puts *all* the rejected values on `ctx.data.values`, and that includes the password.
The sign up view only ever shows the username, so the password still never comes back in the page.

Try [sign up][signup] with a short password: the messages and the username come back, and the password field is empty.

## A sign in link {#signin-link}

Most visits to the site will be from people who already have an account.
The nav should offer sign in, rather than sign up.

Update **server/views/layout.js**.

**server/views/layout.js**
```JavaScript{hl_lines="24"}
import { html } from "../tools/html.js";

// Placeholder: this will need to change when we add sign-in
export function layout(content, ctx) {
    const { data, url } = ctx;
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
                        ${navLink("/signin", "sign in", url.pathname)}
                    </nav>
                </header>
                ${flashMessage(data.flash)}
                <main>
                ${content}
                </main>
                <footer class="secondary">
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

// A header link, marked as the current page when it matches the path exactly
function navLink(href, content, path) {
    return html`<a href="${href}" aria-current="${
        href === path ? "page" : "false"
    }">${content}</a>`;
}
```

> The placeholder comment is still true, because there's no real signing in yet.

There's no route for `/signin` yet, so the link gets the not found page for now.
The sign up page is still at [http://localhost:8000/signup][signup].

## A sign in page {#signin-page}

Signing in will start a session, so its controllers get a module of their own.
Create **server/controllers/sessions.js**.

**server/controllers/sessions.js**
```JavaScript{hl_lines="1-7"}
import { render } from "../tools/render.js";
import { signinView } from "../views/signin.js";

export function signinFormController(ctx) {
    ctx.data.title = "Sign in";
    return render(signinView, ctx);
}
```

Create **server/views/signin.js**.

**server/views/signin.js**
```JavaScript{hl_lines="1-19"}
import { html } from "../tools/html.js";

export function signinView() {
    return html`
        <section>
            <header>
                <h1>Sign in</h1>
                <p>Don't have an account? <a href="/signup">sign up</a> to gain access.</p>
            </header>
            <form class="secondary" method="POST">
                <label for="username">Username</label>
                <input id="username" name="username" autocomplete="username">
                <label for="password">Password</label>
                <input id="password" type="password" name="password" autocomplete="current-password">
                <button>sign in</button>
            </form>
        </section>
    `;
}
```

The form looks like the sign up form, but the password's `autocomplete` is `current-password`.
That tells a password manager to offer the saved password, rather than suggesting a new one.
The header links to the sign up page for anyone without an account, so the nav doesn't need a link to it.

Update **server/main.js** to add the route.

**server/main.js**
```JavaScript{hl_lines="18 35"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import { addBookController, removeBookController } from "./controllers/shelfBooks.js";
import {
    createShelfController,
    deleteShelfController,
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
import { signinFormController } from "./controllers/sessions.js";
import { withBook, withShelf } from "./middleware/resources.js";

const validateNewShelf = withValidation(shelfSchema, shelvesController);
const validateNewUser = withValidation(userSchema, signupFormController);

const app = new Router();

app.use(withLogs);
app.use(withErrors);

app.get("/assets/*", staticController);
app.get("/", homeController);
app.get("/books", booksController);
app.get("/books/:bookId", withBook, bookController);
app.get("/signup", signupFormController);
app.post("/signup", validateNewUser, createUserController);
app.get("/signin", signinFormController);
app.get("/shelves", shelvesController);
app.post("/shelves", validateNewShelf, createShelfController);
app.get("/shelves/:shelfId", withShelf, shelfController);
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

Follow the **sign in** link: the page opens, and links to sign up.
The form posts back to `/signin`, which has no `POST` route yet, so submitting it gets the not found page for now.

## Checking sign in {#signin-schema}

Signing in only needs both fields to be filled in.
The length and character rules belong to *choosing* a password, not to typing one you already have.

Create **server/schema/signin.js**.

**server/schema/signin.js**
```JavaScript{hl_lines="1-12"}
import { required } from "../tools/validators.js";

export const signinSchema = {
    username: {
        displayName: "Username",
        validators: [required],
    },
    password: {
        displayName: "Password",
        validators: [required],
    },
};
```

Each field has just the `required` validator, and a display name, so a missing username reads **Username is a required field.**, like the sign up form.

Add a stub `signinController` to **server/controllers/sessions.js**.

**server/controllers/sessions.js**
```JavaScript{hl_lines="1 9-14"}
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { signinView } from "../views/signin.js";

export function signinFormController(ctx) {
    ctx.data.title = "Sign in";
    return render(signinView, ctx);
}

export function signinController(ctx) {
    const { validated: { username } } = ctx;
    ctx.flash = `Ignored sign in for '${username}'`;
    return redirect("/signin", ctx);
}
```

There are no users to check the password against yet, so it flashes that it ignored the sign in, and redirects back to the sign in page.

Update **server/views/signin.js** to show each field's message under its input, and keep the typed username.

**server/views/signin.js**
```JavaScript{hl_lines="3-4 13-16 18-20"}
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
                <label for="username">Username</label>
                <input id="username" name="username" autocomplete="username" value="${values
                    ?.username ?? ""}"
                    aria-describedby="username-error" aria-invalid="${Boolean(errors?.username)}">
                <span id="username-error" class="error">${errors?.username ?? ""}</span>
                <label for="password">Password</label>
                <input id="password" type="password" name="password" autocomplete="current-password"
                    aria-describedby="password-error" aria-invalid="${Boolean(errors?.password)}">
                <span id="password-error" class="error">${errors?.password ?? ""}</span>
                <button>sign in</button>
            </form>
        </section>
    `;
}
```

Just like the sign up form, the password is never put back in the page.

Update **server/main.js** to make `validateSignin` and add the `POST` route.

**server/main.js**
```JavaScript{hl_lines="18-19 24 38"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import { addBookController, removeBookController } from "./controllers/shelfBooks.js";
import {
    createShelfController,
    deleteShelfController,
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
import { withBook, withShelf } from "./middleware/resources.js";

const validateNewShelf = withValidation(shelfSchema, shelvesController);
const validateNewUser = withValidation(userSchema, signupFormController);
const validateSignin = withValidation(signinSchema, signinFormController);

const app = new Router();

app.use(withLogs);
app.use(withErrors);

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

`validateSignin` is made from `signinSchema`, with `signinFormController` as the fallback.

Open [sign in][signin] and press **sign in** with both fields empty.
Both messages appear under their fields.
Fill in both and press it again, and the flash says **Ignored sign in for** your username.

```plaintext {linenos=false}

============================
POST /signin
[flash sent] Ignored sign in for 'testUser1'
RESPONSE 303
============================
```

{{<figure src="images/signin-errors.webp" alt="The sign in page with Username is a required field. and Password is a required field. under the empty inputs" caption="The sign in form, checked against signinSchema.">}}

## A form partial {#shelf-title-form}

The shelf page is about to get a form to rename the shelf.
It's the same form as the new shelf form: one title input with its message, and a button.
Only the action, the starting title and the button label are different.

Create **server/views/partials/shelfTitleForm.js**.

**server/views/partials/shelfTitleForm.js**
```JavaScript{hl_lines="1-13"}
import { html } from "../../tools/html.js";

export function shelfTitleForm(action, title, error, label) {
    return html`
        <form class="secondary" method="POST" action="${action}">
            <label for="title">Title</label>
            <input id="title" name="title" value="${title}"
                aria-describedby="title-error" aria-invalid="${Boolean(error)}">
            <span id="title-error" class="error">${error ?? ""}</span>
            <button>${label}</button>
        </form>
    `;
}
```

Update **server/views/shelves.js** to use it for the new shelf form.

**server/views/shelves.js**
```JavaScript{hl_lines="4 19"}
import { html } from "../tools/html.js";
import { DEFAULT_COVER } from "./partials/bookCard.js";
import { cardGrid } from "./partials/cardGrid.js";
import { shelfTitleForm } from "./partials/shelfTitleForm.js";

export function shelvesView(ctx) {
    const { data: { shelves, errors, values } } = ctx;

    return html`
        <section>
            <header>
                <h1>Shelves</h1>
            </header>
            <h2>All shelves</h2>
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

The new shelf form starts with an empty title, unless a rejected one came back.
The page looks and behaves exactly as before.

## A rename form {#rename-form}

Update **server/views/shelf.js** to add a rename form to the aside, above the delete button.

**server/views/shelf.js**
```JavaScript{hl_lines="5 22-23"}
import { html } from "../tools/html.js";
import { bookCard } from "./partials/bookCard.js";
import { cardGrid } from "./partials/cardGrid.js";
import { postButton } from "./partials/postButton.js";
import { shelfTitleForm } from "./partials/shelfTitleForm.js";

export function shelfView(ctx) {
    const { data: { shelf, books } } = ctx;

    return html`
        <section>
            <header>
                <h1>${shelf.title}</h1>
            </header>
            ${cardGrid(
                books.map((book) => bookCard(book, removeForm(shelf, book))),
                html`No books yet, <a href="/books">find some books</a> to add`,
            )}
            <p><a href="/shelves">Back to all shelves</a></p>
        </section>
        <aside>
            <h2>Rename shelf</h2>
            ${shelfTitleForm(`/shelves/${shelf.id}/edit`, shelf.title, "", "rename")}
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

It's the partial's second use.
It posts to `/shelves/:shelfId/edit`, starts with the shelf's current title, and its button says **rename**.
There are no errors to show yet, so it passes an empty one.

Open [**classics**][shelf 1]: the rename form shows **classics**.
Nothing handles the `POST` yet, so renaming gets the not found page for now.

## Renaming a shelf {#rename-shelf}

Add `updateShelfTitleById` to **server/models/shelves.js**.

**server/models/shelves.js**
```JavaScript{hl_lines="26-32"}
import { db } from "./db.js";

// Placeholder: this will need to change when shelves have owners
export function createShelf({ title }) {
    return db.prepare(`
        INSERT INTO shelves(title)
        VALUES (:title)
        RETURNING id, title
    `).get({ title });
}

// Placeholder: this will need to change when shelves have owners
export function getShelfById(id) {
    return db.prepare(`SELECT id, title FROM shelves WHERE id=:id`).get({ id });
}

// Placeholder: this will need to change when shelves have owners
export function getShelves() {
    return db.prepare(`
        SELECT id, title,
            (SELECT count(*) FROM shelfBooks WHERE shelfId = shelves.id) AS bookCount
        FROM shelves
    `).all();
}

// Placeholder: this will need to change when shelves have owners
export function updateShelfTitleById(id, title) {
    return db.prepare(
        `UPDATE shelves SET title=:title WHERE id=:id RETURNING id, title`,
    ).get({ id, title });
}

// Placeholder: this will need to change when shelves have owners
export function deleteShelfById(id) {
    return db.prepare(`DELETE FROM shelves WHERE id=:id RETURNING id, title`).get({ id });
}
```

> `updateShelfTitleById` gets the same placeholder comment as the other shelf functions, because they will all change when shelves have owners.

`RETURNING` gives back the renamed row, so the controller can use its new title.

Add `editShelfController` to **server/controllers/shelves.js**.

**server/controllers/shelves.js**
```JavaScript{hl_lines="1-6 33-39"}
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

It passes the shelf's id and the validated title to `updateShelfTitleById`, flashes the new name, and redirects to the shelf.

A rejected title needs the shelf page again, with the message and the typed title.
Update **server/views/shelf.js**.

**server/views/shelf.js**
```JavaScript{hl_lines="8 23-28"}
import { html } from "../tools/html.js";
import { bookCard } from "./partials/bookCard.js";
import { cardGrid } from "./partials/cardGrid.js";
import { postButton } from "./partials/postButton.js";
import { shelfTitleForm } from "./partials/shelfTitleForm.js";

export function shelfView(ctx) {
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
            <p><a href="/shelves">Back to all shelves</a></p>
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

The rename form starts with the typed title if one came back, or the shelf's own title if not.

A new title has the same rules as a new shelf's, so we can use the same schema.
Update **server/main.js**.

**server/main.js**
```JavaScript{hl_lines="7 24 44"}
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
import { withBook, withShelf } from "./middleware/resources.js";

const validateNewShelf = withValidation(shelfSchema, shelvesController);
const validateEditShelf = withValidation(shelfSchema, shelfController);
const validateNewUser = withValidation(userSchema, signupFormController);
const validateSignin = withValidation(signinSchema, signinFormController);

const app = new Router();

app.use(withLogs);
app.use(withErrors);

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

`validateEditShelf` is made from the same `shelfSchema`, with `shelfController` as the fallback.
`withShelf` runs first, so a missing shelf still gets the not found page, and the fallback always has a shelf to show.

Open [**classics**][shelf 1] and rename it to **ab**.
The shelf page comes back with **Title must be at least 3 characters.** and **ab** in the form, while the heading still says **classics**.
Rename it to **favourites**, and the flash says **Shelf renamed to 'favourites'**.

```plaintext {linenos=false}

============================
POST /shelves/1/edit
RESPONSE 400
============================

============================
POST /shelves/1/edit
[flash sent] Shelf renamed to 'favourites'
RESPONSE 303
============================
```

{{<figure src="images/rename-rejected.webp" alt="The classics shelf page with ab in the rename form and Title must be at least 3 characters. under it" caption="A rejected title comes back to the shelf page.">}}

## Conclusion

Every form in our app is now described by a schema: `shelfSchema` for new and renamed shelves, `userSchema` for sign up, and `signinSchema` for sign in.
`withValidation` checks the form before the controller runs, so our controllers only deal with good data from `ctx.validated`.
When a form is rejected, its fallback controller shows the form again with a `400`, the messages and the typed values.

Sign in and sign up are still stubs.
Next we will add users, and sessions, so that signing in means something.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="18 27 34-35 52 60"}
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
│   │   └── shelves.js
│   ├── schema/
│   │   ├── shelf.js
│   │   ├── signin.js
│   │   └── user.js
│   ├── tools/
│   │   ├── errors.js
│   │   ├── flash.js
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

## Challenge: Order matters

These are questions to investigate, not changes to keep.

- In **server/schema/shelf.js**, change `maxLength(25)` to `maxLength(10)`. Which forms does that change? Try a long title in each.
- In **server/main.js**, swap `withShelf` and `validateEditShelf` on the rename route. Open [**classics**][shelf 1], use the developer tools to change the rename form's `action` to `/shelves/99/edit`, and rename it to **ab**. What happens, and why?
- Why does `signinSchema` not use `userSchema`'s password rules?

<details>
<summary>Answers</summary>

Both shelf forms use `shelfSchema`, so creating and renaming a shelf both reject a title over 10 characters.

With the validation first, a rejected title calls `shelfController` before `withShelf` has loaded a shelf.
There's no `ctx.data.shelf`, so the controller fails, and you get the server error page with a `500`, instead of not found.
With `withShelf` first, the missing shelf is a `404` before the form is even read.

The password rules are for choosing a new password.
If they ever changed, people with older passwords could still sign in, because sign in only needs the fields filled in.

</details>

> If you tried any changes, undo them, and run `deno task db:reset`, so that your code and data match the starting point of the next unit.

[validating input]: validating-input/
[middleware]: router-class/middleware/#middleware
[async]: router-class/middleware/#async
[shelves]: http://localhost:8000/shelves
[signup]: http://localhost:8000/signup
[signin]: http://localhost:8000/signin
[shelf 1]: http://localhost:8000/shelves/1

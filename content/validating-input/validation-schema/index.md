---
type: exercise
title: A validation schema
weight: 2
author: Dr Graeme Stuart
---

In this exercise we will add a sign up page, with a form for a username and a password.
We will describe the whole form with a *schema*, check every field against it, and show every problem at once.

<!--more-->

> You should have the validated new shelf form from the [previous exercise][previous exercise].
> If you have been experimenting, run `deno task db:reset` to get back to the seed data.
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

Our new shelf form has one field, and `validateField` checks it.
Real forms have several fields, and each needs its own validators.
We don't have users yet, but they are coming, and a sign up form is a good place to start.
It won't create anyone yet: for now it just tells us what it would have done.
By the end of this exercise you should be able to:

- Build a sign up form with a password input and `autocomplete` hints
- Describe a form with a *schema*, an object that lists each field's validators
- Loop over an object's entries with `Object.entries`
- Collect every field's error into one `SchemaValidationError`, and show each under its field
- Give an error class a constructor, so it can carry extra data
- Check a value against a *regular expression*

## A sign up link {#signup-link}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

The sign up page will be at **/signup**.
Update **server/views/layout.js** to add a link to the nav.

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
                        ${navLink("/signup", "sign up", url.pathname)}
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

The link text is **sign up**, in lower case like the other nav links.

> The layout's placeholder is still there.
> It will change when users can sign in.

Click it, and we get the not found page, because there is no route for **/signup** yet.

## A sign up page {#signup-page}

Create **server/views/signup.js**.

**server/views/signup.js**
```JavaScript{hl_lines="1-19"}
import { html } from "../tools/html.js";

export function signupView() {
    return html`
        <section>
            <header>
                <h1>Sign up</h1>
                <p>Already have an account? <a href="/signin">sign in</a> to gain access.</p>
            </header>
            <form class="secondary" method="POST">
                <label for="username">Username</label>
                <input id="username" name="username" autocomplete="username">
                <label for="password">Password</label>
                <input id="password" type="password" name="password" autocomplete="new-password">
                <button>sign up</button>
            </form>
        </section>
    `;
}
```

- The form asks for a username and a password, with a `<label>` for each, just like the new shelf form.
- The password input has `type="password"`, so the browser hides what is typed.
- The [`autocomplete`][autocomplete] attributes help a password manager. `username` says which field is the username, and `new-password` asks it to suggest a new password, rather than fill in an old one.
- The form has `method="POST"` but no `action`. A form without an `action` is sent back to the address of the page it is on, so this one will post to **/signup**.
- The header links to a sign in page, which we haven't written yet.

Users are a new part of our application, so they get a new controller module.
Create **server/controllers/users.js**.

**server/controllers/users.js**
```JavaScript{hl_lines="1-7"}
import { render } from "../tools/render.js";
import { signupView } from "../views/signup.js";

export function signupFormController(ctx) {
    ctx.data.title = "Sign up";
    return render(signupView, ctx);
}
```

Register the route in **server/main.js**.

**server/main.js**
```JavaScript{hl_lines="11 26"}
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
import { signupFormController } from "./controllers/users.js";
import { Router } from "./tools/router.js";
import { withLogs } from "./middleware/logging.js";
import { withErrors } from "./middleware/errors.js";
import { withBook, withShelf } from "./middleware/resources.js";

const app = new Router();

app.use(withLogs);
app.use(withErrors);

app.get("/assets/*", staticController);
app.get("/", homeController);
app.get("/books", booksController);
app.get("/books/:bookId", withBook, bookController);
app.get("/signup", signupFormController);
app.get("/shelves", shelvesController);
app.post("/shelves", createShelfController);
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

Click the **sign up** link again.

{{<figure src="images/signup-page.webp" alt="The Sign up page, with books, shelves and a filled sign up link in the nav, a line that says Already have an account? sign in to gain access, and a dark form with Username and Password boxes and a sign up button" caption="The sign up page.">}}

Type a username and a password and press **sign up**.
We get the not found page, because there is no `POST` route for **/signup**.
The **sign in** link is not found too, for now.

## A stub controller {#create-user-stub}

Before we check anything, we will make sure the form reaches the server.

Add a `createUserController` to **server/controllers/users.js**.

**server/controllers/users.js**
```JavaScript{hl_lines="1 9-16"}
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { signupView } from "../views/signup.js";

export function signupFormController(ctx) {
    ctx.data.title = "Sign up";
    return render(signupView, ctx);
}

export async function createUserController(ctx) {
    const { request } = ctx;
    const formData = await request.formData();
    const username = formData.get("username") ?? "";
    ctx.flash = `Ignored registration for '${username}'`;
    return redirect("/signup", ctx);
}
```

There are no users to create yet.
For now the controller reads the username, [flashes][set flash] a message to say that the registration was ignored, and redirects back to the sign up page.
It doesn't touch the password at all.

Register it in **server/main.js**, with `app.post`.

**server/main.js**
```JavaScript{hl_lines="11 27"}
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
import { withBook, withShelf } from "./middleware/resources.js";

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
app.post("/shelves", createShelfController);
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

Sign up as **alice**, with any password.
We land back on the sign up page, with the message **Ignored registration for 'alice'**.
The terminal shows the redirect.

```plaintext {linenos=false}

============================
POST /signup
[flash sent] Ignored registration for 'alice'
RESPONSE 303
============================
```

Now sign up with both fields empty.
That works too, with **Ignored registration for ''**.
We need to check the form.

## A schema {#user-schema}

The sign up form has two fields, and each needs its own validators.
We could call `validateField` twice in the controller, but every new form would need the same code again.
Instead, we will describe the form with a *schema*: an object that says which validators each field needs.

Create a **schema** folder inside the **server** folder, with a file called **user.js**.

**server/schema/user.js**
```JavaScript{hl_lines="1-6"}
import { maxLength, minLength, required } from "../tools/validators.js";

export const userSchema = {
    username: [required, minLength(3), maxLength(50)],
    password: [required, minLength(8)],
};
```

`userSchema` has a property for each field in the form.
Each property is the list of validators for that field, reusing the ones we wrote for the shelf title.

- A username must be there, with between 3 and 50 characters.
- A password must be there, with at least 8 characters.

Now we need a function that checks a form against a schema.
Add `validateSchema` to **server/tools/validation.js**.

**server/tools/validation.js**
```JavaScript{hl_lines="6-12"}
export function validateField(name, value, validators) {
    for (const validate of validators) {
        validate(name, value);
    }
}

export function validateSchema(formData, schema) {
    for (const [key, validators] of Object.entries(schema)) {
        const value = formData.get(key) ?? "";
        validateField(key, value, validators);
    }
}
```

- [`Object.entries`][object entries] turns an object into an array of `[key, value]` pairs. For `userSchema`, that is `["username", [...]]` and `["password", [...]]`.
- `const [key, validators]` takes each pair apart with [array destructuring][array destructuring].
- For each field, we read its value from the form, with `?? ""` just as in the shelf controller, and check it with `validateField`, using the key as the field's name.

`validateSchema` doesn't return anything yet.
The first `ValidationError` stops it, and escapes to the caller.

Update `createUserController` in **server/controllers/users.js** to use it.

**server/controllers/users.js**
```JavaScript{hl_lines="1 4-5 16-25"}
import { ValidationError } from "../tools/errors.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { validateSchema } from "../tools/validation.js";
import { userSchema } from "../schema/user.js";
import { signupView } from "../views/signup.js";

export function signupFormController(ctx) {
    ctx.data.title = "Sign up";
    return render(signupView, ctx);
}

export async function createUserController(ctx) {
    const { request } = ctx;
    const formData = await request.formData();
    try {
        validateSchema(formData, userSchema);
        const username = formData.get("username") ?? "";
        ctx.flash = `Ignored registration for '${username}'`;
        return redirect("/signup", ctx);
    } catch (error) {
        if (!(error instanceof ValidationError)) throw error;
        ctx.status = 400;
        return signupFormController(ctx);
    }
}
```

The controller reads the form data once, and passes it to `validateSchema` with `userSchema`.
If the form is fine, it carries on with the stub's flash and redirect.
If a `ValidationError` comes out, it sets the status to `400`, and shows the form again with `signupFormController`, just as the shelf controller does.

Sign up as **ab**, with the password **sekrit**.
The form comes back empty, with no message, and the network panel shows a `400`, just like the first version of the shelf form.
Sign up as **alice** with the password **Passw0rd!**, and the stub's flash and redirect still work.

## Returning the values {#validated}

The controller still reads the username straight from the form data.
It would be safer to only use values that have been checked.

Update `validateSchema` in **server/tools/validation.js** to return the checked values.

**server/tools/validation.js**
```JavaScript{hl_lines="8 12 14"}
export function validateField(name, value, validators) {
    for (const validate of validators) {
        validate(name, value);
    }
}

export function validateSchema(formData, schema) {
    const validated = {};
    for (const [key, validators] of Object.entries(schema)) {
        const value = formData.get(key) ?? "";
        validateField(key, value, validators);
        validated[key] = value;
    }
    return validated;
}
```

Once a field has passed all its validators, its value is added to the `validated` object, under the field's key.
At the end, `validateSchema` returns that object.

Update **server/controllers/users.js** to take the username from it.

**server/controllers/users.js**
```JavaScript{hl_lines="17"}
import { ValidationError } from "../tools/errors.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { validateSchema } from "../tools/validation.js";
import { userSchema } from "../schema/user.js";
import { signupView } from "../views/signup.js";

export function signupFormController(ctx) {
    ctx.data.title = "Sign up";
    return render(signupView, ctx);
}

export async function createUserController(ctx) {
    const { request } = ctx;
    const formData = await request.formData();
    try {
        const { username } = validateSchema(formData, userSchema);
        ctx.flash = `Ignored registration for '${username}'`;
        return redirect("/signup", ctx);
    } catch (error) {
        if (!(error instanceof ValidationError)) throw error;
        ctx.status = 400;
        return signupFormController(ctx);
    }
}
```

`const { username } = validateSchema(...)` takes the username out of the returned object with destructuring.
Nothing changes in the browser.

## Errors under their fields {#schema-errors}

The message from a `ValidationError` doesn't say which field it came from.
To show it under the right input, `validateSchema` needs to say which field failed.
We will do that with a new error that carries an object of messages, just like `ctx.data.errors`.

Add a `SchemaValidationError` to **server/tools/errors.js**.

**server/tools/errors.js**
```JavaScript{hl_lines="6-12"}
export class NotFoundError extends Error {}

export class UnexpectedError extends Error {}

export class ValidationError extends Error {}

export class SchemaValidationError extends Error {
    constructor(errors) {
        super("validation failed");
        this.errors = errors;
    }
}
```

This error class has a [`constructor`][constructor], like our [`Router`][router class].

- It takes an `errors` object.
- [`super`][super] runs the constructor of `Error`, the class it extends, with the message **validation failed**. Every error needs a message, even though we won't show this one.
- `this.errors = errors` keeps the errors object on the error, so whoever catches it can read it.

Update **server/tools/validation.js**.

**server/tools/validation.js**
```JavaScript{hl_lines="1-2 13-20"}
import { SchemaValidationError, ValidationError } from "./errors.js";

export function validateField(name, value, validators) {
    for (const validate of validators) {
        validate(name, value);
    }
}

export function validateSchema(formData, schema) {
    const validated = {};
    for (const [key, validators] of Object.entries(schema)) {
        const value = formData.get(key) ?? "";
        try {
            validateField(key, value, validators);
        } catch (error) {
            if (!(error instanceof ValidationError)) throw error;
            const errors = {};
            errors[key] = error.message;
            throw new SchemaValidationError(errors);
        }
        validated[key] = value;
    }
    return validated;
}
```

Each call to `validateField` is now wrapped in its own `try`.

- If the field fails, we make an `errors` object, and store the message under the field's key. `errors[key]` uses square brackets, because the key is in a variable: when `key` is `"username"`, it sets `errors.username`.
- Then we throw a `SchemaValidationError` carrying that object.
- Any other error is thrown again unchanged, just as in the shelf controller.

Update **server/controllers/users.js** to catch the new error.

**server/controllers/users.js**
```JavaScript{hl_lines="1 21-22"}
import { SchemaValidationError } from "../tools/errors.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { validateSchema } from "../tools/validation.js";
import { userSchema } from "../schema/user.js";
import { signupView } from "../views/signup.js";

export function signupFormController(ctx) {
    ctx.data.title = "Sign up";
    return render(signupView, ctx);
}

export async function createUserController(ctx) {
    const { request } = ctx;
    const formData = await request.formData();
    try {
        const { username } = validateSchema(formData, userSchema);
        ctx.flash = `Ignored registration for '${username}'`;
        return redirect("/signup", ctx);
    } catch (error) {
        if (!(error instanceof SchemaValidationError)) throw error;
        ctx.data.errors = error.errors;
        ctx.status = 400;
        return signupFormController(ctx);
    }
}
```

The controller now catches a `SchemaValidationError`, and puts its `errors` on `ctx.data.errors`.

Update **server/views/signup.js** to show them.

**server/views/signup.js**
```JavaScript{hl_lines="3-4 13-15 17-19"}
import { html } from "../tools/html.js";

export function signupView(ctx) {
    const { data: { errors } } = ctx;
    return html`
        <section>
            <header>
                <h1>Sign up</h1>
                <p>Already have an account? <a href="/signin">sign in</a> to gain access.</p>
            </header>
            <form class="secondary" method="POST">
                <label for="username">Username</label>
                <input id="username" name="username" autocomplete="username"
                    aria-describedby="username-error" aria-invalid="${Boolean(errors?.username)}">
                <span id="username-error" class="error">${errors?.username ?? ""}</span>
                <label for="password">Password</label>
                <input id="password" type="password" name="password" autocomplete="new-password"
                    aria-describedby="password-error" aria-invalid="${Boolean(errors?.password)}">
                <span id="password-error" class="error">${errors?.password ?? ""}</span>
                <button>sign up</button>
            </form>
        </section>
    `;
}
```

Each input gets a message span under it, with `aria-describedby` and `aria-invalid`, exactly as in the [new shelf form][error message].
The view takes `ctx` now, because it needs the errors.

Sign up with both fields empty.
The form says **username is a required field.**, under the username.
The first bad field still stops the check, so there is no message for the password yet.
And the message starts with the key, `username`, in lower case.
We will fix both of these next.

## Every field at once {#all-errors}

Stopping at the first bad field means the user fixes the form one message at a time.
It would be better to check every field, then report every problem together.

Update `validateSchema` in **server/tools/validation.js**.

**server/tools/validation.js**
```JavaScript{hl_lines="10 16 21-23"}
import { SchemaValidationError, ValidationError } from "./errors.js";

export function validateField(name, value, validators) {
    for (const validate of validators) {
        validate(name, value);
    }
}

export function validateSchema(formData, schema) {
    const errors = {};
    const validated = {};
    for (const [key, validators] of Object.entries(schema)) {
        const value = formData.get(key) ?? "";
        try {
            validateField(key, value, validators);
            validated[key] = value;
        } catch (error) {
            if (!(error instanceof ValidationError)) throw error;
            errors[key] = error.message;
        }
    }
    if (Object.keys(errors).length) {
        throw new SchemaValidationError(errors);
    }
    return validated;
}
```

- The `errors` object has moved outside the loop, so every field's message goes into the same object.
- A field that fails no longer throws. Its message is added to `errors`, and the loop carries on to the next field.
- A field that passes is added to `validated`, inside the `try`, so only good values are returned.
- After the loop, [`Object.keys(errors)`][object keys] is an array of the fields that failed. If it has any, we throw one `SchemaValidationError` with all of them.

The controller and the view don't change.
They already handle any number of errors.

Sign up with both fields empty again.
This time there is a message under each field.

## Display names {#display-name}

The messages use each field's key, so they start with a lower case **username** or **password**.
The shelf form's messages say **Title**, because we give `validateField` the name we want to show.
The schema needs to say what each field should be called.

Update **server/schema/user.js**.

**server/schema/user.js**
```JavaScript{hl_lines="4-11"}
import { maxLength, minLength, required } from "../tools/validators.js";

export const userSchema = {
    username: {
        displayName: "Username",
        validators: [required, minLength(3), maxLength(50)],
    },
    password: {
        displayName: "Password",
        validators: [required, minLength(8)],
    },
};
```

Each field is now an object, with a `displayName` and its list of `validators`.

Update **server/tools/validation.js** to use the display name.

**server/tools/validation.js**
```JavaScript{hl_lines="12 15"}
import { SchemaValidationError, ValidationError } from "./errors.js";

export function validateField(name, value, validators) {
    for (const validate of validators) {
        validate(name, value);
    }
}

export function validateSchema(formData, schema) {
    const errors = {};
    const validated = {};
    for (const [key, { validators, displayName }] of Object.entries(schema)) {
        const value = formData.get(key) ?? "";
        try {
            validateField(displayName || key, value, validators);
            validated[key] = value;
        } catch (error) {
            if (!(error instanceof ValidationError)) throw error;
            errors[key] = error.message;
        }
    }
    if (Object.keys(errors).length) {
        throw new SchemaValidationError(errors);
    }
    return validated;
}
```

- `[key, { validators, displayName }]` takes each pair apart, and takes the field's object apart at the same time.
- `displayName || key` uses the display name, or falls back to the key for a field that doesn't have one.

Sign up with both fields empty again.
The messages now read **Username is a required field.** and **Password is a required field.**

## Keeping the username {#keep-username}

Just like the shelf title, a rejected username should stay in the form.

Update **server/tools/errors.js**, so that a `SchemaValidationError` carries the values too.

**server/tools/errors.js**
```JavaScript{hl_lines="8 11"}
export class NotFoundError extends Error {}

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

Update **server/tools/validation.js** to collect every value it reads.

**server/tools/validation.js**
```JavaScript{hl_lines="12 15 25"}
import { SchemaValidationError, ValidationError } from "./errors.js";

export function validateField(name, value, validators) {
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

`values` collects every value, good or bad, while `validated` only holds the values that passed.
If anything failed, `values` goes with the errors.

Update **server/controllers/users.js**.

**server/controllers/users.js**
```JavaScript{hl_lines="23"}
import { SchemaValidationError } from "../tools/errors.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { validateSchema } from "../tools/validation.js";
import { userSchema } from "../schema/user.js";
import { signupView } from "../views/signup.js";

export function signupFormController(ctx) {
    ctx.data.title = "Sign up";
    return render(signupView, ctx);
}

export async function createUserController(ctx) {
    const { request } = ctx;
    const formData = await request.formData();
    try {
        const { username } = validateSchema(formData, userSchema);
        ctx.flash = `Ignored registration for '${username}'`;
        return redirect("/signup", ctx);
    } catch (error) {
        if (!(error instanceof SchemaValidationError)) throw error;
        ctx.data.errors = error.errors;
        ctx.data.values = { username: error.values.username };
        ctx.status = 400;
        return signupFormController(ctx);
    }
}
```

The controller only passes the *username* back to the view.
The password is never sent back to the browser, so it has to be typed again.
That's what users expect from a sign up form, and it keeps the password out of the page.

Update **server/views/signup.js** to put the username back in its input.

**server/views/signup.js**
```JavaScript{hl_lines="4 13-14"}
import { html } from "../tools/html.js";

export function signupView(ctx) {
    const { data: { errors, values } } = ctx;
    return html`
        <section>
            <header>
                <h1>Sign up</h1>
                <p>Already have an account? <a href="/signin">sign in</a> to gain access.</p>
            </header>
            <form class="secondary" method="POST">
                <label for="username">Username</label>
                <input id="username" name="username" autocomplete="username" value="${values
                    ?.username ?? ""}"
                    aria-describedby="username-error" aria-invalid="${Boolean(errors?.username)}">
                <span id="username-error" class="error">${errors?.username ?? ""}</span>
                <label for="password">Password</label>
                <input id="password" type="password" name="password" autocomplete="new-password"
                    aria-describedby="password-error" aria-invalid="${Boolean(errors?.password)}">
                <span id="password-error" class="error">${errors?.password ?? ""}</span>
                <button>sign up</button>
            </form>
        </section>
    `;
}
```

The line is split in an odd place, between `values` and `?.username`, but it means just the same as `values?.username ?? ""` on one line.

Sign up as **ab**, with the password **sekrit**.

{{<figure src="images/signup-errors.webp" alt="The Sign up form in a dark panel, with ab still in the Username box and the message Username must be at least 3 characters under it, and an empty Password box with the message Password must be at least 8 characters under it" caption="Every problem is shown at once, and the username is kept.">}}

Both messages are there, and **ab** is still in the username input.
View the page source (**Ctrl+U**), and you won't find **sekrit** anywhere.

## Password rules {#password-rules}

A password of eight letters is easy to guess.
We will also ask for a digit and a symbol.
To check for those, we need a *regular expression*.

### Regular expressions {#regex}

A [regular expression][regex] is a pattern that text can match.
In JavaScript, it is written between slashes, and its [`test`][regex test] method tells us whether a string contains a match.
Here is a demonstration, with the results in comments.

```JavaScript
/\d/.test("Password1"); // true, there is a digit
/\d/.test("Password"); // false

/[^A-Za-z0-9\s]/.test("Passw0rd!"); // true, "!" is a symbol
/[^A-Za-z0-9\s]/.test("Password1"); // false
```

- `\d` matches any one digit, from `0` to `9`.
- Square brackets make a *character class*, which matches any one of the characters inside. `[A-Za-z0-9]` would match any letter or digit.
- A `^` at the start of the class turns it around, so it matches any character that *isn't* in it. `\s` is any white space, such as a space or a tab.
- So `[^A-Za-z0-9\s]` matches anything that isn't a letter, a digit or white space, which is what we mean by a symbol.

The patterns don't have to match the whole password, just one character somewhere in it.

### hasDigit and hasSymbol {#has-digit}

Add two validators to **server/tools/validators.js**.

**server/tools/validators.js**
```JavaScript{hl_lines="28-39"}
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
```

Each one tests the value against its pattern, and throws a `ValidationError` if there is no match.
They don't need a limit, so they are plain validators, like `required`, rather than factories.

Add them to the password's list in **server/schema/user.js**.

**server/schema/user.js**
```JavaScript{hl_lines="1 10"}
import { hasDigit, hasSymbol, maxLength, minLength, required } from "../tools/validators.js";

export const userSchema = {
    username: {
        displayName: "Username",
        validators: [required, minLength(3), maxLength(50)],
    },
    password: {
        displayName: "Password",
        validators: [required, minLength(8), hasDigit, hasSymbol],
    },
};
```

That's all it takes.
The schema already runs every validator in order, so a password is checked for its length first, then for a digit, then for a symbol.

Sign up as **alice** with the password **Password1**.
The form says **Password must include a symbol.**, and **alice** is kept.
Then try **Passw0rd!**, and we get the stub's flash and redirect.

## Conclusion

We have a sign up page, with a form for a username and a password.
`userSchema` describes the form, with a display name and a list of validators for each field.
`validateSchema` checks every field against the schema, and either returns the values that passed, or throws one `SchemaValidationError` with a message for every field that failed and the values that were sent.
The controller shows the messages under their fields, and keeps the username, but never the password.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="21 32 55"}
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
│   │   ├── shelfBooks.js
│   │   ├── shelves.js
│   │   ├── static.js
│   │   └── users.js
│   ├── middleware/
│   │   ├── errors.js
│   │   ├── logging.js
│   │   └── resources.js
│   ├── models/
│   │   ├── books.js
│   │   ├── db.js
│   │   ├── shelfBooks.js
│   │   └── shelves.js
│   ├── schema/
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
│   │   │   └── searchForm.js
│   │   ├── book.js
│   │   ├── books.js
│   │   ├── error.js
│   │   ├── home.js
│   │   ├── layout.js
│   │   ├── shelf.js
│   │   ├── shelves.js
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

The new shelf form still calls `validateField` itself.
In the next unit it will get a schema of its own, and we will add a sign in page and a way to rename a shelf.

## Challenge: Test the sign up form

These are questions to investigate, not changes to keep.

- Sign up with the password **Pässw0rd**. Is it accepted? Why?
- Sign up with a username of three spaces. Is it accepted?
- Why does `validateSchema` read every field with `?? ""`? Use the developer tools' **elements** panel to change the password input's `name` to something else, and sign up. What message do you get?
- Write a `hasUpperCase` validator, and add it to the password's list. Where in the list should it go?

<details>
<summary>What to expect</summary>

**Pässw0rd** is accepted.
`ä` isn't one of the letters `A` to `Z` or `a` to `z`, so `[^A-Za-z0-9\s]` counts it as a symbol.
Our idea of a "letter" only covers plain English letters.

Three spaces pass `required` and `minLength(3)`, just like the shelf title, so the stub flashes **Ignored registration for '   '**.

With the input renamed, the form has no `password` field, so `formData.get("password")` is `null`.
`?? ""` turns that into an empty string, and the form says **Password is a required field.**
Without it, `required` would read `null.length`, and throw a `TypeError` that isn't a `ValidationError`, so we would get the server error page.

A `hasUpperCase` validator could use `/[A-Z]/`.
It could go anywhere after `minLength(8)`, but each field shows one message, so the order decides which problem the user hears about first.

</details>

> If you tried any changes, undo them, so that your code matches the starting point of the next unit.

[previous exercise]: validating-input/rejecting-bad-input/
[error message]: validating-input/rejecting-bad-input/#error-message
[set flash]: post-redirect-flash/cookies-and-flash/#set-flash
[router class]: router-class/router/#class
[array destructuring]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring#array_destructuring
[autocomplete]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Attributes/autocomplete
[object entries]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/entries
[object keys]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/keys
[constructor]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/constructor
[super]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/super
[regex]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Regular_expressions
[regex test]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/RegExp/test

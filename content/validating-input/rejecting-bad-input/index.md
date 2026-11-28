---
type: exercise
title: Rejecting bad input
weight: 1
author: Dr Graeme Stuart
---

In this exercise we will stop the new shelf form from accepting a bad title.
An empty title, a title that is too short and a title that is too long will all be sent back, with a message that says what is wrong.

<!--more-->

> You should have shelves that you can create and delete, with books that you can add and remove, from the previous unit.
> If you have been experimenting, run `deno task db:reset` to get back to the seed data.
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

Our users can change our data now.
They can make shelves, delete them, and put books on them.
But our server believes everything it is sent.
When we added the [new shelf form][create shelf], we noticed that an empty title makes a shelf with no title at all.
In this exercise we will fix that.
By the end of this exercise you should be able to:

- Reject bad input with a `400 Bad Request`, and show the form again rather than redirecting
- Show an error message beside the field it belongs to, in a way that screen readers can find
- Write a *validator*, a function that throws a `ValidationError` when a value isn't acceptable
- Write a *factory*, a function that makes validators, such as `minLength(3)`
- Run a list of validators on a field with `validateField`
- Keep a rejected value in the form, so the user can fix it
- Explain why checks in the browser can never replace checks on the server

## An empty title {#empty-title}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Visit [http://localhost:8000/shelves][shelves], leave the title empty, and press **create**.
We get a new shelf page with an empty heading, and the flash message **Shelf '' created**.
Nothing stopped us.

Update `createShelfController` in **server/controllers/shelves.js**.

**server/controllers/shelves.js**
```JavaScript{hl_lines="17-22"}
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

export async function createShelfController(ctx) {
    const { request } = ctx;
    const formData = await request.formData();
    const title = formData.get("title") ?? "";
    if (!title) {
        ctx.status = 400;
        return shelvesController(ctx);
    }
    const shelf = createShelf({ title });
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

- We read the title first. If the form has no `title` field at all, `formData.get` returns `null`, so `?? ""` gives us an empty string instead. Either way, `title` is a string.
- An empty string is [falsy][falsy], so `!title` is `true` when the title is empty.
- For an empty title, we set `ctx.status` to [`400 Bad Request`][400], and show the shelves page again with `shelvesController`. The `return` means the shelf is never created.
- A good title carries on as before.

`render` already passes `ctx.status` on to the response, so the shelves page goes back with a `400`.

Notice that a rejected form is *not* answered with a redirect.
We [redirect after a `POST`][post redirect get] that changed something, so that a refresh can't make the change again.
Here nothing has changed, so we answer with the page, and the `400` says that the request was bad.

Delete the shelf with no title, then go back to [http://localhost:8000/shelves][shelves] and try an empty title again, this time with the developer tools (**F12**) open on the **network** panel.

The page looks exactly the same as before.
There is no shelf, but there is no message either.
The only sign that anything went wrong is in the network panel, where the `POST` to `/shelves` has the status `400`.
The terminal shows it too.

```plaintext {linenos=false}

============================
POST /shelves
RESPONSE 400
============================
```

A good title such as **Tom's picks** still gets a `303` to the new shelf, with its flash message.

## Saying why {#error-message}

A user who doesn't look at the network panel has no idea why nothing happened.
We need to tell them, beside the field they need to fix.

Update `createShelfController` in **server/controllers/shelves.js** to send a message to the view.

**server/controllers/shelves.js**
```JavaScript{hl_lines="19"}
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

export async function createShelfController(ctx) {
    const { request } = ctx;
    const formData = await request.formData();
    const title = formData.get("title") ?? "";
    if (!title) {
        ctx.data.errors = { title: "Title is a required field." };
        ctx.status = 400;
        return shelvesController(ctx);
    }
    const shelf = createShelf({ title });
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

`ctx.data.errors` is an object with a message for each field that has a problem.
The key is the field's name, `title`, so the view knows where to show the message.

Now update **server/views/shelves.js** to show it.

**server/views/shelves.js**
```JavaScript{hl_lines="6 20-22"}
import { html } from "../tools/html.js";
import { DEFAULT_COVER } from "./partials/bookCard.js";
import { cardGrid } from "./partials/cardGrid.js";

export function shelvesView(ctx) {
    const { data: { shelves, errors } } = ctx;

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
            <form class="secondary" method="POST" action="/shelves">
                <label for="title">Title</label>
                <input id="title" name="title"
                    aria-describedby="title-error" aria-invalid="${Boolean(errors?.title)}">
                <span id="title-error" class="error">${errors?.title ?? ""}</span>
                <button>create</button>
            </form>
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

The view takes `errors` from `ctx.data`.
It is only there when the title was rejected, so most of the time it is `undefined`.

- `errors?.title` uses [optional chaining][optional chaining]. If `errors` is `undefined`, `errors?.title` is `undefined` too, rather than throwing a `TypeError`.
- The message goes in a `<span>` with the id `title-error`, just below the input. With no message, `?? ""` leaves the span empty.
- [`aria-describedby`][aria-describedby] points the input at that span, so a screen reader reads the message along with the field.
- [`aria-invalid`][aria-invalid] says whether the field has an error. `Boolean` turns the message (or `undefined`) into `true` or `false`, so it is `"true"` only when there is a message.

Try an empty title again.
This time the form says **Title is a required field.**, just under the input.

## Styling the errors {#error-styles}

The message looks just like any other text in the form.
It should stand out next to its field.

Update the `form.secondary` rule in **public/styles.css**.

**public/styles.css**
```css {linenostart="207", hl_lines="1 5-13"}
/* A secondary form is padded with rounded corners, and its errors have an accent bar */
form.secondary {
    padding: 1.5rem;
    border-radius: 0.5rem;
    .error {
        padding-inline-start: 0.5em;
        border-inline-start: 0.25em solid var(--highlight);
        font-size: 0.8em;
        /* An empty error is hidden */
        &:empty {
            display: none;
        }
    }
}
```

- An error is a little smaller than the text around it, with an accent bar down its left side, using the same `--highlight` colour as our [flash message][flash styles].
- The error span is always in the form, even when it is empty. [`:empty`][empty] matches an element with nothing in it, so `&:empty` hides an empty error, rather than leaving an empty bar under every field.

The comment above the rule now mentions the errors too.

Try an empty title again.

{{<figure src="images/title-required.webp" alt="The Shelves page, with the shelves link filled in the nav, and the New shelf form in a dark panel in the right-hand aside, with an empty Title box and the message Title is a required field underneath, marked with a red bar down its left side" caption="An empty title is rejected, with a message under the field.">}}

## A validator {#required}

Checking that a field isn't empty is something we will need again and again.
Rather than write an `if` in every controller, we will write a *validator*: a function that takes a field's name and value, and *throws an error* if the value isn't acceptable.

First, add a `ValidationError` to **server/tools/errors.js**.

**server/tools/errors.js**
```JavaScript{hl_lines="4-5"}
export class NotFoundError extends Error {}

export class UnexpectedError extends Error {}

export class ValidationError extends Error {}
```

It is an `Error` with its own name, just like [`NotFoundError`][not found error], so we can tell it apart from other errors.

Create **server/tools/validators.js**, with our first validator.

**server/tools/validators.js**
```JavaScript{hl_lines="1-7"}
import { ValidationError } from "./errors.js";

export function required(name, value) {
    if (!value.length) {
        throw new ValidationError(`${name} is a required field.`);
    }
}
```

`required` throws a `ValidationError` if the value has no characters.
The message uses the field's name, so the same validator works for any field.

Now update **server/controllers/shelves.js** to use it.

**server/controllers/shelves.js**
```JavaScript{hl_lines="3 8 20-24"}
import { createShelf, deleteShelfById, getShelves } from "../models/shelves.js";
import { addThumbnails, getBooksByShelfId } from "../models/shelfBooks.js";
import { ValidationError } from "../tools/errors.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { shelvesView } from "../views/shelves.js";
import { shelfView } from "../views/shelf.js";
import { required } from "../tools/validators.js";

export function shelvesController(ctx) {
    ctx.data.shelves = addThumbnails(getShelves());
    ctx.data.title = "Shelves";
    return render(shelvesView, ctx);
}

export async function createShelfController(ctx) {
    const { request } = ctx;
    const formData = await request.formData();
    const title = formData.get("title") ?? "";
    try {
        required("Title", title);
    } catch (error) {
        if (!(error instanceof ValidationError)) throw error;
        ctx.data.errors = { title: error.message };
        ctx.status = 400;
        return shelvesController(ctx);
    }
    const shelf = createShelf({ title });
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

We call `required` inside a [`try`][try catch] block.

- If the title is fine, `required` returns without doing anything, and the controller carries on and creates the shelf.
- If `required` throws, we land in the `catch` block. The error's message goes on `ctx.data.errors`, just as before.
- If the error *isn't* a `ValidationError`, we throw it again. A real bug should still reach [`withErrors`][with errors], and show the server error page, rather than being shown to the user as a message about their title.

Try an empty title again.
The page looks just the same as before, but the message now comes from the validator.

## Validator factories {#validate-field}

A shelf title should also be a sensible length.
We will say between 3 and 25 characters.

We could write a `minLength3` validator and a `maxLength25` validator, but then we would need another for every limit.
Instead, we will write a function that *makes* validators.

### A function that makes a function {#factories}

Here is a demonstration of the idea, with the results in comments.

```JavaScript
function minLength(len) {
    return (name, value) => {
        if (value.length < len) {
            throw new ValidationError(`${name} must be at least ${len} characters.`);
        }
    };
}

const atLeast3 = minLength(3); // a validator, just like required
atLeast3("Title", "poetry"); // returns, because "poetry" is long enough
atLeast3("Title", "ab"); // throws "Title must be at least 3 characters."
```

`minLength` doesn't check anything itself.
It returns a new [arrow function][arrow functions], with the same `(name, value)` shape as `required`.
That new function remembers the `len` it was made with, so `minLength(3)` checks for 3 characters, and `minLength(8)` would check for 8.
A function that makes other functions like this is called a *factory*.

### minLength and maxLength {#min-max-length}

Add `minLength` and `maxLength` to **server/tools/validators.js**.

**server/tools/validators.js**
```JavaScript{hl_lines="8-27"}
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
```

`maxLength` is the same idea the other way round.

A title now has three validators to pass.
Running a list of validators on a value is something every form will need, so it gets a function of its own.

Create **server/tools/validation.js**.

**server/tools/validation.js**
```JavaScript{hl_lines="1-5"}
export function validateField(name, value, validators) {
    for (const validate of validators) {
        validate(name, value);
    }
}
```

`validateField` runs each validator in turn on the same name and value.
The first validator to throw stops the loop, and the error escapes to whoever called `validateField`.
So the form shows one message at a time, starting with the most basic problem.

Update **server/controllers/shelves.js** to use it.

**server/controllers/shelves.js**
```JavaScript{hl_lines="6 9 22"}
import { createShelf, deleteShelfById, getShelves } from "../models/shelves.js";
import { addThumbnails, getBooksByShelfId } from "../models/shelfBooks.js";
import { ValidationError } from "../tools/errors.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { validateField } from "../tools/validation.js";
import { shelvesView } from "../views/shelves.js";
import { shelfView } from "../views/shelf.js";
import { maxLength, minLength, required } from "../tools/validators.js";

export function shelvesController(ctx) {
    ctx.data.shelves = addThumbnails(getShelves());
    ctx.data.title = "Shelves";
    return render(shelvesView, ctx);
}

export async function createShelfController(ctx) {
    const { request } = ctx;
    const formData = await request.formData();
    const title = formData.get("title") ?? "";
    try {
        validateField("Title", title, [required, minLength(3), maxLength(25)]);
    } catch (error) {
        if (!(error instanceof ValidationError)) throw error;
        ctx.data.errors = { title: error.message };
        ctx.status = 400;
        return shelvesController(ctx);
    }
    const shelf = createShelf({ title });
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

The controller calls `validateField` with the name **Title**, the title, and its three validators.
`required` goes in the list as it is, but `minLength(3)` and `maxLength(25)` are *calls*, which make the two validators we need.
A `ValidationError` from any of them is caught just as before.

Try a title of **ab**.
The form says **Title must be at least 3 characters.**
Then try **a title that is far too long for a shelf**, and the form says **Title must be no more than 25 characters.**

## Keeping the title {#keep-value}

When a long title is rejected, the input comes back empty, and the user has to type it all again.
It would be much kinder to give them back what they typed, so they can shorten it.

Update **server/controllers/shelves.js** to send the rejected title back.

**server/controllers/shelves.js**
```JavaScript{hl_lines="26"}
import { createShelf, deleteShelfById, getShelves } from "../models/shelves.js";
import { addThumbnails, getBooksByShelfId } from "../models/shelfBooks.js";
import { ValidationError } from "../tools/errors.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { validateField } from "../tools/validation.js";
import { shelvesView } from "../views/shelves.js";
import { shelfView } from "../views/shelf.js";
import { maxLength, minLength, required } from "../tools/validators.js";

export function shelvesController(ctx) {
    ctx.data.shelves = addThumbnails(getShelves());
    ctx.data.title = "Shelves";
    return render(shelvesView, ctx);
}

export async function createShelfController(ctx) {
    const { request } = ctx;
    const formData = await request.formData();
    const title = formData.get("title") ?? "";
    try {
        validateField("Title", title, [required, minLength(3), maxLength(25)]);
    } catch (error) {
        if (!(error instanceof ValidationError)) throw error;
        ctx.data.errors = { title: error.message };
        ctx.data.values = { title };
        ctx.status = 400;
        return shelvesController(ctx);
    }
    const shelf = createShelf({ title });
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

`ctx.data.values` holds the values the user sent, keyed by field, just like `ctx.data.errors`.

Update **server/views/shelves.js** to put it back in the input.

**server/views/shelves.js**
```JavaScript{hl_lines="6 20"}
import { html } from "../tools/html.js";
import { DEFAULT_COVER } from "./partials/bookCard.js";
import { cardGrid } from "./partials/cardGrid.js";

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
            <form class="secondary" method="POST" action="/shelves">
                <label for="title">Title</label>
                <input id="title" name="title" value="${values?.title ?? ""}"
                    aria-describedby="title-error" aria-invalid="${Boolean(errors?.title)}">
                <span id="title-error" class="error">${errors?.title ?? ""}</span>
                <button>create</button>
            </form>
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

The input's `value` is the rejected title, or an empty string when there isn't one, so the input starts empty as before.

Try **a title that is far too long for a shelf** again.
This time the title is still in the input, under the message, ready to be fixed.

{{<figure src="images/title-kept.webp" alt="The New shelf form in a dark panel, with the long title still in the Title box, slightly cut off at the end, and the message Title must be no more than 25 characters underneath" caption="A rejected title is kept, so it can be fixed.">}}

In the picture the box is a little narrow, so the end of the title is slightly cut off.

Now try **"Quotes" are fine here, honest**.
It is too long, so it comes back, quotes and all.
The user typed this, so it is untrusted, and it is going inside an attribute.
Our [`html` helper][html] escapes it, so the quotes can't end the `value` attribute early.

## Checks in the browser {#browser-checks}

HTML can check some things for us.
An input with the `required` attribute won't let the browser send an empty form, and `minlength` and `maxlength` can limit its length.
These are good for our users, because they get an answer straight away, without a trip to the server.

But they are never enough on their own.
We already [changed a form in the developer tools][wrong method], and anyone can do the same to remove a `required` attribute or change a `maxlength`.
A request doesn't even have to come from our form at all.

So the server must always check what it is sent.
Checks in the browser are a convenience for honest users, and checks on the server are what keep our data honest.

## Conclusion

A shelf title has to be between 3 and 25 characters now.
A bad title gets the shelves page again, with a `400`, a message under the field and the title kept in the input.

The checks are made by *validators*, functions that throw a `ValidationError` when a value isn't acceptable.
`minLength` and `maxLength` are *factories* that make validators for any limit, and `validateField` runs a list of validators on one field.
The controller catches a `ValidationError` and shows its message, and lets any other error through to `withErrors`.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="37 38"}
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
│   │   └── static.js
│   ├── middleware/
│   │   ├── errors.js
│   │   ├── logging.js
│   │   └── resources.js
│   ├── models/
│   │   ├── books.js
│   │   ├── db.js
│   │   ├── shelfBooks.js
│   │   └── shelves.js
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
│   │   └── shelves.js
│   └── main.js
├── tasks/
│   ├── db-create.js
│   ├── db-seed.js
│   └── server.js
├── .gitignore
├── deno.json
└── deno.lock
```

The shelf form has just one field.
In the next exercise we will add a sign up form with two fields, and check the whole form at once.

## Challenge: Test the title

These are questions to investigate, not changes to keep.

- Try a title of three spaces. What happens, and why? How might a validator stop it?
- Try a title of exactly 3 characters, and exactly 25. Are they accepted?
- In `validateField`'s list, move `required` after `minLength(3)`. What message does an empty title get now? Why does the order matter?
- Add `required` to the title input in **server/views/shelves.js**, and try an empty title. Then remove the attribute in the developer tools' **elements** panel, and try again. Which check stopped you each time?

<details>
<summary>What to expect</summary>

Three spaces have a length of 3, so they pass `required`, `minLength(3)` and `maxLength(25)`, and we get a shelf whose title is just spaces.
A validator could check `value.trim().length` instead, because [`trim`][trim] removes the white space from both ends.

Both are accepted: the limits are "at least 3" and "no more than 25".

With `minLength(3)` first, an empty title gets **Title must be at least 3 characters.** That's true, but not as helpful as saying the field is required.
The first validator to throw decides the message, so the most basic check goes first.

With the attribute, the browser stops the form before it is sent, and nothing appears in the terminal.
Without it, the request reaches the server, and our validator sends back the `400`.

</details>

> If you tried any changes, undo them, and run `deno task db:reset`, so that your code and data match the starting point of the next exercise.

[shelves]: http://localhost:8000/shelves
[create shelf]: post-redirect-flash/form-that-posts/#create-shelf
[post redirect get]: post-redirect-flash/form-that-posts/#post-redirect-get
[wrong method]: post-redirect-flash/form-that-posts/#wrong-method
[flash styles]: post-redirect-flash/cookies-and-flash/#flash-styles
[falsy]: user-input-and-security/search/#title
[html]: user-input-and-security/escaping/#html
[not found error]: router-class/router/#not-found-error
[try catch]: router-class/router/#catch
[with errors]: router-class/middleware/#with-errors
[arrow functions]: javascript-fundamentals/book-search/#arrow-functions
[400]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/400
[optional chaining]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining
[aria-describedby]: https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Attributes/aria-describedby
[aria-invalid]: https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Attributes/aria-invalid
[empty]: https://developer.mozilla.org/en-US/docs/Web/CSS/:empty
[trim]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/trim

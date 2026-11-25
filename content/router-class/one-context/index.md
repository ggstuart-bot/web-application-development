---
type: exercise
title: One context
weight: 3
author: Dr Graeme Stuart
---

In this exercise we will give our controllers, views and layout one shared context object, and use it to highlight the current page in the nav.

<!--more-->

> You should have a router from the [previous exercise][previous exercise].
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

Like the previous exercise, most of this one is a refactor.
By the end of this exercise you should be able to:

- Pass one context object through our controllers, `render`, views and layout
- Put each page's data on `ctx.data`
- Mark the current page in the nav with `aria-current`, and style it

## Where we are {#debt}

Our controllers already receive our [context object][ctx], but they don't do much with it.
Instead, they give `render` an options object with a [`status`][options], a [`title`][title] and some [`data`][passing data].

That has two problems.

- Every new piece of information has needed a new option, and changes to `render`.
- The views get `{ data }`, the layout gets `{ title }`, and the controllers get `ctx`. Three different shapes for one request.

We said that `ctx` would grow as our app does.
It's time to let it carry everything a page needs.

Start the server in development mode, if it isn't already running.

```bash {linenos=false}
deno task dev
```

## A data object {#data}

Start with **server/main.js**, which makes `ctx`.

**server/main.js**
```JavaScript{hl_lines="16 26"}
import { homeController } from "./controllers/home.js";
import { booksController } from "./controllers/books.js";
import { staticController } from "./controllers/static.js";
import { notFoundController } from "./controllers/notFound.js";
import { NotFoundError } from "./tools/errors.js";
import { Router } from "./tools/router.js";

const app = new Router();

app.register("/assets/*", staticController);
app.register("/", homeController);
app.register("/books", booksController);

export function server(request) {
    const url = new URL(request.url);
    const ctx = { request, url, data: {} };

    // Placeholder: this will need to change when we add middleware
    console.log(request.method, url.pathname);

    try {
        return app.handle(ctx);
    } catch (error) {
        if (error instanceof NotFoundError) {
            console.log("Not found:", error.message);
            return notFoundController(ctx);
        }
        throw error;
    }
}
```

Every request now gets its own `ctx` with an empty `data` object.
`ctx.data` is where the controller and any middleware collect everything this response's HTML needs, in one place that both the view and the layout read from.
The layout only uses `title` (and, later, `flash`).
The not-found controller is given `ctx` too, like every other controller.

> The middleware placeholder is still there.
> We will replace it in the next exercise.

Nothing uses `ctx.data` yet, so refresh [http://localhost:8000/books][books] to check that nothing has changed.

## One context everywhere {#ctx-data}

`render`, the layout, the views and every controller all have to change for this.
That's a lot at once, so we will make the changes one file at a time.
Some pages will break along the way, and we will fix them one by one.

### render takes ctx {#render-ctx}

Update **server/tools/render.js** to take `ctx` instead of an options object.

**server/tools/render.js**
```JavaScript{hl_lines="4-5 7-9"}
import { layout } from "../views/layout.js";

// Placeholder: this will need to change when we add flash messages and sign-in
export function render(view, ctx) {
    const { status } = ctx;
    const headers = new Headers({ "content-type": "text/html" });
    const content = view(ctx);
    const html = layout(content, ctx);
    return new Response(html, { headers, status });
}
```

`render` takes the `status` out of `ctx`, then passes the whole of `ctx` to the view and to the layout.
We have also given `content` and `html` names of their own, so each line does one thing.

> The placeholder hasn't changed.
> Flash messages and sign-in will need `render` to do more.

Update **server/views/layout.js** to take its title from `ctx.data`.

**server/views/layout.js**
```JavaScript{hl_lines="2-4"}
// Placeholder: this will need to change when we add flash messages, sign-in and nav highlighting
export function layout(content, ctx) {
    const { data } = ctx;
    const title = data.title ? `${data.title} | Bookshelves` : "Bookshelves";

    return `
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
                        <a href="/books">books</a>
                    </nav>
                </header>
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
```

The layout takes what it needs out of `ctx`, with [destructuring][destructuring].

> The layout placeholder hasn't changed yet.
> We will deal with nav highlighting in the next step.

This version only partly works.
Refresh [http://localhost:8000/][localhost].
It is `500 Internal Server Error`, and the terminal explains why.

```plaintext {linenos=false}
TypeError: Cannot destructure property 'status' of 'ctx' as it is undefined.
    at render (file:///…/server/tools/render.js:5:13)
    at Object.homeController [as handler] (file:///…/server/controllers/home.js:5:12)
```

The home controller still calls `render(homeView)`, so `ctx` is `undefined`.
[http://localhost:8000/nope][nope] is a `500` too.
[http://localhost:8000/books][books] still shows the books, because its old options object happens to have a `data` property, but its tab now says just **Bookshelves**.

Let's fix the controllers one at a time.

### The home page {#home-ctx}

**server/controllers/home.js**
```JavaScript{hl_lines="4-5"}
import { render } from "../tools/render.js";
import { homeView } from "../views/home.js";

export function homeController(ctx) {
    return render(homeView, ctx);
}
```

The home controller has nothing to add, but `render` needs `ctx`, so it now declares it and passes it on.

Refresh [http://localhost:8000/][localhost].
It is `200 OK` again, and its tab says **Bookshelves**.

### The books page {#books-ctx}

The books controller puts its title and its books on `ctx.data`.

**server/controllers/books.js**
```JavaScript{hl_lines="7-10"}
import { getBooks } from "../models/books.js";
import { render } from "../tools/render.js";
import { booksView } from "../views/books.js";

const PAGE_SIZE = 16;

export function booksController(ctx) {
    ctx.data.books = getBooks(PAGE_SIZE);
    ctx.data.title = "Books";
    return render(booksView, ctx);
}
```

The view takes the books from `ctx`.

**server/views/books.js**
```JavaScript{hl_lines="3-4"}
const DEFAULT_COVER = "/assets/default-cover.svg";

export function booksView(ctx) {
    const { data: { books } } = ctx;

    return `
        <section>
            <header>
                <h1>Books</h1>
            </header>
            <ul class="grid">
                ${books.map(bookCard).join("")}
            </ul>
        </section>
    `;
}

function bookCard(book) {
    const cover = book.coverUrl ?? DEFAULT_COVER;
    return `
        <li class="card book">
            <img class="cover" src="${cover}" alt="" width="200" height="300" loading="lazy">
            <div>
                <h2>${book.title}</h2>
                <p>${book.author} (${book.year})</p>
            </div>
        </li>
    `;
}
```

`const { data: { books } } = ctx;` takes `data` out of `ctx`, and then `books` out of `data`.
It is the same destructuring the view used before, moved inside the function.
`bookCard` hasn't changed.

> Our home and not-found views take no arguments, so they simply ignore `ctx`.

Refresh [http://localhost:8000/books][books].
The tab says **Books | Bookshelves** again.

### The not-found page {#not-found-ctx}

**server/controllers/notFound.js**
```JavaScript{hl_lines="4-7"}
import { render } from "../tools/render.js";
import { notFoundView } from "../views/notFound.js";

export function notFoundController(ctx) {
    ctx.status = 404;
    ctx.data.title = "Not found";
    return render(notFoundView, ctx);
}
```

The not-found controller sets `ctx.status` to `404`, and puts its title on `ctx.data`.

Refresh [http://localhost:8000/nope][nope].
It is `404 Not Found` again, and its tab says **Not found | Bookshelves**.

### One shape for everything {#one-shape}

Nothing has changed for our users.
But now a controller really is a function which takes the request context and returns a `Response`, and the views and the layout take the same context.
When a page needs something new, its controller puts it on `ctx`, and `render` doesn't have to change.

## The current page {#nav-link}

Now that the layout has `ctx`, it can find out which page it is wrapping.
Our router put the `url` on `ctx`, so the layout can compare each nav link with `url.pathname`.

Update **server/views/layout.js**.

**server/views/layout.js**
```JavaScript{hl_lines="1 3 20 36-39"}
// Placeholder: this will need to change when we add flash messages and sign-in
export function layout(content, ctx) {
    const { data, url } = ctx;
    const title = data.title ? `${data.title} | Bookshelves` : "Bookshelves";

    return `
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
                    </nav>
                </header>
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

// A header link, marked as the current page when it matches the path exactly
function navLink(href, content, path) {
    return `<a href="${href}" aria-current="${href === path ? "page" : "false"}">${content}</a>`;
}
```

`navLink` builds one link in the header.
If the link's `href` is the path of the current page, it gets [`aria-current="page"`][aria-current].
That tells screen readers, and our stylesheet, which link is the current page.
The [conditional operator][conditional operator] picks `"page"` or `"false"`.

`navLink` isn't exported, because only the layout uses it.

> The placeholder no longer mentions nav highlighting, because that's done.
> Flash messages and sign-in are still to come.

Open [http://localhost:8000/books][books], and look at the nav in the **elements** panel.

```html
<nav>
    <a href="/books" aria-current="page">books</a>
</nav>
```

On [http://localhost:8000/][localhost] the same link has `aria-current="false"`.
`"false"` means the same as leaving the attribute out: this link is not the current page.
The page looks the same, because nothing in our stylesheet uses the attribute yet.

## Filling the current link {#nav-styles}

Add a rule to **public/styles.css**, after the `a` rule.

**public/styles.css**
```css {linenostart="28"}
/* The current page's nav link is filled with the accent */
nav a[aria-current="page"] {
    background: var(--highlight);
    color: contrast-color(var(--highlight)); /* black or white, whichever is easier to read */
}
```

- [`[aria-current="page"]`][attribute selectors] is an *attribute selector*. It only matches the link for the current page, and fills it with our accent colour.
- [`contrast-color()`][contrast-color] picks black or white, whichever is easier to read on the accent, and the inline comment says so.

Refresh [http://localhost:8000/books][books].
The **books** link in the header is filled with the accent colour.
On [http://localhost:8000/][localhost] it still looks like any other link.

## Outlined pills {#nav-pills}

Now add a `nav a` rule to the end of **public/styles.css**.

**public/styles.css**
```css {linenostart="141"}
/* Nav links are outlined pills, underlined on hover */
nav a {
    padding: 0.25em 0.75em;
    border: 1px solid;
    border-radius: 1em;
    background: none;
    color: inherit;
    text-decoration: none;
    /* Hovering adds an accent underline */
    &:hover {
        text-decoration: underline var(--highlight) 2px;
    }
}
```

`nav a` gives every link in the nav an outline with round ends, like a pill, and takes away the underline.
The `&:hover` is nested inside it, like the `& > header` in our `body` rule, and its comment says that hovering adds an accent underline.

> The attribute selector makes the first rule more specific than `nav a`, so it wins even though it comes first in the file.

Refresh [http://localhost:8000/books][books].
The **books** link is still filled with the accent colour.

{{<figure src="images/nav-current.webp" alt="The books page, with the books link in the dark header filled with the red accent colour, as a pill with round ends" caption="On the books page, the books link is filled with the accent.">}}

On [http://localhost:8000/][localhost] it is an outlined pill.

{{<figure src="images/nav-pill.webp" alt="The home page, with the books link in the dark header drawn as an outlined pill with no fill" caption="On the home page, the books link is an outlined pill.">}}

## Conclusion

We paid off another piece of technical debt.
Our controllers, `render`, views and layout now share one context object, and each page's data goes on `ctx.data`.
The layout used it straight away, to mark the current page in the nav, and our stylesheet made the current page's link stand out.

We changed a lot of files, but we didn't add any, so our project looks just as it did at the end of the [previous exercise][project].

Next, we will replace our logging placeholder with our first *middleware*.

## Challenge: Experiment with the nav

- Add an about page with a view, a controller and a route, and a second `navLink` for it in the layout. Does the highlight move when you click between the pages?
- Change `--hue` in **public/styles.css** to a few different values. When does `contrast-color()` pick black text, and when white?

> Undo your changes, so that your code matches the version above before moving on.

[previous exercise]: router-class/router/
[project]: router-class/router/#conclusion
[ctx]: views-and-controllers/controllers/#ctx
[options]: views-and-controllers/controllers/#options
[title]: views-and-controllers/controllers/#title
[passing data]: sqlite-and-models/books-page/#passing-data
[destructuring]: javascript-fundamentals/books-data/#destructuring
[localhost]: http://localhost:8000/
[books]: http://localhost:8000/books
[nope]: http://localhost:8000/nope
[aria-current]: https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Attributes/aria-current
[conditional operator]: views-and-controllers/controllers/#title
[attribute selectors]: https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Selectors/Attribute_selectors
[contrast-color]: https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Values/color_value/contrast-color

---
type: exercise
title: Book pages
weight: 1
author: Dr Graeme Stuart
---

In this exercise we will give every book a page of its own, at an address like **/books/1**.
We will link to it from the books page, and teach our router to read the book's id from the path.

<!--more-->

> You should have the books page with search, escaping and pagination from the [user input and security][previous unit] unit.
> Your database should hold all 1,000 books. If you have been experimenting, run `deno task db:reset` to get back to the seed data.
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

So far our application has one page that lists the books, a page at a time.
A card shows a book's cover, title, author and year, but there is nowhere to go to see just one book.
In this unit every book, and then every shelf, becomes a *resource* with an address of its own, and our pages link them together.
By the end of this exercise you should be able to:

- Link each item in a list to a page of its own
- Name part of a path in a route pattern, and read it from `ctx.params`
- Load one row from the database by its id, and throw a `NotFoundError` when there isn't one
- Build a new page in small steps: a stub first, then its data, then its markup and styles
- Export a value from one module so that another module can share it
- Put items side by side in a row that wraps when there isn't room

## Links to book pages {#book-links}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Each book will have its own page, and the address of that page will include the book's `id`.
The Hunger Games is book 1, so its page will be **/books/1**.

Update **server/views/books.js** so that each card's title links to its book's page.

**server/views/books.js**
```JavaScript{hl_lines="31"}
import { html } from "../tools/html.js";
import { searchForm } from "./partials/searchForm.js";

const DEFAULT_COVER = "/assets/default-cover.svg";

export function booksView(ctx) {
    const { data: { title, books, q, page, pageCount } } = ctx;
    const results = books.length
        ? html`<ul class="grid">${books.map(bookCard)}</ul>`
        : html`<p>${q ? `No books match "${q}"` : "No books found"}</p>`;

    return html`
        <section>
            <header>
                <h1>${title}</h1>
                ${q ? html`<p><a href="/books">show all books</a></p>` : ""}
            </header>
            ${searchForm(q)}
            ${results}
            ${pagination(q, page, pageCount)}
        </section>
    `;
}

function bookCard(book) {
    const cover = book.coverUrl ?? DEFAULT_COVER;
    return html`
        <li class="card book">
            <img class="cover" src="${cover}" alt="" width="200" height="300" loading="lazy">
            <div>
                <h2><a href="/books/${book.id}">${book.title}</a></h2>
                <p>${book.author}${book.year ? ` (${book.year})` : ""}</p>
            </div>
        </li>
    `;
}

// Previous, the current page and next. Nothing at all if there's only one page
function pagination(q, page, pageCount) {
    if (pageCount <= 1) return "";
    return html`<nav aria-label="Pagination">
            ${page > 1 ? html`<a href="${pageUrl(q, page - 1)}" rel="prev">previous</a>` : ""}
            <a href="${pageUrl(q, page)}" aria-current="page">page ${page} of ${pageCount}</a>
            ${page < pageCount ? html`<a href="${pageUrl(q, page + 1)}" rel="next">next</a>` : ""}
        </nav>`;
}

// Only keep q in the link if there's a search
function pageUrl(q, page) {
    const params = q ? { q, page } : { page };
    return `/books?${new URLSearchParams(params)}`;
}
```

We use the `id` rather than the title, because every book has a different `id`, and it never changes.
Titles can be long, they can contain spaces and punctuation, and two books can even share a title.

Visit [http://localhost:8000/books][books] and click on **The Hunger Games**.
The address bar shows [http://localhost:8000/books/1][book 1], and we get our not found page.
The terminal explains why.

```plaintext {linenos=false}

============================
GET /books/1
[error] /books/1 doesn't match any known route
RESPONSE 404
============================
```

There is no route for **/books/1** yet.
That's fine.
A link can arrive before the page it points to, and until then our [not found error][not found error] does its job.

## Route parameters {#route-params}

We can't register a route for every book.
There are 1,000 of them, and there will be more one day.
Instead, we need one route that matches **/books/1**, **/books/2** and every other book, and tells us which book was asked for.

### A pattern with a name {#url-pattern}

Our router already turns each path into a [`URLPattern`][url pattern] when we [register a route][router class].
So far our patterns have been fixed paths like `/books`, plus `/assets/*` for our static files.

A pattern can also *name* part of the path, by starting it with a colon.
Here is a demonstration, with the results in comments.

```JavaScript
const pattern = new URLPattern({ pathname: "/books/:bookId" });

pattern.test("http://localhost:8000/books"); // false
pattern.test("http://localhost:8000/books/"); // false
pattern.test("http://localhost:8000/books/42"); // true
pattern.test("http://localhost:8000/books/anything"); // true
pattern.test("http://localhost:8000/books/42/"); // false
pattern.test("http://localhost:8000/books/42/more"); // false

const match = pattern.exec("http://localhost:8000/books/42");
match.pathname.groups; // { bookId: "42" }

pattern.exec("http://localhost:8000/books/anything").pathname.groups;
// { bookId: "anything" }
```

- [`test`][url pattern test] tells us whether a URL matches, which is what our router already uses.
- The pattern matches paths beginning with `/books/`.
- `:bookId` matches one part of the path, up to the next `/`, whatever it is.
- The pattern doesn't include any `/` characters after `:bookId`.
- [`exec`][url pattern exec] tells us *how* it matched. Its `pathname.groups` has a property for each named part, holding the text that it matched.

Notice that `bookId` is the *string* `"42"`, not a number.
Everything in a URL is text.

### Setting ctx.params {#ctx-params}

Update **server/tools/router.js** so that `dispatch` puts the named parts of the path on `ctx`.

**server/tools/router.js**
```JavaScript{hl_lines="30"}
import { NotFoundError } from "./errors.js";

export class Router {
    constructor() {
        this.routes = [];
        this.middleware = [];
    }

    register(pathname, handler) {
        const pattern = new URLPattern({ pathname });
        this.routes.push({ pattern, handler });
    }

    use(fn) {
        this.middleware.push(fn);
    }

    handle(ctx) {
        return this.chain(ctx, [...this.middleware, (ctx) => this.dispatch(ctx)]);
    }

    dispatch(ctx) {
        const { url } = ctx;
        const route = this.routes.find((route) => {
            return route.pattern.test(url);
        });
        if (!route) {
            throw new NotFoundError(`${url.pathname} doesn't match any known route`);
        }
        ctx.params = route.pattern.exec(url).pathname.groups;
        return route.handler(ctx);
    }

    chain(ctx, handlers) {
        const [fn, ...remaining] = handlers;
        if (!remaining.length) return fn(ctx);
        const next = (ctx) => this.chain(ctx, remaining);
        return fn(ctx, next);
    }
}
```

Once `dispatch` has found the matching route, it runs that route's pattern again with `exec`, and stores the groups as `ctx.params`.
Any handler can now read the named parts of its path from `ctx.params`.
A route with no named parts, like `/books`, gets an empty object.

> The router's placeholder comment has gone.
> It was waiting for URL parameters, and now we have them.

### A stub book page {#book-stub}

We will build the book page in small steps.
The first version is a *stub*: just enough to show that the route works.

Create **server/views/book.js**.

**server/views/book.js**
```JavaScript{hl_lines="1-13"}
import { html } from "../tools/html.js";

export function bookView(ctx) {
    const { params: { bookId } } = ctx;

    return html`
        <section>
            <header>
                <h1>Book ${bookId}</h1>
            </header>
        </section>
    `;
}
```

For now, the view takes the `bookId` straight from `ctx.params`, with the same nested destructuring we use for `ctx.data` in `booksView`.

Add a `bookController` to **server/controllers/books.js**.

**server/controllers/books.js**
```JavaScript{hl_lines="5 25-28"}
import { countPages, searchBooks } from "../models/books.js";
import { render } from "../tools/render.js";
import { NotFoundError } from "../tools/errors.js";
import { booksView } from "../views/books.js";
import { bookView } from "../views/book.js";

const PAGE_SIZE = 16;

export function booksController(ctx) {
    const { url } = ctx;
    const q = url.searchParams.get("q") ?? "";
    const pageParam = url.searchParams.get("page");
    const page = pageParam === null ? 1 : Number(pageParam);
    const pageCount = Math.max(1, countPages(q, PAGE_SIZE));
    if (!Number.isSafeInteger(page) || page < 1 || page > pageCount) {
        throw new NotFoundError(`Page ${pageParam} not found`);
    }
    ctx.data.q = q;
    ctx.data.page = page;
    ctx.data.books = searchBooks(q, page, PAGE_SIZE);
    ctx.data.pageCount = pageCount;
    ctx.data.title = q ? `Books matching "${q}"` : "Books";
    return render(booksView, ctx);
}

export function bookController(ctx) {
    return render(bookView, ctx);
}
```

It does nothing but render the view, for now.

Finally, register the route in **server/main.js**.

**server/main.js**
```JavaScript{hl_lines="2 16"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import { staticController } from "./controllers/static.js";
import { Router } from "./tools/router.js";
import { withLogs } from "./middleware/logging.js";
import { withErrors } from "./middleware/errors.js";

const app = new Router();

app.use(withLogs);
app.use(withErrors);

app.register("/assets/*", staticController);
app.register("/", homeController);
app.register("/books", booksController);
app.register("/books/:bookId", bookController);

export function server(request) {
    const url = new URL(request.url);
    const ctx = { request, url, data: {} };
    return app.handle(ctx);
}
```

The route `/books` doesn't match **/books/1**, because a pattern has to match the whole path.
So the two routes never get in each other's way.

Visit [http://localhost:8000/books][books] and click on **The Hunger Games** again.
This time we get a page of our own, with the heading **Book 1**.
Try [http://localhost:8000/books/42][book 42] too.

Now try [http://localhost:8000/books/hello][book hello].
The page says **Book hello**, because `:bookId` matches anything.
Our stub doesn't know which books exist yet, so the next step is to load the book.

## Loading the book {#get-book-by-id}

We need a model function that finds one book by its `id`.
Add `getBookById` to **server/models/books.js**.

**server/models/books.js**
```JavaScript{hl_lines="11-16"}
import { db } from "./db.js";

export function createBook({ title, author, year, thumbnailUrl, coverUrl }) {
    return db.prepare(`
        INSERT INTO books(title, author, year, thumbnailUrl, coverUrl)
        VALUES (:title, :author, :year, :thumbnailUrl, :coverUrl)
        RETURNING id, title, author, year, thumbnailUrl, coverUrl
    `).get({ title, author, year, thumbnailUrl, coverUrl });
}

export function getBookById(id) {
    return db.prepare(
        `SELECT id, title, author, year, thumbnailUrl, coverUrl FROM books WHERE id=:id`,
    ).get({ id });
}

export function searchBooks(q, page, pageSize) {
    return db.prepare(`
        SELECT id, title, author, year, thumbnailUrl, coverUrl FROM books
        WHERE title LIKE :pattern OR author LIKE :pattern
        ORDER BY title
        LIMIT :limit OFFSET :offset
    `).all({ pattern: `%${q}%`, limit: pageSize, offset: (page - 1) * pageSize });
}

function countBooks(q) {
    return db.prepare(`
        SELECT count(*) AS count FROM books
        WHERE title LIKE :pattern OR author LIKE :pattern
    `).get({ pattern: `%${q}%` }).count;
}

export function countPages(q, pageSize) {
    return Math.ceil(countBooks(q) / pageSize);
}
```

`WHERE id=:id` picks out the one row whose `id` matches, using a [named parameter][named parameters].
This time we call `.get()` rather than `.all()`, because there can only be one book with a given `id`.
`.get()` returns the first matching row as an object, or `undefined` if there isn't one.

Update **server/controllers/books.js** to load the book.

**server/controllers/books.js**
```JavaScript{hl_lines="1 27-33"}
import { countPages, getBookById, searchBooks } from "../models/books.js";
import { render } from "../tools/render.js";
import { NotFoundError } from "../tools/errors.js";
import { booksView } from "../views/books.js";
import { bookView } from "../views/book.js";

const PAGE_SIZE = 16;

export function booksController(ctx) {
    const { url } = ctx;
    const q = url.searchParams.get("q") ?? "";
    const pageParam = url.searchParams.get("page");
    const page = pageParam === null ? 1 : Number(pageParam);
    const pageCount = Math.max(1, countPages(q, PAGE_SIZE));
    if (!Number.isSafeInteger(page) || page < 1 || page > pageCount) {
        throw new NotFoundError(`Page ${pageParam} not found`);
    }
    ctx.data.q = q;
    ctx.data.page = page;
    ctx.data.books = searchBooks(q, page, PAGE_SIZE);
    ctx.data.pageCount = pageCount;
    ctx.data.title = q ? `Books matching "${q}"` : "Books";
    return render(booksView, ctx);
}

export function bookController(ctx) {
    const { params: { bookId } } = ctx;
    const book = getBookById(Number(bookId));
    if (!book) {
        throw new NotFoundError(`Book ${bookId} not found`);
    }
    ctx.data.book = book;
    ctx.data.title = book.title;
    return render(bookView, ctx);
}
```

- `bookId` is a string, so we turn it into a number with [`Number`][number], just as we did for the [page number][page number].
- If there is no such book, we throw a `NotFoundError`, just like the books controller does for a [page that doesn't exist][invalid page]. [`withErrors`][with errors] catches it and shows our not found page.
- Otherwise, we put the book on `ctx.data` for the view, and use its title as the page title, so it shows in the browser tab.

Now update **server/views/book.js** to show the title.

**server/views/book.js**
```JavaScript{hl_lines="4 9"}
import { html } from "../tools/html.js";

export function bookView(ctx) {
    const { data: { book } } = ctx;

    return html`
        <section>
            <header>
                <h1>${book.title}</h1>
            </header>
        </section>
    `;
}
```

The view reads the book from `ctx.data` now, and doesn't need `ctx.params` any more.
The controller deals with the URL, and the view only deals with the data it is given.

Visit [http://localhost:8000/books/1][book 1].
The heading is **The Hunger Games**, and so is the browser tab.

Now visit [http://localhost:8000/books/99999][book 99999] and [http://localhost:8000/books/hello][book hello].
Both get the not found page.
`Number("hello")` is `NaN`, and no book has that `id`.
The terminal shows our message.

```plaintext {linenos=false}

============================
GET /books/99999
[error] Book 99999 not found
RESPONSE 404
============================
```

## The book's details {#book-details}

Update **server/views/book.js** to show more about the book.

**server/views/book.js**
```JavaScript{hl_lines="10 12-13"}
import { html } from "../tools/html.js";

export function bookView(ctx) {
    const { data: { book } } = ctx;

    return html`
        <section>
            <header>
                <h1>${book.title}</h1>
                <p>by ${book.author}</p>
            </header>
            ${book.year ? html`<p>Published ${book.year}</p>` : ""}
            <p><a href="/books">Back to all books</a></p>
        </section>
    `;
}
```

- The author goes in the `<header>`, under the title.
- The year is shown only when there is one, using the [conditional operator][missing year] just like the book card. This time the year gets its own paragraph, so we need the `html` helper for it.
- A link takes us back to all the books.

Visit [http://localhost:8000/books/1][book 1] to see the details.
Then visit [http://localhost:8000/books/267][book 267], the movie companion with no year.
It has no **Published** line at all.

## The cover {#book-cover}

A book page should show the book's cover.
We already have a default cover for books without one, but `DEFAULT_COVER` belongs to **server/views/books.js**.
Rather than write the address out again, we will share it.

Update **server/views/books.js** to export `DEFAULT_COVER`.

**server/views/books.js**
```JavaScript{hl_lines="4"}
import { html } from "../tools/html.js";
import { searchForm } from "./partials/searchForm.js";

export const DEFAULT_COVER = "/assets/default-cover.svg";

export function booksView(ctx) {
    const { data: { title, books, q, page, pageCount } } = ctx;
    const results = books.length
        ? html`<ul class="grid">${books.map(bookCard)}</ul>`
        : html`<p>${q ? `No books match "${q}"` : "No books found"}</p>`;

    return html`
        <section>
            <header>
                <h1>${title}</h1>
                ${q ? html`<p><a href="/books">show all books</a></p>` : ""}
            </header>
            ${searchForm(q)}
            ${results}
            ${pagination(q, page, pageCount)}
        </section>
    `;
}

function bookCard(book) {
    const cover = book.coverUrl ?? DEFAULT_COVER;
    return html`
        <li class="card book">
            <img class="cover" src="${cover}" alt="" width="200" height="300" loading="lazy">
            <div>
                <h2><a href="/books/${book.id}">${book.title}</a></h2>
                <p>${book.author}${book.year ? ` (${book.year})` : ""}</p>
            </div>
        </li>
    `;
}

// Previous, the current page and next. Nothing at all if there's only one page
function pagination(q, page, pageCount) {
    if (pageCount <= 1) return "";
    return html`<nav aria-label="Pagination">
            ${page > 1 ? html`<a href="${pageUrl(q, page - 1)}" rel="prev">previous</a>` : ""}
            <a href="${pageUrl(q, page)}" aria-current="page">page ${page} of ${pageCount}</a>
            ${page < pageCount ? html`<a href="${pageUrl(q, page + 1)}" rel="next">next</a>` : ""}
        </nav>`;
}

// Only keep q in the link if there's a search
function pageUrl(q, page) {
    const params = q ? { q, page } : { page };
    return `/books?${new URLSearchParams(params)}`;
}
```

It wasn't exported before, because nothing else needed it.
Now something does.

Update **server/views/book.js** to import it, and show the cover.

**server/views/book.js**
```JavaScript{hl_lines="2 13-20"}
import { html } from "../tools/html.js";
import { DEFAULT_COVER } from "./books.js";

export function bookView(ctx) {
    const { data: { book } } = ctx;

    return html`
        <section>
            <header>
                <h1>${book.title}</h1>
                <p>by ${book.author}</p>
            </header>
            <div class="row">
                <img class="cover" src="${book.coverUrl ?? DEFAULT_COVER}"
                    alt="Cover of '${book.title}'">
                <div>
                    ${book.year ? html`<p>Published ${book.year}</p>` : ""}
                    <p><a href="/books">Back to all books</a></p>
                </div>
            </div>
        </section>
    `;
}
```

- We use the large `coverUrl`, or the default cover when there isn't one, with `??` just like the book card.
- The cover and the details are wrapped in a `<div class="row">`. We will style the row next.
- The `alt` text names the book. On the books page the cover is decoration, because the title is right next to it, so it has an [empty alt][alt]. Here the picture *is* the content, so a screen reader should say what it is.

Visit [http://localhost:8000/books/1][book 1].
The cover is there, but it fills the whole width of the page, and the details are pushed far below it.

{{<figure src="images/book-cover-unstyled.webp" alt="The page for The Hunger Games, where the cover image is stretched across the full width of the page and the details are out of sight below it" caption="Without a style, the cover fills the page.">}}

## A row {#row}

We want the cover and the details side by side.
Add a `.row` rule to **public/styles.css**, after the `.card.book` rule.

**public/styles.css**
```css {linenostart="141"}
/* A row puts its items side by side, wrapping when there isn't room. A picture in it is 15rem wide */
.row {
    display: flex;
    flex-wrap: wrap;
    align-items: start;
    gap: 1rem;
    > img {
        inline-size: 15rem;
    }
}

```

- `display: flex` lays the row's children out side by side, just like the links in our `nav`.
- `flex-wrap: wrap` lets the details move below the cover when there isn't room beside it.
- `align-items: start` keeps each item at the top of the row, rather than stretching it to the height of the tallest one.
- `gap` puts some space between them.
- `> img` is a nested rule for an `<img>` that is a direct child of the row. It gives the picture a fixed width of `15rem`.

The rule doesn't mention books at all.
Anything in a `.row` sits side by side, so we can use it again.

Visit [http://localhost:8000/books/1][book 1] again.

{{<figure src="images/book-page.webp" alt="The page for The Hunger Games, with the title and author at the top, then the cover at a sensible size with Published 2008 and a Back to all books link beside it" caption="The book page, with its cover beside its details.">}}

Make the browser window narrow, and the details move below the cover.
Visit [http://localhost:8000/books/36][book 36] too: **Memoirs of a Geisha** has no cover, so it gets the default one.

## Conclusion

Every book now has a page of its own, at **/books/** followed by its `id`, and every card on the books page links to one.
A route pattern can name part of the path, and our router puts the named parts on `ctx.params`.
`bookController` loads the book with `getBookById`, and throws a `NotFoundError` when there isn't one.

We built the page in steps: a stub that proved the route worked, then the data, then the markup, and finally the styles.
We will build every new page this way.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="32"}
my-app/
├── .vscode/
│   └── settings.json
├── db/
│   ├── app.db
│   ├── BOOKS-ATTRIBUTION.md
│   └── books.json
├── public/
│   ├── default-cover.svg
│   ├── icon.svg
│   └── styles.css
├── server/
│   ├── controllers/
│   │   ├── books.js
│   │   ├── errors.js
│   │   ├── home.js
│   │   └── static.js
│   ├── middleware/
│   │   ├── errors.js
│   │   └── logging.js
│   ├── models/
│   │   ├── books.js
│   │   └── db.js
│   ├── tools/
│   │   ├── errors.js
│   │   ├── html.js
│   │   ├── render.js
│   │   └── router.js
│   ├── views/
│   │   ├── partials/
│   │   │   └── searchForm.js
│   │   ├── book.js
│   │   ├── books.js
│   │   ├── error.js
│   │   ├── home.js
│   │   └── layout.js
│   └── main.js
├── tasks/
│   ├── db-create.js
│   ├── db-seed.js
│   └── server.js
├── .gitignore
├── deno.json
└── deno.lock
```

In the next exercise we will add shelves, starting with a table in the database and a page that lists them.

## Challenge: Explore the book pages

These are questions to investigate, not changes to keep.

- Visit [http://localhost:8000/books/1000][book 1000] and [http://localhost:8000/books/1001][book 1001]. Why is one of them not found?
- Visit [http://localhost:8000/books/01][book 01], [http://localhost:8000/books/1.0][book 1.0] and [http://localhost:8000/books/0x1][book 0x1]. Which book do you get, and why? Compare this with the [page number challenge][page challenge].
- Visit [http://localhost:8000/books/1/][book 1 slash], with a slash on the end. What happens, and why?
- Add `console.log(ctx.params);` to `staticController`, and refresh any page. What is `ctx.params` for **/assets/styles.css**?

<details>
<summary>What the router does with these</summary>

There are 1,000 books, with ids from 1 to 1,000, so `getBookById(1001)` finds nothing and returns `undefined`.

`Number` is forgiving.
`Number("01")`, `Number("1.0")` and `Number("0x1")` are all `1`, so all three addresses show **The Hunger Games**.
The same idea as the stricter page number would turn them away.

`/books/:bookId` doesn't match **/books/1/**, because the forward slash is not in the pattern, so the router finds no matching route and throws its own `NotFoundError`.

The `*` in `/assets/*` is a group too, but it has no name, so it is numbered instead.
For **/assets/styles.css**, `ctx.params` is `{ "0": "styles.css" }`.
Patterns with additional wildcards would add numbered entries in order of appearance. 

</details>

> If you tried any changes, undo them, so that your code matches the starting point of the next exercise.

[previous unit]: user-input-and-security/
[books]: http://localhost:8000/books
[book 1]: http://localhost:8000/books/1
[book 42]: http://localhost:8000/books/42
[book hello]: http://localhost:8000/books/hello
[book 99999]: http://localhost:8000/books/99999
[book 267]: http://localhost:8000/books/267
[book 36]: http://localhost:8000/books/36
[book 1000]: http://localhost:8000/books/1000
[book 1001]: http://localhost:8000/books/1001
[book 01]: http://localhost:8000/books/01
[book 1.0]: http://localhost:8000/books/1.0
[book 0x1]: http://localhost:8000/books/0x1
[book 1 slash]: http://localhost:8000/books/1/
[not found error]: router-class/router/#not-found-error
[router class]: router-class/router/#class
[url pattern]: https://developer.mozilla.org/en-US/docs/Web/API/URLPattern
[url pattern test]: https://developer.mozilla.org/en-US/docs/Web/API/URLPattern/test
[url pattern exec]: https://developer.mozilla.org/en-US/docs/Web/API/URLPattern/exec
[named parameters]: user-input-and-security/search/#like
[number]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/Number
[page number]: user-input-and-security/pagination/#page-number
[invalid page]: user-input-and-security/pagination/#invalid-page
[missing year]: user-input-and-security/pagination/#missing-year
[alt]: sqlite-and-models/books-page/#cover
[page challenge]: user-input-and-security/pagination/#challenge-test-the-page-number
[with errors]: router-class/middleware/#with-errors

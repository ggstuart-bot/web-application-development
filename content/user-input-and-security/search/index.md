---
type: exercise
title: Search for books
weight: 3
author: Dr Graeme Stuart
---

In this exercise we will add a search form, so our users can find books by title or author.
It is the first time our application accepts input that our users type.

<!--more-->

> You should have a books page that shows one page at a time, from the [previous exercise][previous exercise].
> Your database should hold all 1,000 books. If you have been experimenting, run `deno task db:reset` to get them back.
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

The search is a new *feature*, and our first *user input*.
We will build it one small step at a time, checking the browser after each one.
By the end of this exercise you should be able to:

- Explain how a form builds a *query string*
- Write an HTML search form with a label, an input and a button
- Filter rows in SQL with `LIKE`, and explain why we pass the user's search as a parameter
- Build a query string with `URLSearchParams`, so that our page links keep the search
- Move a piece of a page that two views share into a *partial*
- Lay out a form with a CSS grid

## A search form {#form}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

We already read the page number from the [query string][query string].
A search can go in the query string too, but we want our users to type it, not edit the URL, so let's give them a form.

Update **server/views/books.js** to add a search form to the books page.

**server/views/books.js**
```JavaScript{hl_lines="11 43-53"}
const DEFAULT_COVER = "/assets/default-cover.svg";

export function booksView(ctx) {
    const { data: { books, page, pageCount } } = ctx;

    return `
        <section>
            <header>
                <h1>Books</h1>
            </header>
            ${searchForm()}
            <ul class="grid">
                ${books.map(bookCard).join("")}
            </ul>
            ${pagination(page, pageCount)}
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
                <p>${book.author}${book.year ? ` (${book.year})` : ""}</p>
            </div>
        </li>
    `;
}

// Previous, the current page and next. Nothing at all if there's only one page
function pagination(page, pageCount) {
    if (pageCount <= 1) return "";
    return `<nav aria-label="Pagination">
            ${page > 1 ? `<a href="/books?page=${page - 1}" rel="prev">previous</a>` : ""}
            <a href="/books?page=${page}" aria-current="page">page ${page} of ${pageCount}</a>
            ${page < pageCount ? `<a href="/books?page=${page + 1}" rel="next">next</a>` : ""}
        </nav>`;
}

function searchForm() {
    return `
        <search>
            <form method="GET" action="/books">
                <label for="q">Search by title or author</label>
                <input id="q" type="search" name="q">
                <button>search</button>
            </form>
        </search>
    `;
}
```

`searchForm` is a small function in the same module, just like `pagination`.
Only the books page uses it, so it stays in **books.js** for now.
We call it inside the template with `${searchForm()}`.

This is our first form, so let's look at each part.

- [`<search>`][search element] tells browsers, and assistive technology such as screen readers, that this part of the page is for searching.
- A [`<form>`][form] collects what the user types. When the form is submitted, the browser sends it to the URL in its `action` attribute, here `/books`.
- `method="GET"` asks the browser to put the form's values into the query string of that URL, and visit it, just as if we had followed a link.
- A [`<label>`][label] names an input. Its `for` attribute matches the input's `id`, so a screen reader reads the label out, and clicking the label puts the cursor in the box.
- The [`<input>`][input search] is where the user types. `type="search"` is a text box meant for searches, and some browsers add a button to clear it.
- The input's [`name`][input name] becomes the name of the parameter, so a search for **hobbit** becomes `q=hobbit`.
- A [`<button>`][button] inside a form submits the form. Pressing **Enter** in the box does the same.

Refresh [http://localhost:8000/books][books], with the developer tools (**F12**) open on the **network** panel.
Type **hobbit** into the box and press **search**.

The address bar now shows `http://localhost:8000/books?q=hobbit`, and so does the request in the **network** panel.
The browser built the query string for us, from the input's `name` and what we typed.

Now search for **harry potter**.
The URL is `/books?q=harry+potter`.
A space isn't allowed in a URL, so the browser *encodes* it as `+`.
Other characters are encoded too, such as `"` which becomes `%22`.

The page still shows the first 16 of all our books, because our server doesn't read `q` yet.

## Searching the books {#search}

To search, our model needs a new query, and the books controller needs to read the search from the URL.
These two files depend on each other, so update both before you refresh.

### A search query {#like}

Update **server/models/books.js**, replacing `getBooks` with `searchBooks`.

**server/models/books.js**
```JavaScript{hl_lines="11 14 17 20 23-24 27-28"}
import { db } from "./db.js";

export function createBook({ title, author, year, thumbnailUrl, coverUrl }) {
    return db.prepare(`
        INSERT INTO books(title, author, year, thumbnailUrl, coverUrl)
        VALUES (:title, :author, :year, :thumbnailUrl, :coverUrl)
        RETURNING id, title, author, year, thumbnailUrl, coverUrl
    `).get({ title, author, year, thumbnailUrl, coverUrl });
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

> The placeholder has gone, because this is the search it was waiting for.

The `WHERE` clause keeps only the rows we want.

- [`LIKE`][sqlite like] compares text with a *pattern*. In a pattern, `%` matches any run of characters, including none at all.
- So the pattern `%hobbit%` means "contains **hobbit** anywhere". We build it with a [template literal][template literals], `` `%${q}%` ``.
- `LIKE` ignores the difference between capital and small letters, so we don't need the [`toLowerCase`][lower case] we used in the book search.
- `OR` keeps a row if *either* test is true, like the [`||`][logical or] in the book search, so we search both the title and the author.

We use the same *named parameter*, `:pattern`, twice, and pass its value alongside `:limit` and `:offset`.

`countBooks` uses the same `WHERE` clause, so it counts only the books that match the search, and `countPages` passes the search on to it.
So a search has its own number of pages.

### Why a parameter matters {#parameters}

This is the first time our SQL includes something a user typed.
We could have pasted `q` straight into the SQL with a template literal, but we must never do that.

A parameter is sent to SQLite *separately* from the SQL.
SQLite treats whatever is in the parameter as a value to compare, never as part of the query.
So, whatever our users type, they can only ever change what we search *for*, not *how* we search.

> Pasting user input into SQL is how [SQL injection][sql injection] attacks happen.
> Parameters are the standard defence. See the [OWASP advice on query parameterisation][owasp params] if you'd like to know more.

### Reading the search {#search-params}

Now update **server/controllers/books.js**.

**server/controllers/books.js**
```JavaScript{hl_lines="1 10 13 17 19"}
import { countPages, searchBooks } from "../models/books.js";
import { render } from "../tools/render.js";
import { NotFoundError } from "../tools/errors.js";
import { booksView } from "../views/books.js";

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
    ctx.data.title = "Books";
    return render(booksView, ctx);
}
```

We read `q` just as we [read the page number][page number], with `url.searchParams.get`.
The value comes back already decoded, so `harry+potter` becomes `"harry potter"`.
If there is no `q` parameter at all, `get` gives us `null`, and we use [`??`][nullish coalescing] to fall back to an empty string, just as we did with `prompt` in the book search.

We pass `q` to `searchBooks` and `countPages`, and also put it on `ctx.data`, ready for the view.

Save both files, then try some searches.

- [http://localhost:8000/books?q=hobbit][q hobbit] shows three books, including **The Hobbit**.
- [http://localhost:8000/books?q=tolkien][q tolkien] shows eight, because it matches the author too.
- [http://localhost:8000/books?q=harry+potter][q harry potter] shows nine.
- [http://localhost:8000/books][books] still shows **page 1 of 63**. With no `q`, the pattern is `%%`, which matches everything.

Use the form too.
It works, but after each search the box is empty again.

### Losing the search {#lost-search}

Now visit [http://localhost:8000/books?q=the][q the].
It says **page 1 of 29**, because only the matching books are counted.

Click **next**.
The address bar says `/books?page=2`, and we are on **page 2 of 63**.
Our page links only include the page, so following one loses the search.
We will fix the box first, then the links.

## Keeping the search in the box {#value}

Update **server/views/books.js** to pass the search to the form.

**server/views/books.js**
```JavaScript{hl_lines="4 11 43 48"}
const DEFAULT_COVER = "/assets/default-cover.svg";

export function booksView(ctx) {
    const { data: { books, q, page, pageCount } } = ctx;

    return `
        <section>
            <header>
                <h1>Books</h1>
            </header>
            ${searchForm(q)}
            <ul class="grid">
                ${books.map(bookCard).join("")}
            </ul>
            ${pagination(page, pageCount)}
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
                <p>${book.author}${book.year ? ` (${book.year})` : ""}</p>
            </div>
        </li>
    `;
}

// Previous, the current page and next. Nothing at all if there's only one page
function pagination(page, pageCount) {
    if (pageCount <= 1) return "";
    return `<nav aria-label="Pagination">
            ${page > 1 ? `<a href="/books?page=${page - 1}" rel="prev">previous</a>` : ""}
            <a href="/books?page=${page}" aria-current="page">page ${page} of ${pageCount}</a>
            ${page < pageCount ? `<a href="/books?page=${page + 1}" rel="next">next</a>` : ""}
        </nav>`;
}

function searchForm(q) {
    return `
        <search>
            <form method="GET" action="/books">
                <label for="q">Search by title or author</label>
                <input id="q" type="search" name="q" value="${q}">
                <button>search</button>
            </form>
        </search>
    `;
}
```

The view takes `q` out of `ctx.data` alongside the books, and passes it to `searchForm`.
The input's [`value`][input value] attribute sets what is in the box when the page loads.

Search for **tolkien** again.
This time the box still says **tolkien** when the results arrive, so it is easy to change the search.

> This is the first time something a user typed has gone back into our HTML.
> Keep that in mind, because we will come back to it at the end of this exercise.

## The search in the title {#title}

Our page should say what it is showing.

Update **server/controllers/books.js** to put the search in the title.

**server/controllers/books.js**
```JavaScript{hl_lines="21"}
import { countPages, searchBooks } from "../models/books.js";
import { render } from "../tools/render.js";
import { NotFoundError } from "../tools/errors.js";
import { booksView } from "../views/books.js";

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
```

This is the [conditional operator][conditional operator] again.
`q` is a string, and an empty string is [falsy][falsy], just like a missing year.
So when `q` is `""`, the title is just **Books**, and when the user has searched, it is **Books matching "tolkien"**.

Now update **server/views/books.js** to show the title, with a link back to all the books.

**server/views/books.js**
```JavaScript{hl_lines="4 9-10"}
const DEFAULT_COVER = "/assets/default-cover.svg";

export function booksView(ctx) {
    const { data: { title, books, q, page, pageCount } } = ctx;

    return `
        <section>
            <header>
                <h1>${title}</h1>
                ${q ? `<p><a href="/books">show all books</a></p>` : ""}
            </header>
            ${searchForm(q)}
            <ul class="grid">
                ${books.map(bookCard).join("")}
            </ul>
            ${pagination(page, pageCount)}
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
                <p>${book.author}${book.year ? ` (${book.year})` : ""}</p>
            </div>
        </li>
    `;
}

// Previous, the current page and next. Nothing at all if there's only one page
function pagination(page, pageCount) {
    if (pageCount <= 1) return "";
    return `<nav aria-label="Pagination">
            ${page > 1 ? `<a href="/books?page=${page - 1}" rel="prev">previous</a>` : ""}
            <a href="/books?page=${page}" aria-current="page">page ${page} of ${pageCount}</a>
            ${page < pageCount ? `<a href="/books?page=${page + 1}" rel="next">next</a>` : ""}
        </nav>`;
}

function searchForm(q) {
    return `
        <search>
            <form method="GET" action="/books">
                <label for="q">Search by title or author</label>
                <input id="q" type="search" name="q" value="${q}">
                <button>search</button>
            </form>
        </search>
    `;
}
```

The `<h1>` now shows the title from `ctx.data`, so the heading and the browser tab always agree.

The line under it uses the conditional operator inside the template, just like our [page links][page links].
When there is a search, it gives a paragraph with a link to [http://localhost:8000/books][books]. Otherwise, it gives an empty string, which adds nothing to the page.

Visit [http://localhost:8000/books?q=tolkien][q tolkien].
The heading and the tab both say **Books matching "tolkien"**, and **show all books** takes us back to every book.

## No matches {#no-results}

Search for something that isn't there, such as [http://localhost:8000/books?q=zebra][q zebra].
The heading says **Books matching "zebra"**, then there is just an empty space.
We should say that nothing matched.

Update **server/views/books.js**.

**server/views/books.js**
```JavaScript{hl_lines="5-7 16"}
const DEFAULT_COVER = "/assets/default-cover.svg";

export function booksView(ctx) {
    const { data: { title, books, q, page, pageCount } } = ctx;
    const results = books.length
        ? `<ul class="grid">${books.map(bookCard).join("")}</ul>`
        : `<p>${q ? `No books match "${q}"` : "No books found"}</p>`;

    return `
        <section>
            <header>
                <h1>${title}</h1>
                ${q ? `<p><a href="/books">show all books</a></p>` : ""}
            </header>
            ${searchForm(q)}
            ${results}
            ${pagination(page, pageCount)}
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
                <p>${book.author}${book.year ? ` (${book.year})` : ""}</p>
            </div>
        </li>
    `;
}

// Previous, the current page and next. Nothing at all if there's only one page
function pagination(page, pageCount) {
    if (pageCount <= 1) return "";
    return `<nav aria-label="Pagination">
            ${page > 1 ? `<a href="/books?page=${page - 1}" rel="prev">previous</a>` : ""}
            <a href="/books?page=${page}" aria-current="page">page ${page} of ${pageCount}</a>
            ${page < pageCount ? `<a href="/books?page=${page + 1}" rel="next">next</a>` : ""}
        </nav>`;
}

function searchForm(q) {
    return `
        <search>
            <form method="GET" action="/books">
                <label for="q">Search by title or author</label>
                <input id="q" type="search" name="q" value="${q}">
                <button>search</button>
            </form>
        </search>
    `;
}
```

Before the template, we work out the `results` with another conditional operator, split over three lines so each part is easy to read.

- `books.length` is the number of books we found. `0` is falsy, so with no books we take the second branch.
- With books, `results` is the grid of cards, as before.
- With no books, `results` is a paragraph. Inside it, one more conditional chooses the message: **No books match "zebra"** for a search, or **No books found** if there is no search and the database is empty.

The template then only needs `${results}`.

Refresh [http://localhost:8000/books?q=zebra][q zebra].

```html
<p>No books match "zebra"</p>
```

## Keeping the search in the page links {#page-url}

Our page links still [lose the search][lost search].
Each link needs the search *and* the page in its query string.
Rather than build that three times, let's write one function for it.

Update **server/views/books.js**.

**server/views/books.js**
```JavaScript{hl_lines="17 36 39-41 45-49"}
const DEFAULT_COVER = "/assets/default-cover.svg";

export function booksView(ctx) {
    const { data: { title, books, q, page, pageCount } } = ctx;
    const results = books.length
        ? `<ul class="grid">${books.map(bookCard).join("")}</ul>`
        : `<p>${q ? `No books match "${q}"` : "No books found"}</p>`;

    return `
        <section>
            <header>
                <h1>${title}</h1>
                ${q ? `<p><a href="/books">show all books</a></p>` : ""}
            </header>
            ${searchForm(q)}
            ${results}
            ${pagination(q, page, pageCount)}
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
                <p>${book.author}${book.year ? ` (${book.year})` : ""}</p>
            </div>
        </li>
    `;
}

// Previous, the current page and next. Nothing at all if there's only one page
function pagination(q, page, pageCount) {
    if (pageCount <= 1) return "";
    return `<nav aria-label="Pagination">
            ${page > 1 ? `<a href="${pageUrl(q, page - 1)}" rel="prev">previous</a>` : ""}
            <a href="${pageUrl(q, page)}" aria-current="page">page ${page} of ${pageCount}</a>
            ${page < pageCount ? `<a href="${pageUrl(q, page + 1)}" rel="next">next</a>` : ""}
        </nav>`;
}

// Only keep q in the link if there's a search
function pageUrl(q, page) {
    const params = q ? { q, page } : { page };
    return `/books?${new URLSearchParams(params)}`;
}

function searchForm(q) {
    return `
        <search>
            <form method="GET" action="/books">
                <label for="q">Search by title or author</label>
                <input id="q" type="search" name="q" value="${q}">
                <button>search</button>
            </form>
        </search>
    `;
}
```

`pagination` now takes the search, `q`, and gives it to `pageUrl` with each page number.

[`URLSearchParams`][url search params] builds a query string from an object.
It gives us the same kind of query string that our search form builds, with each value encoded.

```JavaScript {linenos=false}
`${new URLSearchParams({ q: "of the", page: 2 })}`
```

```plaintext {linenos=false}
q=of+the&page=2
```

- `{ q, page }` uses [shorthand properties][shorthand], so it means `{ q: q, page: page }`.
- With no search, `q` is `""`, which is falsy, so we leave it out and the link is just `/books?page=2`, as before.
- Because the values are encoded, a search with spaces, `&` or quotes in it stays one value, and can't add parameters of its own.

Visit [http://localhost:8000/books?q=of+the][q of the] and click **next**.
The address bar says `/books?q=of+the&page=2`, and we are on **page 2 of 3** of the search.

{{<figure src="images/search-page-2.webp" alt="The second page of results for the search of the, with the search in the address bar, the heading Books matching “of the” and the search box" caption="The search is kept on page 2.">}}

{{<figure src="images/search-page-2-links.webp" alt="The bottom of the second page of search results, with previous, page 2 of 3 and next links, and the search still in the address bar" caption="The page links keep the search too.">}}

View the page source and find the links.

```html {linenos=false}
            <nav aria-label="Pagination">
            <a href="/books?q=of+the&page=1" rel="prev">previous</a>
            <a href="/books?q=of+the&page=2" aria-current="page">page 2 of 3</a>
            <a href="/books?q=of+the&page=3" rel="next">next</a>
        </nav>
```

[http://localhost:8000/books][books], with no search, still links to `/books?page=2`.
A search has its own number of pages, so [http://localhost:8000/books?q=of+the&page=3][q of the page 3] works, and [http://localhost:8000/books?q=of+the&page=4][q of the page 4] is not found.

## One form, two pages {#partial}

Our home page would be a good place to search from, too.
Two views need the same form, so it should be a *partial*: a piece of a page that more than one view uses.

Create a **partials** folder in the **server/views** folder, and create **searchForm.js** in it.
Move `searchForm` into it.

**server/views/partials/searchForm.js**
```JavaScript
// The book search form, shared by the home page and the books page.
// Pass the current search to pre-fill the box
export function searchForm(q = "") {
    return `
        <search>
            <form method="GET" action="/books">
                <label for="q">Search by title or author</label>
                <input id="q" type="search" name="q" value="${q}">
                <button>search</button>
            </form>
        </search>
    `;
}
```

It is now exported, so other views can import it.
`q = ""` is a [default value][default parameters], so we can call `searchForm()` with nothing, and the box starts empty.

Update **server/views/books.js** to import it, and remove the old `searchForm` from the bottom of the file.

**server/views/books.js**
```JavaScript{hl_lines="1"}
import { searchForm } from "./partials/searchForm.js";

const DEFAULT_COVER = "/assets/default-cover.svg";

export function booksView(ctx) {
    const { data: { title, books, q, page, pageCount } } = ctx;
    const results = books.length
        ? `<ul class="grid">${books.map(bookCard).join("")}</ul>`
        : `<p>${q ? `No books match "${q}"` : "No books found"}</p>`;

    return `
        <section>
            <header>
                <h1>${title}</h1>
                ${q ? `<p><a href="/books">show all books</a></p>` : ""}
            </header>
            ${searchForm(q)}
            ${results}
            ${pagination(q, page, pageCount)}
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
                <p>${book.author}${book.year ? ` (${book.year})` : ""}</p>
            </div>
        </li>
    `;
}

// Previous, the current page and next. Nothing at all if there's only one page
function pagination(q, page, pageCount) {
    if (pageCount <= 1) return "";
    return `<nav aria-label="Pagination">
            ${page > 1 ? `<a href="${pageUrl(q, page - 1)}" rel="prev">previous</a>` : ""}
            <a href="${pageUrl(q, page)}" aria-current="page">page ${page} of ${pageCount}</a>
            ${page < pageCount ? `<a href="${pageUrl(q, page + 1)}" rel="next">next</a>` : ""}
        </nav>`;
}

// Only keep q in the link if there's a search
function pageUrl(q, page) {
    const params = q ? { q, page } : { page };
    return `/books?${new URLSearchParams(params)}`;
}
```

Then update **server/views/home.js** to add the form to the home page.

**server/views/home.js**
```JavaScript{hl_lines="1 10-11"}
import { searchForm } from "./partials/searchForm.js";

export function homeView() {
    return `
        <section>
            <header>
                <h1>Welcome to Bookshelves</h1>
                <p>Organise your books with bookshelves.</p>
            </header>
            <h2>Find a book</h2>
            ${searchForm()}
        </section>
    `;
}
```

Visit [http://localhost:8000/][localhost] and search for **orwell**.
The form's `action` sends us to the books page, `/books?q=orwell`, with George Orwell's three books.

{{<figure src="images/search-home.webp" alt="The Bookshelves home page, with the heading Welcome to Bookshelves, a Find a book heading, and an empty search form labelled Search by title or author" caption="The home page now has the same search form.">}}

## Styling the form {#styles}

Finally, let's make the form match the rest of our site.
We will update **public/styles.css** three times.

### Buttons {#buttons}

The search button still has the browser's grey look.
Give buttons the same look as our nav links, by adding `button` to the selector of the `nav a` rule.

**public/styles.css**
```css {linenostart="141" hl_lines="1-2"}
/* Buttons and nav links share one look. The current page is filled in */
button,
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

The rule already has `background: none`, which clears the browser's grey, and `color: inherit`, which gives the button our page's text colour.
So the button becomes an outlined pill, with the accent underline on hover, just like the nav links.
The rule's comment now says it covers both.

### A grid for forms {#form-grid}

Browsers give inputs and buttons their own smaller font, and an input has a default width that can push it out of a narrow container.
Add these rules at the end of the file.

**public/styles.css**
```css {linenostart="156"}
/* min-width: 0 lets an input shrink below its default width to fit its container */
input {
    min-width: 0;
}

/* Buttons and inputs use the page font rather than the browser's */
button,
input {
    font: inherit;
}

/* A form is a one-column grid: each label, input and button on its own row */
form {
    display: grid;
    gap: 0.5rem;
    max-inline-size: 30rem;
    /* A button is only as wide as its text */
    button {
        justify-self: start;
    }
}
```

- `min-width: 0` lets the box shrink to fit its container, rather than pushing out of it on a narrow screen.
- `font: inherit` makes inputs and buttons use our page font.
- Each `form` is a [grid][grid] with one column, so each label, input and button sits on its own row, with a gap between them, and no wider than `30rem`.
- The nested rule stops the button stretching across the row, so it is only as wide as its text.

### One row for the search {#search-row}

A search form is short, so its box and button can share a row.
Add one more rule at the end of the file.

**public/styles.css**
```css {linenostart="178"}
/* The search box and its button share a row, under the label */
search > form {
    grid-template-columns: 1fr auto;
    /* The label spans both columns */
    label {
        grid-column: 1 / -1;
    }
}
```

- `search > form` is a [child combinator][child combinator]: it selects a form directly inside a `<search>` element, so other forms keep the one-column layout.
- `1fr auto` gives the grid two columns: `1fr` for the box, which takes the space that is left, and `auto` for the button, which is as wide as its text.
- The label spans both columns with [`grid-column: 1 / -1`][grid-column], from the first grid line to the last, so it stays above them.

Refresh [http://localhost:8000/books?q=tolkien][q tolkien] to see the finished search.
In the picture the books are in three columns, with **The Fellowship of the Ring** first, and only the first row fully visible.

{{<figure src="images/search-results.webp" alt="The books page after a search for tolkien, with the heading Books matching “tolkien”, a show all books link, a styled form that still contains the word tolkien, and a three-column grid of matching book cards, only the first row fully visible, starting with The Fellowship of the Ring" caption="A search for tolkien shows the matching books, with the query kept in the form.">}}

## Something to fix {#quotes}

Our search works, but there is a problem.

Search for **"hobbit"**, *with* the double quotes, or visit [http://localhost:8000/books?q=%22hobbit%22][q quoted].
Nothing matches, which is fair, since no title contains quotes.
But look at the search box: it is empty, even though we typed something into it.

View the page source (**Ctrl+U**) and find the input.

```html
<input id="q" type="search" name="q" value=""hobbit"">
```

The user's first `"` closed our `value` attribute early.
What they typed has become part of our HTML, not just text on the page.
A quote is harmless, but a user could type any HTML at all.

> Anything a user types is *untrusted*.
> In the next exercise we will see why this matters, and *escape* everything we write into our pages.

## Conclusion

We added a search to our application.
A form builds a *query string*, the controller reads it with `url.searchParams`, and the model filters the books with `LIKE`.
The search goes to SQLite as a parameter, so it can never change our SQL.
The page keeps the search in the box, in its title and in its page links, and says when nothing matches.
The form is a partial, shared by the home page and the books page, and our styles lay it out as a grid.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="30"}
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
│   │   ├── render.js
│   │   └── router.js
│   ├── views/
│   │   ├── partials/
│   │   │   └── searchForm.js
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

## Challenge: Explore the search

These are questions to investigate, not changes to keep.

- Search for **TOLKIEN** in capitals. Why does it still match?
- Visit [http://localhost:8000/books?q=tolkien&q=rowling][q two]. Which search does the page show, and what does that tell you about `get`?
- Search for `%`, and then for `_`. Why do both show every book? Look up what `_` means in a [`LIKE` pattern][sqlite like].
- Search for three spaces. What does the heading say, and why does nothing match? How could the string method [`trim`][trim] help?
- Search for **stephen king** and click **next**. What does the address bar say, and where did the `+` come from this time?

> If you tried any changes, undo them, so that your code matches the starting point of the next exercise.

[previous exercise]: user-input-and-security/pagination/
[query string]: user-input-and-security/pagination/#query-string
[page number]: user-input-and-security/pagination/#page-number
[page links]: user-input-and-security/pagination/#page-links
[falsy]: user-input-and-security/pagination/#missing-year
[lost search]: user-input-and-security/search/#lost-search
[q hobbit]: http://localhost:8000/books?q=hobbit
[q tolkien]: http://localhost:8000/books?q=tolkien
[q harry potter]: http://localhost:8000/books?q=harry+potter
[q zebra]: http://localhost:8000/books?q=zebra
[q the]: http://localhost:8000/books?q=the
[q of the]: http://localhost:8000/books?q=of+the
[q of the page 3]: http://localhost:8000/books?q=of+the&page=3
[q of the page 4]: http://localhost:8000/books?q=of+the&page=4
[q quoted]: http://localhost:8000/books?q=%22hobbit%22
[q two]: http://localhost:8000/books?q=tolkien&q=rowling
[books]: http://localhost:8000/books
[localhost]: http://localhost:8000/
[search element]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/search
[form]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/form
[label]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/label
[input search]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/input/search
[input name]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/input#name
[input value]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/input#value
[button]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/button
[sqlite like]: https://sqlite.org/lang_expr.html#like
[template literals]: javascript-fundamentals/books-data/#template-literals
[lower case]: javascript-fundamentals/book-search/#lower-case
[logical or]: javascript-fundamentals/book-search/#logical-or
[sql injection]: https://en.wikipedia.org/wiki/SQL_injection
[owasp params]: https://cheatsheetseries.owasp.org/cheatsheets/Query_Parameterization_Cheat_Sheet.html
[nullish coalescing]: javascript-fundamentals/book-search/#prompt
[conditional operator]: views-and-controllers/controllers/#title
[url search params]: https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams
[shorthand]: router-class/router/#classes
[default parameters]: views-and-controllers/controllers/#options
[grid]: sqlite-and-models/books-page/#grid
[child combinator]: https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Selectors/Child_combinator
[grid-column]: https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/grid-column
[trim]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/trim

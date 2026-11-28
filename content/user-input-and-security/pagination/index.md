---
type: exercise
title: Page through the books
weight: 2
author: Dr Graeme Stuart
---

In this exercise we will show our 1,000 books a page at a time, with links to the previous and next pages.
It is the first time our application reads anything from the URL other than the path.

<!--more-->

> You should have the `withErrors` middleware and the error pages from the [previous exercise][previous exercise].
> Your database should hold all 1,000 books. If you have been experimenting, run `deno task db:reset` to get them back.
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

Our books page only ever shows the first 16 books.
In this exercise we will let our users reach the other 984.
By the end of this exercise you should be able to:

- Explain what a *query string* is, and read a value from it with `url.searchParams`
- Turn a value from the query string into a number, and check that it is a whole number
- Fetch one page of rows from the database with `LIMIT` and `OFFSET`, in a stable order
- Count rows with `count(*)`, and work out how many pages there are
- Use *falsy* values, such as `null` or an empty string, in a conditional
- Throw a `NotFoundError` from a controller when a page doesn't exist

## The query string {#query-string}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Now visit [http://localhost:8000/books?page=2][page 2].

The part of the URL after the `?` is the [*query string*][url search].
It holds *parameters*, written as `name=value`, so this one has a parameter called `page` with the value `2`.
Several parameters are joined with `&`, as in `?page=2&lang=en`.

Our page hasn't changed: it still shows the first 16 books, from **'Salem's Lot** to **A Court of Mist and Fury**.
The router matches on the path, `/books`, and nothing reads the query string yet.
Our `withLogs` middleware [logs `url.pathname`][with logs], which doesn't include the query string, so the terminal only says `GET /books`.

## One page at a time {#limit-offset}

Instead of the first 16 books, we will show whichever 16 the query string asks for.
The controller and the model depend on each other here, so update both before you refresh.

### Which page? {#page-number}

Update **server/controllers/books.js** to work out which page was asked for.

**server/controllers/books.js**
```JavaScript{hl_lines="8-12"}
import { getBooks } from "../models/books.js";
import { render } from "../tools/render.js";
import { booksView } from "../views/books.js";

const PAGE_SIZE = 16;

export function booksController(ctx) {
    const { url } = ctx;
    const pageParam = url.searchParams.get("page");
    let page = pageParam === null ? 1 : Number(pageParam);
    if (!Number.isSafeInteger(page) || page < 1) page = 1;
    ctx.data.books = getBooks(page, PAGE_SIZE);
    ctx.data.title = "Books";
    return render(booksView, ctx);
}
```

The router already put the `url` on `ctx` [for its own use][router url], so we can take it out of `ctx` here too, with no other changes.

[`url.searchParams`][searchparams] reads the query string for us.

- [`get("page")`][usp get] gives us the value of the `page` parameter as a string, so `?page=2` gives us `"2"`.
- If there is no `page` parameter at all, `get` gives us `null`, and we use page 1.
- Otherwise, [`Number`][number] turns the string into a number.

Anyone can type anything into the query string, so we can't trust it.
`Number` turns `""` into `0`, `"two"` into `NaN` and `"1.5"` into `1.5`.
[`Number.isSafeInteger`][is safe integer] is only true for a whole number that JavaScript can store exactly, so `NaN`, `1.5` and even `1e30` fail it, and `page < 1` catches zero and negative numbers.
For now, in all of those cases we use page 1 instead.

> `page` is declared with [`let`][let] rather than `const`, because we might change it on the next line.

Finally, we pass `page` and [`PAGE_SIZE`][page size] to `getBooks`, instead of just `PAGE_SIZE`.

### One page of books {#one-page}

Now update the model in **server/models/books.js**, so that `getBooks` returns one page of books.

**server/models/books.js**
```JavaScript{hl_lines="12 16-17"}
import { db } from "./db.js";

export function createBook({ title, author, year, thumbnailUrl, coverUrl }) {
    return db.prepare(`
        INSERT INTO books(title, author, year, thumbnailUrl, coverUrl)
        VALUES (:title, :author, :year, :thumbnailUrl, :coverUrl)
        RETURNING id, title, author, year, thumbnailUrl, coverUrl
    `).get({ title, author, year, thumbnailUrl, coverUrl });
}

// Placeholder: this will need to change when we add search
export function getBooks(page, pageSize) {
    return db.prepare(`
        SELECT id, title, author, year, thumbnailUrl, coverUrl FROM books
        ORDER BY title
        LIMIT :limit OFFSET :offset
    `).all({ limit: pageSize, offset: (page - 1) * pageSize });
}
```

> The placeholder is still there, because `getBooks` still can't search.

`getBooks` now takes the page number and the page size, rather than a limit.

[`LIMIT` and `OFFSET`][limit offset] ask the database for part of the results.
`OFFSET` skips some rows, and `LIMIT` says how many to return after that.
Page 1 skips none and returns the first 16, page 2 skips 16 and returns the next 16, and so on.
`:limit` and `:offset` are [named parameters][named parameters], just like the ones in `createBook`.

We already [sort the books][sorted] with [`ORDER BY title`][order by], and now it matters even more.

> Paging needs a stable order.
> Each page is a separate query, and the database only skips rows by counting them.
> If the order could change between two queries, a book could appear on two pages, or on none.
> `ORDER BY title` gives every query the same order.

Save both files and refresh [http://localhost:8000/books][books].
There are still 16 books, starting with **'Salem's Lot**, **11/22/63** and **1776**.
Now try [http://localhost:8000/books?page=2][page 2], and you get the next 16, from **A Court of Thorns and Roses** to **A Monster Calls**.
The last page, [http://localhost:8000/books?page=63][page 63], has only 8 books, from **Wonder** to **Zen and the Art of Motorcycle Maintenance**.

Each part of [model-view-controller][mvc] has its own job.
The model knows how to fetch one page of books, of any size.
The controller works out which page was asked for, and how many books go on it.
The view hasn't changed.
It still shows whatever books it is given.

> Try [http://localhost:8000/books?page=0][page 0] and [http://localhost:8000/books?page=two][page two].
> For now, both give you page 1.
> And [http://localhost:8000/books?page=100][page 100] has no books at all, because the database skips past every book.

## Counting the pages {#count}

Nobody is going to type `?page=2` into the address bar.
We need links to the previous and next pages, and for those, the view needs to know how many pages there are.

Add `countBooks` and `countPages` functions to the end of **server/models/books.js**.

**server/models/books.js**
```JavaScript{hl_lines="20-24 26-28"}
import { db } from "./db.js";

export function createBook({ title, author, year, thumbnailUrl, coverUrl }) {
    return db.prepare(`
        INSERT INTO books(title, author, year, thumbnailUrl, coverUrl)
        VALUES (:title, :author, :year, :thumbnailUrl, :coverUrl)
        RETURNING id, title, author, year, thumbnailUrl, coverUrl
    `).get({ title, author, year, thumbnailUrl, coverUrl });
}

// Placeholder: this will need to change when we add search
export function getBooks(page, pageSize) {
    return db.prepare(`
        SELECT id, title, author, year, thumbnailUrl, coverUrl FROM books
        ORDER BY title
        LIMIT :limit OFFSET :offset
    `).all({ limit: pageSize, offset: (page - 1) * pageSize });
}

function countBooks() {
    return db.prepare(`
        SELECT count(*) AS count FROM books
    `).get().count;
}

export function countPages(pageSize) {
    return Math.ceil(countBooks() / pageSize);
}
```

- [`count(*)`][count] counts the rows, and `AS count` names the column in the result.
- `.get()` returns that one row as an object, so `.get().count` is the number of books.
- `countBooks` isn't exported, because only `countPages` uses it.

`countPages` divides the number of books by the page size and rounds up with [`Math.ceil`][math ceil].
1,000 books at 16 per page is 62.5, so we have 63 pages, the last with 8 books.

Now update **server/controllers/books.js** to count the pages, and put `page` and `pageCount` on `ctx.data`, ready for the view.

**server/controllers/books.js**
```JavaScript{hl_lines="1 12-13 15"}
import { countPages, getBooks } from "../models/books.js";
import { render } from "../tools/render.js";
import { booksView } from "../views/books.js";

const PAGE_SIZE = 16;

export function booksController(ctx) {
    const { url } = ctx;
    const pageParam = url.searchParams.get("page");
    let page = pageParam === null ? 1 : Number(pageParam);
    if (!Number.isSafeInteger(page) || page < 1) page = 1;
    const pageCount = Math.max(1, countPages(PAGE_SIZE));
    ctx.data.page = page;
    ctx.data.books = getBooks(page, PAGE_SIZE);
    ctx.data.pageCount = pageCount;
    ctx.data.title = "Books";
    return render(booksView, ctx);
}
```

The controller only needs the number of pages, so `countPages` is the one we export and import.
With no books at all, `countPages` would give us 0.
[`Math.max`][math max] makes sure there is always at least one page.

Nothing on the page has changed yet, because the view doesn't use `page` or `pageCount`.

## Links to the other pages {#page-links}

Update **server/views/books.js** to add the links below the list.

**server/views/books.js**
```JavaScript{hl_lines="4 14 32-40"}
const DEFAULT_COVER = "/assets/default-cover.svg";

export function booksView(ctx) {
    const { data: { books, page, pageCount } } = ctx;

    return `
        <section>
            <header>
                <h1>Books</h1>
            </header>
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
                <p>${book.author} (${book.year})</p>
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
```

The view takes `page` and `pageCount` out of `ctx.data`, alongside the books.
`pagination` is another function used only by this view, so it isn't exported, and it stays in **books.js**.

- With only one page there is nothing to link to, so `pagination` returns an empty string straight away.
- The previous and next links use the [conditional operator][conditional operator]. Each one is only there if there is a page to go to, and otherwise it is an empty string, which adds nothing to the page.
- Each link is itself a template literal, inside the `${}` of another one.
- The current page is a link to itself, which says which page we are on and how many there are. It has [`aria-current="page"`][aria-current], just like the [current link][nav link] in our header.
- [`rel="prev"` and `rel="next"`][rel] tell browsers and screen readers how the pages relate.
- There are now two `<nav>` elements on the page. [`aria-label`][aria-label] gives this one a name, so screen reader users can tell it apart from the one in our header.

Refresh [http://localhost:8000/books][books].
Below the books are **page 1 of 63** and **next**.
Our `nav` rules in **public/styles.css** already style them like the [header links][nav styles]: a row of pills, with the current page filled in.
Click **next** twice, then **previous**.

{{<figure src="images/books-page-1.webp" alt="The bottom of the first page of books, with page 1 of 63 filled in and a next link below the last card" caption="The first page ends with links to the other pages.">}}

{{<figure src="images/books-page-2.webp" alt="The bottom of the second page of books, with previous, page 2 of 63 and next links" caption="Page 2 has a link back to page 1.">}}

View the page source (**Ctrl+U**) on [http://localhost:8000/books?page=2][page 2] and find the links.

```html {linenos=false}
            <nav aria-label="Pagination">
            <a href="/books?page=1" rel="prev">previous</a>
            <a href="/books?page=2" aria-current="page">page 2 of 63</a>
            <a href="/books?page=3" rel="next">next</a>
        </nav>
```

The last page, [http://localhost:8000/books?page=63][page 63], says **page 63 of 63** and has no **next** link.

> Notice that the server now tells the browser where it can go next, with links in the page.
> This idea is part of [HATEOAS][hateoas], and we will come back to it.

## A missing year {#missing-year}

Now that we can reach every page, the gaps in our data show.
Our books without a cover already [get the default cover][default cover], but one book has no year.
Click [http://localhost:8000/books?page=59][page 59] and find **Twilight: The Complete Illustrated Movie Companion**.
Its card says **Mark Cotta Vaz (null)**.

{{<figure src="images/missing-year.webp" alt="A book card for Twilight: The Complete Illustrated Movie Companion, with the author line Mark Cotta Vaz (null)" caption="This book has no year, so the card says null.">}}

This is the book with no year that we met in the [book search][missing year].

We can't make up a year, so a book without one should simply leave it out.
Update **server/views/books.js** again.

**server/views/books.js**
```JavaScript{hl_lines="26"}
const DEFAULT_COVER = "/assets/default-cover.svg";

export function booksView(ctx) {
    const { data: { books, page, pageCount } } = ctx;

    return `
        <section>
            <header>
                <h1>Books</h1>
            </header>
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
```

This is the conditional operator inside a template again, with a template literal inside the `${}`.
`book.year` is a number, not `true` or `false`, so how does it choose?

In a condition, JavaScript treats a few values as false.
These are called [*falsy*][falsy] values.
They include `""` (an empty string), `0`, `null` and `undefined`.
Anything else, including any number other than `0` and any non-empty string, counts as true.

- `null` is falsy, so a book with no year gets an empty string, and nothing is added after the author.
- Otherwise, we add a space and the year in brackets.

Refresh [http://localhost:8000/books?page=59][page 59].
The card now says just **Mark Cotta Vaz**.

## A page that doesn't exist {#invalid-page}

Click [http://localhost:8000/books?page=100][page 100].
The heading still says **Books**, but there are no cards.
A **previous** link and **page 100 of 63** sit in the middle of the page.

{{<figure src="images/page-past-end.webp" alt="The books page for page 100, with the heading Books, no book cards, a previous link and a filled button that says page 100 of 63, sitting mid-page" caption="Page 100 of 63 shouldn’t exist.">}}

And [http://localhost:8000/books?page=two][page two] quietly gives us page 1.

Neither page exists.
Like any other missing page, they should get our not-found page with `404 Not Found`.

Our router already [throws a `NotFoundError`][throw] when no route matches, and [`withErrors`][with errors] catches it and answers with the not-found page.
A controller can throw one too.

Update **server/controllers/books.js**.

**server/controllers/books.js**
```JavaScript{hl_lines="3 11 13-15"}
import { countPages, getBooks } from "../models/books.js";
import { render } from "../tools/render.js";
import { NotFoundError } from "../tools/errors.js";
import { booksView } from "../views/books.js";

const PAGE_SIZE = 16;

export function booksController(ctx) {
    const { url } = ctx;
    const pageParam = url.searchParams.get("page");
    const page = pageParam === null ? 1 : Number(pageParam);
    const pageCount = Math.max(1, countPages(PAGE_SIZE));
    if (!Number.isSafeInteger(page) || page < 1 || page > pageCount) {
        throw new NotFoundError(`Page ${pageParam} not found`);
    }
    ctx.data.page = page;
    ctx.data.books = getBooks(page, PAGE_SIZE);
    ctx.data.pageCount = pageCount;
    ctx.data.title = "Books";
    return render(booksView, ctx);
}
```

- The check comes after we count the pages, so `page > pageCount` catches a page past the end.
- Instead of quietly using page 1, we throw a `NotFoundError`, with a message that says which page was asked for.
- `page` is a `const` now, because we never change it after it is set.

The error leaves `booksController` straight away, and travels back up the chain to `withErrors`, which answers with our not-found page.

Click [http://localhost:8000/books?page=100][page 100] again.
It is `404 Not Found`, with our not-found page.

{{<figure src="images/page-not-found.webp" alt="The not-found page, shown for page 100 of the books" caption="Page 100 is now not found.">}}

`withErrors` put the error on `ctx.error`, so `withLogs` logs it like any other `404`, and the terminal says why.

```plaintext {linenos=false}

============================
GET /books
[error] Page 100 not found
RESPONSE 404
============================
```

[http://localhost:8000/books?page=two][page two], [http://localhost:8000/books?page=0][page 0] and [http://localhost:8000/books?page=64][page 64] are not found too, but [http://localhost:8000/books?page=63][page 63] still works.

## Conclusion

The books controller reads the page number from the query string with `url.searchParams`, and checks that it is a whole number.
The model fetches one page of books with `LIMIT` and `OFFSET`, in a stable order with `ORDER BY`, and counts the pages with `count(*)`.
The view links to the previous and next pages, and leaves out a missing year.
A page that doesn't exist throws a `NotFoundError`, so it gets our not-found page.

There are no new files in this exercise, so our project still looks like it did at the end of the [previous exercise][previous exercise].

```plaintext {linenos=false}
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

## Challenge: Test the page number

These are questions to investigate, not changes to keep.

- Visit [http://localhost:8000/books?page=1.0][page 1.0], [http://localhost:8000/books?page=0x2][page 0x2] and [http://localhost:8000/books?page=%202%20][page spaces]. Which pages do you get, and why? What does each link on the page point to?
- Visit [http://localhost:8000/books?page=two][page two] and look at the terminal. Why does the `[error]` line say `Page two not found`, rather than `Page NaN not found`?
- If our database had no books at all, what would `countPages` give us? What would happen to [/books][books] without `Math.max`? Work it out rather than emptying your database.
- Change `PAGE_SIZE` to 5. How many pages are there now? Work it out first, then check.

<details>
<summary>A stricter page number</summary>

`Number` is very forgiving.
`Number("1.0")` is `1`, and so are `Number(" 1 ")` and `Number("01")`.
`Number("0x2")` is `2`, because `0x` starts a hexadecimal number.

One way to be strict is to turn the number back into a string and check that it matches what the user sent.
`String(1)` is `"1"`, which is not the same as `"1.0"`.

**server/controllers/books.js**

```js
    if (
        !Number.isSafeInteger(page) || page < 1 || page > pageCount ||
        (pageParam !== null && String(page) !== pageParam)
    ) {
```

Now `1.0`, `0x2`, `" 2 "` and `02` are all rejected with a 404.
The `pageParam !== null` part matters: without a `page` in the query string, `pageParam` is `null`, and `String(1)` is never equal to `null`, so every visit to [/books][books] would be a 404.

A [regular expression][regex] such as `/^[1-9]\d*$/` would also work, but that is a new idea for another day.

</details>

> If you tried any changes, undo them, so that your code matches the starting point of the next exercise.

[previous exercise]: router-class/middleware/
[with logs]: router-class/middleware/#with-logs
[with errors]: router-class/middleware/#with-errors
[router url]: router-class/router/#handle
[url search]: https://developer.mozilla.org/en-US/docs/Web/API/URL/search
[searchparams]: https://developer.mozilla.org/en-US/docs/Web/API/URL/searchParams
[usp get]: https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams/get
[books]: http://localhost:8000/books
[default cover]: sqlite-and-models/database/#default-cover
[missing year]: javascript-fundamentals/books-data/#missing-year
[conditional operator]: views-and-controllers/controllers/#title
[falsy]: https://developer.mozilla.org/en-US/docs/Glossary/Falsy
[page 2]: http://localhost:8000/books?page=2
[page 59]: http://localhost:8000/books?page=59
[number]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/Number
[is safe integer]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/isSafeInteger
[page size]: sqlite-and-models/books-page/#page-size
[let]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let
[order by]: https://www.sqlite.org/lang_select.html#orderby
[sorted]: sqlite-and-models/database/#order-by
[limit offset]: https://www.sqlite.org/lang_select.html#limitoffset
[named parameters]: sqlite-and-models/database/#models
[mvc]: sqlite-and-models/books-page/#mvc
[page 0]: http://localhost:8000/books?page=0
[page two]: http://localhost:8000/books?page=two
[page 100]: http://localhost:8000/books?page=100
[count]: https://www.sqlite.org/lang_aggfunc.html#count
[math ceil]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/ceil
[math max]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/max
[aria-current]: https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Attributes/aria-current
[nav link]: router-class/one-context/#nav-link
[rel]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Attributes/rel
[aria-label]: https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Attributes/aria-label
[nav styles]: router-class/one-context/#nav-pills
[page 63]: http://localhost:8000/books?page=63
[page 64]: http://localhost:8000/books?page=64
[hateoas]: https://en.wikipedia.org/wiki/HATEOAS
[throw]: router-class/router/#throw
[page 1.0]: http://localhost:8000/books?page=1.0
[page 0x2]: http://localhost:8000/books?page=0x2
[page spaces]: http://localhost:8000/books?page=%202%20
[regex]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Regular_expressions

---
type: exercise
title: Page through the books
weight: 4
author: Dr Graeme Stuart
---

In this exercise we will seed all 1,000 books, and show them a page at a time, with links to the previous and next pages.
Along the way we will fix the books that have no cover or no year.

<!--more-->

> You should have every view and the layout using the `html` helper from the [previous exercise][previous exercise].
> Your database should hold the 20 books we seeded. If you have been experimenting, run `deno task db:reset` to get back to 20.
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

Our database has only ever held 20 books.
In this exercise we will add the other 980, and make our books page cope with them.
By the end of this exercise you should be able to:

- Show a default value when some data are missing
- Fetch one page of rows from the database with `ORDER BY`, `LIMIT` and `OFFSET`
- Turn a value from the query string into a number, and check that it is a whole number
- Count rows with `count(*)`, and work out how many pages there are
- Build a query string with `URLSearchParams`, so that our page links keep the search
- Throw a `NotFoundError` from a controller when a page doesn't exist

## All the books {#all-books}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Our seed task still has a temporary limit of 20 books.
Update **tasks/db-seed.js** to seed them all.

**tasks/db-seed.js**
```JavaScript{hl_lines="4"}
import books from "../db/books.json" with { type: "json" };
import { createBook } from "../server/models/books.js";

for (const book of books) {
    createBook(book);
}
```

The `Temporary:` comment has gone, along with the `slice`.

Reset the database.

```bash {linenos=false}
deno task db:reset
```

```plaintext {linenos=false}
Task db:reset deno task db:create && deno task db:seed
Task db:create deno -A tasks/db-create.js
Task db:seed deno -A tasks/db-seed.js
```

Open [http://localhost:8000/books][books] with the developer tools (**F12**) open on the **network** panel.
Our server is still quick: it sends the page in a few milliseconds.
The covers have [`loading="lazy"`][lazy loading], so the browser only fetches the ones we scroll to.
But the page is over 300 kB of HTML, with 1,000 cards, and the scroll bar is tiny.
Nobody is going to scroll through 1,000 books, and every visit sends all of them, whatever our users actually look at.
With more books it would only get longer.

Scroll down to **Memoirs of a Geisha**, the 36th book.
Its card has no cover, just an empty space.

{{<figure src="images/missing-cover.webp" alt="The books page scrolled down to Memoirs of a Geisha. Its card, and the Little Women card below it, show only a broken image icon where the cover should be" caption="Memoirs of a Geisha has no cover.">}}

We noted when we [added covers][covers] that some books in **books.json** have no cover, so their `coverUrl` is `null`.
None of our first 20 books were like that, but 181 of the others are.
View the page source (**Ctrl+U**) and find **Memoirs of a Geisha**.

```html
<img class="cover" src="" alt="" width="200" height="300" loading="lazy">
```

Our `html` helper turns `null` into an [empty string][to html], so the image has an empty `src` and nothing to show.

Now press **Ctrl+F** and find **Movie Companion**.
The first match is book 267, **Twilight: The Complete Illustrated Movie Companion**, by **Mark Cotta Vaz ()**.
This is the book with no year that we met in the [book search][missing year].
Its `year` is `null` too, so all we see is an empty pair of brackets.

We will fix both of these first, then split that long page into pages.

## A default cover {#default-cover}

A book with no cover should still look like a book.

Save {{<a href="images/default-cover.svg" download="default-cover.svg" >}}this default cover{{< /a >}} as **default-cover.svg** in the **public** folder.
Click [http://localhost:8000/assets/default-cover.svg][default cover] to check that it is served.

Now update **server/views/partials/bookCard.js** to use it when a book has no cover.

**server/views/partials/bookCard.js**
```JavaScript{hl_lines="3 6 9"}
import { html } from "../../tools/html.js";

const DEFAULT_COVER = "/assets/default-cover.svg";

export function bookCard(book) {
    const cover = book.coverUrl ?? DEFAULT_COVER;
    return html`
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

`DEFAULT_COVER` is the address of our default cover.
It isn't exported, because only this module uses it.
Writing a constant like this in capitals is a common convention for a fixed value.

`??` is the [nullish coalescing operator][nullish coalescing], which we first used for missing years in the book search.
If `book.coverUrl` is `null` (or `undefined`), `cover` is `DEFAULT_COVER`.
Otherwise, `cover` is `book.coverUrl`.

Refresh [http://localhost:8000/books][books].
**Memoirs of a Geisha** and **Little Women** now have our default cover.

{{<figure src="images/default-cover.webp" alt="The same part of the books page, where Memoirs of a Geisha and Little Women now show a grey book icon as their cover" caption="Books without a cover now get the default one.">}}

## A missing year {#missing-year}

We can't make up a year, so a book without one should simply leave it out.
Update **server/views/partials/bookCard.js** again.

**server/views/partials/bookCard.js**
```JavaScript{hl_lines="12"}
import { html } from "../../tools/html.js";

const DEFAULT_COVER = "/assets/default-cover.svg";

export function bookCard(book) {
    const cover = book.coverUrl ?? DEFAULT_COVER;
    return html`
        <li class="card book">
            <img class="cover" src="${cover}" alt="" width="200" height="300" loading="lazy">
            <div>
                <h2>${book.title}</h2>
                <p>${book.author}${book.year ? ` (${book.year})` : ""}</p>
            </div>
        </li>
    `;
}
```

This is the [conditional operator][conditional operator] inside a template again, just like the "show all books" link.

- `null` is [falsy][falsy], so a book with no year gets an empty string, and nothing is added after the author.
- Otherwise, we add a space and the year in brackets, with a template literal inside the `${}`.

Refresh [http://localhost:8000/books][books] and find **Movie Companion** again.
The card now says just **Mark Cotta Vaz**.

## One page at a time {#limit-offset}

Every card looks right now, but the page is still 1,000 books long.
Instead, we will show 16 at a time, with links to move between them.

### Which page? {#page-number}

We need a way to ask for a particular page, and we already have one: the [query string][query string].
We will use a `page` parameter, so [http://localhost:8000/books?page=2][page 2] asks for page 2.

Update **server/controllers/books.js** to work out which page was asked for.

**server/controllers/books.js**
```JavaScript{hl_lines="5 10-12 14"}
import { searchBooks } from "../models/books.js";
import { render } from "../tools/render.js";
import { booksView } from "../views/books.js";

const PAGE_SIZE = 16;

export function booksController(ctx) {
    const { url } = ctx;
    const q = url.searchParams.get("q") ?? "";
    const pageParam = url.searchParams.get("page");
    let page = pageParam === null ? 1 : Number(pageParam);
    if (!Number.isSafeInteger(page) || page < 1) page = 1;
    ctx.data.q = q;
    ctx.data.books = searchBooks(q, page, PAGE_SIZE);
    ctx.data.title = q ? `Books matching "${q}"` : "Books";
    return render(booksView, ctx);
}
```

- `PAGE_SIZE` is the number of books on a page. The controller decides how many books to show, so the constant lives here.
- We [read the parameter][search params] with `url.searchParams.get("page")`, just as we read `q`. It gives us the string `"2"` for `?page=2`, and `null` if there is no `page` at all.
- With no `page` at all, we use page 1. Otherwise, [`Number`][number] turns the string into a number.

Anyone can type anything into the query string, so we can't trust it.
`Number` turns `""` into `0`, `"abc"` into `NaN` and `"1.5"` into `1.5`.
[`Number.isSafeInteger`][is safe integer] is only true for a whole number, and `page < 1` catches zero and negative numbers.
For now, in all of those cases we use page 1 instead.

> `page` is declared with [`let`][let] rather than `const`, because we might change it on the next line.

Finally, we pass `page` and `PAGE_SIZE` to `searchBooks`.
`searchBooks` doesn't take them yet, so it simply ignores them.
Refresh [http://localhost:8000/books?page=2][page 2], and you should still see all 1,000 books.

### One page of books {#one-page}

Now update the model in **server/models/books.js**, so that `searchBooks` returns only one page of books.

**server/models/books.js**
```JavaScript{hl_lines="11 15-17"}
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
```

> The placeholder has gone, because this is the pagination it was waiting for.

`searchBooks` now takes the page number and the page size, as well as the search.

[`ORDER BY`][order by] sorts the results, here by `title`.
Without it, we have never said what order we want, so SQLite is free to return the rows in any order it likes.
So far they have come back in the order we added them, but nothing promises that.

[`LIMIT` and `OFFSET`][limit offset] ask the database for part of the results.
`OFFSET` skips some rows, and `LIMIT` says how many to return after that.
Page 1 skips none and returns the first 16, page 2 skips 16 and returns the next 16, and so on.
`:limit` and `:offset` are [named parameters][named parameters], just like `:pattern`.

> Paging needs a stable order.
> Each page is a separate query, and the database only skips rows by counting them.
> If the order could change between two queries, a book could appear on two pages, or on none.
> `ORDER BY title` gives every query the same order.

Refresh [http://localhost:8000/books][books].
There are 16 books, and the page is a fraction of the size.
They are in order of title now, starting with **'Salem's Lot**, **11/22/63** and **1776**.
Now try [http://localhost:8000/books?page=2][page 2], and you get the next 16, from **A Court of Thorns and Roses** to **A Monster Calls**.

Searches are paged too.
[http://localhost:8000/books?q=the][q the] shows the first 16 books with **the** in the title or author, and [http://localhost:8000/books?q=the&page=2][q the page 2] shows the next 16.

Each part of [model-view-controller][mvc] has its own job.
The model knows how to fetch one page of books, of any size.
The controller works out which page was asked for, and how many books go on it.
The view hasn't changed.
It still shows whatever books it is given.

> Try [http://localhost:8000/books?page=0][page 0] and [http://localhost:8000/books?page=abc][page abc].
> For now, both give you page 1.
> And [http://localhost:8000/books?page=100][page 100] says **No books found**, because the database skips past every book.

## Links to the other pages {#page-links}

Nobody is going to type `?page=2` into the address bar.
We need links to the previous and next pages.

The view needs to know how many pages there are, which means we need to know how many books match the search.
Add `countBooks` and `countPages` functions to the end of **server/models/books.js**.

**server/models/books.js**
```JavaScript{hl_lines="20-24 27-29"}
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

`countBooks` uses the same `WHERE` clause as `searchBooks`, so it counts only the books that match the search.

- [`count(*)`][count] counts the rows, and `AS count` names the column in the result.
- `.get()` returns that one row as an object, so `.get(...).count` is the number of books.
- `countBooks` isn't exported, because only `countPages` uses it.

`countPages` divides the number of books by the page size and rounds up with [`Math.ceil`][math ceil].
1,000 books at 16 per page is 62.5, so we have 63 pages.

Now update **server/controllers/books.js** to count the pages, and pass `page` and `pageCount` to the view.

**server/controllers/books.js**
```JavaScript{hl_lines="1 13 15 17"}
import { countPages, searchBooks } from "../models/books.js";
import { render } from "../tools/render.js";
import { booksView } from "../views/books.js";

const PAGE_SIZE = 16;

export function booksController(ctx) {
    const { url } = ctx;
    const q = url.searchParams.get("q") ?? "";
    const pageParam = url.searchParams.get("page");
    let page = pageParam === null ? 1 : Number(pageParam);
    if (!Number.isSafeInteger(page) || page < 1) page = 1;
    const pageCount = Math.max(1, countPages(q, PAGE_SIZE));
    ctx.data.q = q;
    ctx.data.page = page;
    ctx.data.books = searchBooks(q, page, PAGE_SIZE);
    ctx.data.pageCount = pageCount;
    ctx.data.title = q ? `Books matching "${q}"` : "Books";
    return render(booksView, ctx);
}
```

A search that matches nothing has no pages at all, so `countPages` gives us 0.
[`Math.max`][math max] makes sure there is always at least one page.

Finally, update **server/views/books.js** to add the links.

**server/views/books.js**
```JavaScript{hl_lines="6 19 24-32"}
import { html } from "../tools/html.js";
import { bookCard } from "./partials/bookCard.js";
import { searchForm } from "./partials/searchForm.js";

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
            ${pagination(page, pageCount)}
        </section>
    `;
}

// Previous, the current page and next. Nothing at all if there's only one page
function pagination(page, pageCount) {
    if (pageCount <= 1) return "";
    return html`<nav aria-label="Pagination">
            ${page > 1 ? html`<a href="/books?page=${page - 1}" rel="prev">previous</a>` : ""}
            <a href="/books?page=${page}" aria-current="page">page ${page} of ${pageCount}</a>
            ${page < pageCount ? html`<a href="/books?page=${page + 1}" rel="next">next</a>` : ""}
        </nav>`;
}
```

The view takes `page` and `pageCount` out of `ctx.data`, alongside the others.
`pagination` is another function used only by this view, so it isn't exported.

- With only one page there is nothing to link to, so `pagination` returns an empty string straight away.
- The previous and next links use the conditional operator. Each one is only there if there is a page to go to.
- The links are HTML we wrote, so they use `html`, [nested inside][nesting] the `html` around the `<nav>`, just like `navLink` in our layout.
- The current page is a link to itself, which says which page we are on and how many there are. It has [`aria-current="page"`][aria-current], just like the [current link][nav link] in our header.
- [`rel="prev"` and `rel="next"`][rel] say what each link is for.
- There are now two `<nav>` elements on the page. [`aria-label`][aria-label] gives this one a name, so screen reader users can tell it apart from the one in our header.

Refresh [http://localhost:8000/books][books].
Below the books are **page 1 of 63** and **next**.
Our `nav` rules in **public/styles.css** already style them like the [header links][nav styles]: a centred row of pills, with the current page filled in.
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

The last page, [http://localhost:8000/books?page=63][page 63], has 8 books, from **Wonder** to **Zen and the Art of Motorcycle Maintenance**, says **page 63 of 63** and has no **next** link.

> Notice that the server now tells the browser where it can go next, with links in the page.
> This idea is part of [HATEOAS][hateoas], and we will come back to it.

### Losing the search {#lost-search}

Now visit [http://localhost:8000/books?q=the][q the].
It says **page 1 of 29**, because only the matching books are counted.

Click **next**.
The address bar says `/books?page=2`, the heading says **Books**, and we are on **page 2 of 63**.
Our links only include the page, so following one loses the search.

## Keeping the search {#page-url}

Each link needs the search *and* the page in its query string.
Rather than build that three times, let's write one function for it.

Update **server/views/books.js**.

**server/views/books.js**
```JavaScript{hl_lines="19 25 28-30 34-38"}
import { html } from "../tools/html.js";
import { bookCard } from "./partials/bookCard.js";
import { searchForm } from "./partials/searchForm.js";

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
            <a href="/books?q=of+the&amp;page=1" rel="prev">previous</a>
            <a href="/books?q=of+the&amp;page=2" aria-current="page">page 2 of 3</a>
            <a href="/books?q=of+the&amp;page=3" rel="next">next</a>
        </nav>
```

`pageUrl` returns a plain string, so our `html` helper escaped the `&` between the parameters as `&amp;`.
That is exactly what we want: the browser reads `&amp;` in an attribute as `&`, so the link goes to the right URL.

[http://localhost:8000/books][books], with no search, still links to `/books?page=2`.

## A page that doesn't exist {#invalid-page}

Click [http://localhost:8000/books?page=100][page 100].
There are no books, a **previous** link, and **page 100 of 63**.

{{<figure src="images/page-past-end.webp" alt="The books page for page 100, showing No books found, a previous link and page 100 of 63" caption="Page 100 of 63 shouldn’t exist.">}}
And [http://localhost:8000/books?page=abc][page abc] quietly gives us page 1.

Neither page exists.
Like any other missing page, they should get our not-found page with `404 Not Found`.

Our router already [throws a `NotFoundError`][throw] when no route matches, and **server/main.js** [catches it][catch] and shows the not-found page.
A controller can throw one too.

Update **server/controllers/books.js**.

**server/controllers/books.js**
```JavaScript{hl_lines="3 12 14-16"}
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

- The check comes after we count the pages, so `page > pageCount` catches a page past the end.
- Instead of quietly using page 1, we throw a `NotFoundError`, with a message that says which page was asked for.
- `page` is a `const` now, because we never change it.

The error leaves `booksController` straight away, passes back through `withLogs` and the router, and reaches the `catch` in **server/main.js**.
`withLogs` is `async`, but we already [await the router][await router], so the `catch` still sees it.

Click [http://localhost:8000/books?page=100][page 100] again.
It is `404 Not Found`, with our not-found page, and the terminal says why.

{{<figure src="images/page-not-found.webp" alt="The not-found page, shown for page 100 of the books" caption="Page 100 is now not found.">}}

```plaintext {linenos=false}

============================
GET /books
Not found: Page 100 not found
```

[http://localhost:8000/books?page=abc][page abc], [http://localhost:8000/books?page=0][page 0] and [http://localhost:8000/books?page=64][page 64] are not found too, but [http://localhost:8000/books?page=63][page 63] still works.
A search has its own number of pages, so [http://localhost:8000/books?q=of+the&page=3][q of the page 3] works, and [http://localhost:8000/books?q=of+the&page=4][q of the page 4] is not found.

> Notice that `withLogs` never logs a `RESPONSE` for these, just as when we [awaited the router][await router].
> The error skips the rest of `withLogs` on its way to the `catch`.

## Conclusion

We seeded all 1,000 books, and found some with no cover or no year.
`bookCard` now uses a default cover, and leaves out a missing year.

The books controller reads the page number from the query string, and asks the model for one page of books with `LIMIT` and `OFFSET`, in a stable order with `ORDER BY`, and for the number of pages.
The view links to the previous and next pages, and `pageUrl` uses `URLSearchParams` to keep the search in each link.
A page that doesn't exist throws a `NotFoundError`, so it gets our not-found page.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="9"}
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
│   │   ├── home.js
│   │   ├── notFound.js
│   │   └── static.js
│   ├── middleware/
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
│   │   │   ├── bookCard.js
│   │   │   └── searchForm.js
│   │   ├── books.js
│   │   ├── home.js
│   │   ├── layout.js
│   │   └── notFound.js
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
- Search for **harry potter**. Why are there no page links?
- Visit [http://localhost:8000/books?q=zebra][q zebra]. How many pages does it have, and what would happen without `Math.max`?
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

[previous exercise]: user-input-and-security/escaping/
[books]: http://localhost:8000/books
[covers]: sqlite-and-models/books-page/#cover
[to html]: user-input-and-security/escaping/#html
[missing year]: javascript-fundamentals/books-data/#missing-year
[default cover]: http://localhost:8000/assets/default-cover.svg
[nullish coalescing]: javascript-fundamentals/books-data/#missing-year
[conditional operator]: user-input-and-security/search/#title
[falsy]: user-input-and-security/search/#title
[query string]: user-input-and-security/search/#query-string
[search params]: user-input-and-security/search/#search-params
[page 2]: http://localhost:8000/books?page=2
[number]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/Number
[is safe integer]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/isSafeInteger
[let]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let
[order by]: https://www.sqlite.org/lang_select.html#orderby
[limit offset]: https://www.sqlite.org/lang_select.html#limitoffset
[lazy loading]: sqlite-and-models/books-page/#lazy-loading
[named parameters]: user-input-and-security/search/#like
[q the]: http://localhost:8000/books?q=the
[q the page 2]: http://localhost:8000/books?q=the&page=2
[mvc]: sqlite-and-models/database/#mvc
[page 0]: http://localhost:8000/books?page=0
[page abc]: http://localhost:8000/books?page=abc
[page 100]: http://localhost:8000/books?page=100
[count]: https://www.sqlite.org/lang_aggfunc.html#count
[math ceil]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/ceil
[math max]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/max
[nesting]: user-input-and-security/escaping/#nesting
[aria-current]: https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Attributes/aria-current
[nav link]: router-class/middleware/#nav-link
[rel]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Attributes/rel
[aria-label]: https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Attributes/aria-label
[nav styles]: router-class/middleware/#nav-styles
[page 63]: http://localhost:8000/books?page=63
[page 64]: http://localhost:8000/books?page=64
[hateoas]: https://en.wikipedia.org/wiki/HATEOAS
[url search params]: https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams
[shorthand]: router-class/router/#classes
[q of the]: http://localhost:8000/books?q=of+the
[q of the page 3]: http://localhost:8000/books?q=of+the&page=3
[q of the page 4]: http://localhost:8000/books?q=of+the&page=4
[throw]: router-class/router/#throw
[catch]: router-class/router/#catch
[await router]: router-class/middleware/#await-router
[page 1.0]: http://localhost:8000/books?page=1.0
[page 0x2]: http://localhost:8000/books?page=0x2
[page spaces]: http://localhost:8000/books?page=%202%20
[q zebra]: http://localhost:8000/books?q=zebra
[regex]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Regular_expressions

---
type: exercise
title: Paging through the books
weight: 10
author: Dr Graeme Stuart
---

In this exercise we will show our books a page at a time, with links to the previous and next pages.
Along the way we will give books with no cover a default one.

<!--more-->

> You should have a books page showing 20 books from the [previous exercise][previous exercise].
> If you have been experimenting, run `deno task db:reset` to get back to 20.
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

We finished the books page by asking what would happen with all 1,000 books.
In this exercise we will find out, and fix it.
By the end of this exercise you should be able to:

- Show a default value when some data are missing, using `??`
- Read a value from the *query string* of a URL
- Add a value to our *context* object when a controller needs it
- Fetch one page of rows from the database with `LIMIT` and `OFFSET`
- Link each page of books to the previous and next pages
- Respond with `404 Not Found` for a page that doesn't exist

## Setup

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Open [http://localhost:8000/books][books] with the developer tools (**F12**) open on the **network** panel.
You should see our 20 books.

## More books {#more-books}

Twenty books fit comfortably on one page.
Let's see what happens with a few more.

Update **tasks/db-seed.js** to seed 40 books.

**tasks/db-seed.js**
```JavaScript{hl_lines="4-5"}
import books from "../db/books.json" with { type: "json" };
import { createBook } from "../server/models/books.js";

// Temporary: seed only 40 books until we introduce paging
for (const book of books.slice(0, 40)) {
    createBook(book);
}
```

Reset the database.

```bash {linenos=false}
deno task db:reset
```

```plaintext {linenos=false}
Task db:reset deno task db:create && deno task db:seed
Task db:create deno -A tasks/db-create.js
Task db:seed deno -A tasks/db-seed.js
```

Refresh [http://localhost:8000/books][books].

```http
HTTP/1.1 200 OK
content-type: text/html
content-length: 13727
date: Fri, 30 Oct 2026 10:26:52 GMT
```

The page is twice as long, and two of the cards have a broken image.
Scroll down to **Memoirs of a Geisha** and **Little Women**.

When we added covers, we noted that some books in **books.json** have no cover, so their `coverUrl` is `null`.
None of our first 20 books were like that, but these two are.

Look at the terminal.

```plaintext {linenos=false}
GET /books
GET /null
```

Our card puts `src="null"` in the `<img>`, so the browser [asks for][linked resources] `/null`.
There is no such page, so our server sends back our [not-found page][not-found page] with `404 Not Found`, which isn't an image.
You can see the request in the **network** panel too.

## A default cover {#default-cover}

A book with no cover should still look like a book.

Save {{<a href="images/default-cover.svg" download="default-cover.svg" >}}this default cover{{< /a >}} as **default-cover.svg** in the **public** folder.
Click [http://localhost:8000/assets/default-cover.svg][default cover] to check that it is served.

Now update **server/views/partials/bookCard.js** to use it when a book has no cover.

**server/views/partials/bookCard.js**
```JavaScript{hl_lines="1 4 7"}
const DEFAULT_COVER = "/assets/default-cover.svg";

export function bookCard(book) {
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

`DEFAULT_COVER` is the address of our default cover.
It isn't exported, because only this module uses it.
Writing a constant like this in capitals is a common convention for a fixed value.

`??` is the [nullish coalescing operator][nullish coalescing].
If `book.coverUrl` is `null` (or `undefined`), `cover` is `DEFAULT_COVER`.
Otherwise, `cover` is `book.coverUrl`.

Refresh [http://localhost:8000/books][books].
**Memoirs of a Geisha** and **Little Women** now have our default cover, and there are no more requests for `/null`.

## Which page? {#query-string}

Our books page is getting long, and it will be far longer with 1,000 books.
Instead, we will show the books a page at a time.

We need a way to ask for a particular page.
Click [http://localhost:8000/books?page=2][books page 2].

The page hasn't changed.
Look at the terminal.

```plaintext {linenos=false}
GET /books
```

The part of the URL after the `?` is the [*query string*][url search].
We log [`url.pathname`][url pathname], which doesn't include it.
Our server ignores the query string, so `/books?page=2` gets exactly the same page as `/books`.

To read the query string, the books controller needs the `url`.

### Adding the url to ctx {#ctx-url}

We already make a `url` in **server/main.js**, and we already pass our [context object][ctx] to every controller.
So let's put the `url` in `ctx` too.

Update **server/main.js**.

**server/main.js**
```JavaScript{hl_lines="8"}
import { homeController } from "./controllers/home.js";
import { booksController } from "./controllers/books.js";
import { staticController } from "./controllers/static.js";
import { notFoundController } from "./controllers/notFound.js";

export function server(request) {
    const url = new URL(request.url);
    const ctx = { request, url };

    // Placeholder: this will need to change when we add middleware
    console.log(request.method, url.pathname);

    // Placeholder: this will need to change when we add a router
    if (url.pathname.startsWith("/assets/")) {
        return staticController(ctx);
    }
    if (url.pathname === "/") {
        return homeController(ctx);
    }
    if (url.pathname === "/books") {
        return booksController(ctx);
    }
    return notFoundController(ctx);
}
```

Now `ctx` holds the `request` and the `url` we have already made.
The books controller doesn't declare `ctx` yet, so refresh [http://localhost:8000/books][books] to check that nothing has changed.

### Reading the page number {#page-number}

Update **server/controllers/books.js** to take `ctx` and work out which page was asked for.

**server/controllers/books.js**
```JavaScript{hl_lines="5 7-12"}
import { getBooks } from "../models/books.js";
import { render } from "../tools/render.js";
import { booksView } from "../views/books.js";

const PAGE_SIZE = 16;

export function booksController(ctx) {
    const { url } = ctx;
    const pageParam = url.searchParams.get("page");
    let page = pageParam === null ? 1 : Number(pageParam);
    if (!Number.isSafeInteger(page) || page < 1) page = 1;
    const books = getBooks(page, PAGE_SIZE);
    return render(booksView, { title: "Books", data: { books } });
}
```

The controller uses `ctx` now, so it declares it.

- `PAGE_SIZE` is the number of books on a page. The controller decides how many books to show, so the constant lives here.
- `const { url } = ctx;` takes the `url` out of `ctx`, with [destructuring][destructuring]. It is the same `URL` that **server/main.js** uses for `url.pathname`.
- [`url.searchParams`][search params] reads the query string for us. `get("page")` gives us the string `"2"` for `?page=2`, and `null` if there is no `page` at all.
- With no `page` at all we use page 1. Otherwise, [`Number`][number] turns the string into a number.

Anyone can type anything into the query string, so we can't trust it.
`Number` turns `""` into `0`, `"abc"` into `NaN` and `"1.5"` into `1.5`.
[`Number.isSafeInteger`][is safe integer] is only true for a whole number, and `page < 1` catches zero and negative numbers.
For now, in all of those cases we use page 1 instead.

> `page` is declared with [`let`][let] rather than `const`, because we might change it on the next line.

Finally, we pass `page` and `PAGE_SIZE` to `getBooks`.
`getBooks` doesn't take them yet, so it simply ignores them.
Refresh [http://localhost:8000/books?page=2][books page 2], and you should still see all 40 books.

### One page of books {#limit-offset}

Now update the [model][mvc] in **server/models/books.js** so that `getBooks` returns only one page of books.

**server/models/books.js**
```JavaScript{hl_lines="12 15-16"}
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
        LIMIT :limit OFFSET :offset
    `).all({ limit: pageSize, offset: (page - 1) * pageSize });
}
```

`getBooks` now takes the page number and the page size.

[`LIMIT` and `OFFSET`][limit offset] ask the database for part of the results.
`OFFSET` skips some rows, and `LIMIT` says how many to return after that.
Page 1 skips none and returns the first 16, page 2 skips 16 and returns the next 16, and so on.
`:limit` and `:offset` are [named parameters][named parameters], just like the ones in `createBook`.

Refresh [http://localhost:8000/books][books].
There are 16 books.
Now try [http://localhost:8000/books?page=2][books page 2].

```http
HTTP/1.1 200 OK
content-type: text/html
content-length: 6255
date: Wed, 04 Nov 2026 10:54:00 GMT
```

A different 16 books.
[http://localhost:8000/books?page=3][books page 3] has the last 8, including **Memoirs of a Geisha** and **Little Women**.

> Try [http://localhost:8000/books?page=0][books page 0] and [http://localhost:8000/books?page=abc][books page abc].
> For now, both give you page 1.

Each part of [model-view-controller][mvc] has its own job.
The model knows how to fetch one page of books, of any size.
The controller works out which page was asked for, and how many books go on it.
The view hasn't changed.
It still shows whatever books it is given.

## Links to the other pages {#page-links}

Nobody is going to type `?page=2` into the address bar.
We need links to the previous and next pages.

The view needs to know how many pages there are, which means we need to know how many books there are.
Add `countBooks` and `countPages` functions to the end of **server/models/books.js**.

**server/models/books.js**
```JavaScript{hl_lines="19-24 26-29"}
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
        LIMIT :limit OFFSET :offset
    `).all({ limit: pageSize, offset: (page - 1) * pageSize });
}

// Placeholder: this will need to change when we add search
function countBooks() {
    return db.prepare(`
        SELECT count(*) AS count FROM books
    `).get().count;
}

// Placeholder: this will need to change when we add search
export function countPages(pageSize) {
    return Math.ceil(countBooks() / pageSize);
}
```

[`count(*)`][count] counts the rows, and `AS count` names the column in the result.
`.get()` returns that one row as an object, so `.get().count` is the number of books.
`countBooks` isn't exported, because only `countPages` uses it.

`countPages` divides the number of books by the page size and rounds up with [`Math.ceil`][math ceil].
40 books at 16 per page is 2.5, so we have 3 pages.

> Like `getBooks`, both new functions have a placeholder.
> This code will need to change when we add search.

Now update **server/controllers/books.js** to count the pages, and pass `page` and `pageCount` to the view.

**server/controllers/books.js**
```JavaScript{hl_lines="1 12 14"}
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
    const books = getBooks(page, PAGE_SIZE);
    return render(booksView, { title: "Books", data: { books, page, pageCount } });
}
```

With no books at all, `countPages` would give us 0.
[`Math.max`][math max] makes sure there is always at least one page.

Finally, update **server/views/books.js** to add the links.

**server/views/books.js**
```JavaScript{hl_lines="3 12 17-25"}
import { bookCard } from "./partials/bookCard.js";

export function booksView({ data: { books, page, pageCount } }) {
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

The view takes `page` and `pageCount` out of `data`, alongside `books`.
`pagination` is another function used only by this view, so it isn't exported.

- With only one page there is nothing to link to, so `pagination` returns an empty string straight away.
- The previous and next links use the [conditional operator][conditional operator], just like our page titles. Each one is only there if there is a page to go to.
- The current page is a link to itself, which says which page we are on and how many there are. [`aria-current="page"`][aria-current] tells screen readers that this is the page we are on.
- [`rel="prev"` and `rel="next"`][rel] say what each link is for.
- There are now two `<nav>` elements on the page. [`aria-label`][aria-label] gives this one a name, so screen reader users can tell it apart from the one in our header.

Refresh [http://localhost:8000/books][books].
Below the books are **page 1 of 3** and **next**.
Our `nav` rule in **public/styles.css** puts them in a centred row, just like the header links.
Click **next** twice, then **previous**.
On page 3 there is no **next** link.

View the page source (**Ctrl + U**) on [http://localhost:8000/books?page=2][books page 2] and find the links.

```html {linenos=false}
            <nav aria-label="Pagination">
            <a href="/books?page=1" rel="prev">previous</a>
            <a href="/books?page=2" aria-current="page">page 2 of 3</a>
            <a href="/books?page=3" rel="next">next</a>
        </nav>
```

> Notice that the server now tells the browser where it can go next, with links in the page.
> This idea is part of [HATEOAS][hateoas], and we will come back to it.

### Building the links {#page-url}

We build a `/books?page=` URL three times in `pagination`.
Let's move that into one function.

Update **server/views/books.js**.

**server/views/books.js**
```JavaScript{hl_lines="21-23 27-29"}
import { bookCard } from "./partials/bookCard.js";

export function booksView({ data: { books, page, pageCount } }) {
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

// Previous, the current page and next. Nothing at all if there's only one page
function pagination(page, pageCount) {
    if (pageCount <= 1) return "";
    return `<nav aria-label="Pagination">
            ${page > 1 ? `<a href="${pageUrl(page - 1)}" rel="prev">previous</a>` : ""}
            <a href="${pageUrl(page)}" aria-current="page">page ${page} of ${pageCount}</a>
            ${page < pageCount ? `<a href="${pageUrl(page + 1)}" rel="next">next</a>` : ""}
        </nav>`;
}

function pageUrl(page) {
    return `/books?${new URLSearchParams({ page })}`;
}
```

[`URLSearchParams`][url search params] builds a query string from an object.
`new URLSearchParams({ page: 2 })` becomes `page=2` inside our template literal, so `pageUrl(2)` is `/books?page=2`.
The links are now built in one place.

This is [refactoring][refactoring].
Refresh [http://localhost:8000/books?page=2][books page 2] and check the **network** panel.

```http
HTTP/1.1 200 OK
content-type: text/html
content-length: 6500
date: Wed, 04 Nov 2026 10:54:03 GMT
```

The page source should be exactly the same as before.

## A page that doesn't exist {#invalid-page}

Click [http://localhost:8000/books?page=4][books page 4].
There are no books, a **previous** link, and **page 4 of 3**.
And [http://localhost:8000/books?page=abc][books page abc] quietly gives us page 1.

Neither page exists.
Like any other missing page, they should get our [not-found page][not-found page] with `404 Not Found`.

Update **server/controllers/books.js**.

**server/controllers/books.js**
```JavaScript{hl_lines="4 11 13-15"}
import { countPages, getBooks } from "../models/books.js";
import { render } from "../tools/render.js";
import { booksView } from "../views/books.js";
import { notFoundController } from "./notFound.js";

const PAGE_SIZE = 16;

export function booksController(ctx) {
    const { url } = ctx;
    const pageParam = url.searchParams.get("page");
    const page = pageParam === null ? 1 : Number(pageParam);
    const pageCount = Math.max(1, countPages(PAGE_SIZE));
    if (!Number.isSafeInteger(page) || page < 1 || page > pageCount) {
        return notFoundController();
    }
    const books = getBooks(page, PAGE_SIZE);
    return render(booksView, { title: "Books", data: { books, page, pageCount } });
}
```

- The books controller imports `notFoundController`, and returns its response for a page that doesn't exist. A controller can use another controller like this.
- The check comes after we count the pages, so `page > pageCount` catches a page past the end.
- `page` is a `const` now, because we never change it.

Click [http://localhost:8000/books?page=4][books page 4] again.

```http
HTTP/1.1 404 Not Found
content-type: text/html
content-length: 1083
date: Wed, 04 Nov 2026 10:54:05 GMT
```

[http://localhost:8000/books?page=abc][books page abc] and [http://localhost:8000/books?page=0][books page 0] are not found too, but [http://localhost:8000/books?page=3][books page 3] still works.

{{<figure src="images/books-page-not-found.webp" alt="The Bookshelves not-found page, headed Not found, with the text The requested resource was not found" caption="A page past the end, such as page 100, gets our not-found page.">}}

## All the books {#all-books}

Our pages can cope with any number of books now.
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

```bash {linenos=false}
deno task db:reset
```

```plaintext {linenos=false}
Task db:reset deno task db:create && deno task db:seed
Task db:create deno -A tasks/db-create.js
Task db:seed deno -A tasks/db-seed.js
```

Refresh [http://localhost:8000/books][books] and click **next** a few times.
1,000 books at 16 per page is 63 pages.
The last one, [http://localhost:8000/books?page=63][books page 63], has 8 books, says **page 63 of 63** and has no **next** link.
[http://localhost:8000/books?page=64][books page 64] is our not-found page.

{{<figure src="images/books-page-1.webp" alt="The books page showing 16 book cards in four columns, with page 1 of 63 and a next link below them" caption="Page 1 of 63, with only a next link.">}}

{{<figure src="images/books-page-2.webp" alt="The books page showing a different 16 book cards, with previous, page 2 of 63 and next links below them" caption="Page 2 has both previous and next links.">}}

Each page is still 16 books, however many are in the database.

## Conclusion

We seeded more books and found some with no cover, so `bookCard` now uses a default cover.
We added the `url` to our context object, so that the books controller can read the page number from the query string.
The controller asks the model for one page of books with `LIMIT` and `OFFSET`, and for the number of pages, so the view can link to the previous and next pages.
A page that doesn't exist gets our not-found page.
The database now holds all 1,000 books.

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
│   ├── models/
│   │   ├── books.js
│   │   └── db.js
│   ├── tools/
│   │   └── render.js
│   ├── views/
│   │   ├── partials/
│   │   │   └── bookCard.js
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

## Challenge: A missing year

Click [http://localhost:8000/books?page=17][books page 17].
One of the books shows **(null)** for its year.

Fix it.

> We will start the next exercise with our solution, so you can compare it with yours.

[previous exercise]: deno-server/books-page/
[books]: http://localhost:8000/books
[linked resources]: deno-server/static-files/#linked-resources
[not-found page]: deno-server/views/#html-404
[default cover]: http://localhost:8000/assets/default-cover.svg
[nullish coalescing]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing
[books page 2]: http://localhost:8000/books?page=2
[url search]: https://developer.mozilla.org/en-US/docs/Web/API/URL/search
[url pathname]: deno-server/static-files/#url-pathname
[ctx]: deno-server/controllers/#ctx
[destructuring]: deno-server/controllers/#title
[search params]: https://developer.mozilla.org/en-US/docs/Web/API/URL/searchParams
[number]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/Number
[is safe integer]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/isSafeInteger
[let]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/let
[mvc]: deno-server/database/#mvc
[limit offset]: https://www.sqlite.org/lang_select.html#limitoffset
[named parameters]: deno-server/database/#models
[books page 3]: http://localhost:8000/books?page=3
[books page 0]: http://localhost:8000/books?page=0
[books page abc]: http://localhost:8000/books?page=abc
[count]: https://www.sqlite.org/lang_aggfunc.html#count
[math ceil]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/ceil
[math max]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/max
[conditional operator]: deno-server/controllers/#title
[aria-current]: https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Attributes/aria-current
[rel]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Attributes/rel
[aria-label]: https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Attributes/aria-label
[hateoas]: https://en.wikipedia.org/wiki/HATEOAS
[url search params]: https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams
[refactoring]: deno-server/views/#refactoring
[books page 4]: http://localhost:8000/books?page=4
[books page 63]: http://localhost:8000/books?page=63
[books page 64]: http://localhost:8000/books?page=64
[books page 17]: http://localhost:8000/books?page=17

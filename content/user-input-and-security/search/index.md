---
type: exercise
title: Search for books
weight: 2
author: Dr Graeme Stuart
---

In this exercise we will add a search form, so our users can find books by title or author.
It is the first time our application accepts input from its users.

<!--more-->

> You should have a router and a `withLogs` middleware from the [previous exercise][previous exercise].
> Your database should hold the 20 books we seeded. If you have been experimenting, run `deno task db:reset` to get back to 20.
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

The search is a new *feature*, and our first *user input*.
We will build it one small step at a time, checking the browser after each one.
By the end of this exercise you should be able to:

- Explain what a *query string* is, and how a form builds one
- Write an HTML search form with a label, an input and a button
- Read a value from the query string with `url.searchParams`
- Filter rows in SQL with `LIKE`, and explain why we pass the user's search as a parameter
- Use *falsy* values, such as an empty string or `0`, in a conditional

## The query string {#query-string}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Now visit [http://localhost:8000/books?q=hobbit][q hobbit].

The part of the URL after the `?` is the [*query string*][url search].
It holds *parameters*, written as `name=value`, so this one has a parameter called `q` with the value `hobbit`.
Several parameters are joined with `&`, as in `?q=hobbit&lang=en`.

Our page hasn't changed: it still shows all 20 books.
The router matches on the path, `/books`, and nothing reads the query string yet.
Our `withLogs` middleware [logs `url.pathname`][with logs], which doesn't include the query string, so the terminal only says `GET /books`.

We want our users to type a search, not edit the URL, so let's give them a form.

## A search form {#form}

Update **server/views/books.js** to add a search form to the books page.

**server/views/books.js**
```JavaScript{hl_lines="11 19-29"}
import { bookCard } from "./partials/bookCard.js";

export function booksView(ctx) {
    const { data: { books } } = ctx;

    return `
        <section>
            <header>
                <h1>Books</h1>
            </header>
            ${searchForm()}
            <ul class="grid">
                ${books.map(bookCard).join("")}
            </ul>
        </section>
    `;
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

`searchForm` is a small function in the same module, just like `navLink` in our layout.
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

The page still shows all 20 books, because our server still ignores the query string.

## Searching the books {#search}

To search, our model needs a new query, and the books controller needs to read the search from the URL.
These two files depend on each other, so update both before you refresh.

### A search query {#like}

Update **server/models/books.js**, replacing `getBooks` with `searchBooks`.

**server/models/books.js**
```JavaScript{hl_lines="11-12 15-16"}
import { db } from "./db.js";

export function createBook({ title, author, year, thumbnailUrl, coverUrl }) {
    return db.prepare(`
        INSERT INTO books(title, author, year, thumbnailUrl, coverUrl)
        VALUES (:title, :author, :year, :thumbnailUrl, :coverUrl)
        RETURNING id, title, author, year, thumbnailUrl, coverUrl
    `).get({ title, author, year, thumbnailUrl, coverUrl });
}

// Placeholder: this will need to change when we add pagination
export function searchBooks(q) {
    return db.prepare(`
        SELECT id, title, author, year, thumbnailUrl, coverUrl FROM books
        WHERE title LIKE :pattern OR author LIKE :pattern
    `).all({ pattern: `%${q}%` });
}
```

> The old placeholder has gone, because this is the search it was waiting for.
> The new one is for pagination: with all 1,000 books, one search could match far too many.

The `WHERE` clause keeps only the rows we want.

- [`LIKE`][sqlite like] compares text with a *pattern*. In a pattern, `%` matches any run of characters, including none at all.
- So the pattern `%hobbit%` means "contains **hobbit** anywhere". We build it with a [template literal][template literals], `` `%${q}%` ``.
- `LIKE` ignores the difference between capital and small letters, so we don't need the [`toLowerCase`][lower case] we used in the book search.
- `OR` keeps a row if *either* test is true, like the [`||`][logical or] in the book search, so we search both the title and the author.

We use the same *named parameter*, `:pattern`, twice, and pass its value to `.all()`, just as `createBook` does with its [named parameters][named parameters].

### Why a parameter matters {#parameters}

This is the first time our SQL includes something a user typed.
We could have pasted `q` straight into the SQL with a template literal, but we must never do that.

A parameter is sent to SQLite *separately* from the SQL.
SQLite treats whatever is in the parameter as a value to compare, never as part of the query.
So, whatever our users type, they can only ever change what we search *for*, not *how* we search.

> Pasting user input into SQL is how [SQL injection][sql injection] attacks happen.
> Parameters are the standard defence. See the [OWASP advice on query parameterisation][owasp params] if you'd like to know more.

### Reading the query string {#search-params}

Now update **server/controllers/books.js**.

**server/controllers/books.js**
```JavaScript{hl_lines="1 6-9"}
import { searchBooks } from "../models/books.js";
import { render } from "../tools/render.js";
import { booksView } from "../views/books.js";

export function booksController(ctx) {
    const { url } = ctx;
    const q = url.searchParams.get("q") ?? "";
    ctx.data.q = q;
    ctx.data.books = searchBooks(q);
    ctx.data.title = "Books";
    return render(booksView, ctx);
}
```

The router already put the `url` on `ctx` [for its own use][router url], so we can take it out of `ctx` here too, with no other changes.

[`url.searchParams`][searchparams] reads the query string for us.

- [`get("q")`][usp get] gives us the value of the `q` parameter as a string, already decoded, so `harry+potter` becomes `"harry potter"`.
- If there is no `q` parameter at all, `get` gives us `null`. We use [`??`][nullish coalescing] to fall back to an empty string, just as we did with `prompt` in the book search.

We pass `q` to `searchBooks`, and also put it on `ctx.data`, ready for the view.

Save both files, then try some searches.

- [http://localhost:8000/books?q=hobbit][q hobbit] shows one book, **The Hobbit**.
- [http://localhost:8000/books?q=tolkien][q tolkien] shows two, because it matches the author too.
- [http://localhost:8000/books?q=harry+potter][q harry potter] shows three.
- [http://localhost:8000/books][books] still shows all 20. With no `q`, the pattern is `%%`, which matches everything.

Use the form too.
It works, but after each search the box is empty again.

## Keeping the search in the box {#value}

Update **server/views/books.js** to pass the search to the form.

**server/views/books.js**
```JavaScript{hl_lines="4 11 19 24"}
import { bookCard } from "./partials/bookCard.js";

export function booksView(ctx) {
    const { data: { books, q } } = ctx;

    return `
        <section>
            <header>
                <h1>Books</h1>
            </header>
            ${searchForm(q)}
            <ul class="grid">
                ${books.map(bookCard).join("")}
            </ul>
        </section>
    `;
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
```JavaScript{hl_lines="10"}
import { searchBooks } from "../models/books.js";
import { render } from "../tools/render.js";
import { booksView } from "../views/books.js";

export function booksController(ctx) {
    const { url } = ctx;
    const q = url.searchParams.get("q") ?? "";
    ctx.data.q = q;
    ctx.data.books = searchBooks(q);
    ctx.data.title = q ? `Books matching "${q}"` : "Books";
    return render(booksView, ctx);
}
```

This is the [conditional operator][conditional operator] again.
`q` is a string, not `true` or `false`, so how does it choose?

In a condition, JavaScript treats a few values as false. These are called [*falsy*][falsy] values.
They include `""` (an empty string), `0`, `null` and `undefined`.
Anything else, including any non-empty string, counts as true.

So when `q` is `""`, the title is just **Books**, and when the user has searched, it is **Books matching "tolkien"**.

Now update **server/views/books.js** to show the title, with a link back to all the books.

**server/views/books.js**
```JavaScript{hl_lines="4 9-10"}
import { bookCard } from "./partials/bookCard.js";

export function booksView(ctx) {
    const { data: { title, books, q } } = ctx;

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
        </section>
    `;
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

The line under it uses the conditional operator inside the template.
When there is a search, it gives a paragraph with a link to [http://localhost:8000/books][books]. Otherwise, it gives an empty string, which adds nothing to the page.
The paragraph is itself a template literal, inside the `${}` of another one.

Visit [http://localhost:8000/books?q=tolkien][q tolkien].
The heading and the tab both say **Books matching "tolkien"**, and **show all books** takes us back to every book.

## No matches {#no-results}

Search for something that isn't there, such as [http://localhost:8000/books?q=zebra][q zebra].
The heading says **Books matching "zebra"**, then there is just an empty space.
We should say that nothing matched.

Update **server/views/books.js**.

**server/views/books.js**
```JavaScript{hl_lines="5-7 16"}
import { bookCard } from "./partials/bookCard.js";

export function booksView(ctx) {
    const { data: { title, books, q } } = ctx;
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
        </section>
    `;
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

## One form, two pages {#partial}

Our home page would be a good place to search from, too.
Two views need the same form, so it should be a [partial][partials], like `bookCard`.

Create **searchForm.js** in the **server/views/partials** folder, and move `searchForm` into it.

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
```JavaScript{hl_lines="2"}
import { bookCard } from "./partials/bookCard.js";
import { searchForm } from "./partials/searchForm.js";

export function booksView(ctx) {
    const { data: { title, books, q } } = ctx;
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
        </section>
    `;
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
The form's `action` sends us to the books page, `/books?q=orwell`, with George Orwell's two books.

{{<figure src="images/search-home.webp" alt="The Bookshelves home page, with the heading Welcome to Bookshelves, a Find a book heading, and an empty search form labelled Search by title or author" caption="The home page now has the same search form.">}}

## Styling the form {#styles}

Finally, let's make the form match the rest of our site.
Update **public/styles.css**.
First, give buttons the same look as our nav links.

**public/styles.css**
```css {linenostart="136" hl_lines="1-2"}
/* Buttons and nav links share one look. The current page is filled in */
button,
nav a {
    padding: 0.25em 0.75em;
    border: 1px solid;
    border-radius: 1em;
    background: none;
    color: inherit;
    text-decoration: none;
    &:hover {
        text-decoration: underline var(--highlight) 2px;
    }
}
```

Then add these rules at the end of the file.

**public/styles.css**
```css {linenostart="150"}
/* An input may shrink below its default width, so the search row never overflows */
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
    button {
        justify-self: start;
    }
}

/* The search box and its button share a row, under the label */
search > form {
    grid-template-columns: 1fr auto;
    label {
        grid-column: 1 / -1;
    }
}
```

- `font: inherit` makes inputs and buttons use our page font. Browsers give them their own font by default.
- Each `form` is a [grid][grid] with one column, so each label, input and button sits on its own row.
- `search > form` is a [child combinator][child combinator]: it selects a form directly inside a `<search>` element. Our search form gets two columns, the box and the button, with the label stretched across both with [`grid-column: 1 / -1`][grid-column].
- `min-width: 0` lets the box shrink on a narrow screen, rather than pushing the button off the edge.

Refresh [http://localhost:8000/books?q=tolkien][q tolkien] to see the finished search.

{{<figure src="images/search-results.webp" alt="The books page after a search for tolkien, with the heading Books matching “tolkien”, a show all books link, the search form with tolkien in the box, and two book cards for The Hobbit and The Fellowship of the Ring" caption="A search for tolkien shows the two matching books, with the query kept in the form.">}}

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
The page keeps the search in the box and in its title, and says when nothing matches.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="31"}
my-app/
├── .vscode/
│   └── settings.json
├── db/
│   ├── app.db
│   ├── BOOKS-ATTRIBUTION.md
│   └── books.json
├── public/
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

## Challenge: Explore the search

These are questions to investigate, not changes to keep.

- Search for **TOLKIEN** in capitals. Why does it still match?
- Visit [http://localhost:8000/books?q=tolkien&q=rowling][q two]. Which search does the page show, and what does that tell you about `get`?
- Search for `%`, and then for `_`. Why do both show every book? Look up what `_` means in a [`LIKE` pattern][sqlite like].
- Search for three spaces. What does the heading say, and why does nothing match? How could the string method [`trim`][trim] help?

> If you tried any changes, undo them, so that your code matches the starting point of the next exercise.

[previous exercise]: router-class/middleware/
[with logs]: router-class/middleware/#with-logs
[router url]: router-class/router/#handle
[url search]: https://developer.mozilla.org/en-US/docs/Web/API/URL/search
[q hobbit]: http://localhost:8000/books?q=hobbit
[q tolkien]: http://localhost:8000/books?q=tolkien
[q harry potter]: http://localhost:8000/books?q=harry+potter
[q zebra]: http://localhost:8000/books?q=zebra
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
[named parameters]: sqlite-and-models/database/#models
[sql injection]: https://en.wikipedia.org/wiki/SQL_injection
[owasp params]: https://cheatsheetseries.owasp.org/cheatsheets/Query_Parameterization_Cheat_Sheet.html
[searchparams]: https://developer.mozilla.org/en-US/docs/Web/API/URL/searchParams
[usp get]: https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams/get
[nullish coalescing]: javascript-fundamentals/book-search/#prompt
[conditional operator]: views-and-controllers/controllers/#title
[falsy]: https://developer.mozilla.org/en-US/docs/Glossary/Falsy
[partials]: sqlite-and-models/books-page/#partials
[default parameters]: views-and-controllers/controllers/#options
[grid]: sqlite-and-models/books-page/#grid
[child combinator]: https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Selectors/Child_combinator
[grid-column]: https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/grid-column
[trim]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/trim

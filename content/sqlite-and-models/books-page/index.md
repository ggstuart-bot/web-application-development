---
type: exercise
title: A books page
weight: 8
author: Dr Graeme Stuart
---

In this exercise we will add a books page which shows the first page of our books as cards, with their covers.
Along the way we will write our first *model*, and pass data from the model, through a controller, to a view.

<!--more-->

> You should have a home page and a not-found page, each with its own controller, from the [previous exercise][previous exercise].
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

The books page is a new *feature*.
We will build it one small step at a time, checking the browser after each one.
By the end of this exercise you should be able to:

- Add data to a project, and explain why we credit its source
- Add a page with a view and a controller, and explain the technical debt it adds
- Write a *model* which looks after our data, and explain where models fit in model-view-controller
- Pass data from a model to a view through `render`
- Turn an array of books into HTML with `map` and `join`
- Lay out a page of cards with CSS grid, including a grid inside each card

## Setup

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Open [http://localhost:8000/][localhost] with the developer tools (**F12**) open on the **network** panel.
Click the **books** link in the header.

```http
HTTP/1.1 404 Not Found
content-type: text/html
content-length: 1139
date: Thu, 29 Oct 2026 14:33:35 GMT
```

The link has been there since we added the header, but it goes to our not-found page.
Let's fix that.

## The data {#data}

*Bookshelves* needs books.
We will use the same list of the 1,000 most-rated books on Goodreads that we met in [Working with the books data][books data], from a public dataset called [goodbooks-10k][goodbooks].

Create a folder called **db** in the root of the project.
Save {{<a href="files/books.json" download="books.json" >}}this list of books{{< /a >}} as **books.json** and {{<a href="files/BOOKS-ATTRIBUTION.txt" download="BOOKS-ATTRIBUTION.md" >}}this attribution file{{< /a >}} as **BOOKS-ATTRIBUTION.md**, both in the **db** folder.

**books.json** is a JSON array with one book on each line.
Here's the start of it.

**db/books.json**
```json{linenostart=1}
[
    { "title": "The Hunger Games", "author": "Suzanne Collins", "year": 2008, "thumbnailUrl": "https://images.gr-assets.com/books/1447303603m/2767052.jpg", "coverUrl": "https://images.gr-assets.com/books/1447303603l/2767052.jpg" },
```

Each book has a `title`, an `author` and a `year`, plus links to a small and a large cover image.
Some books have no cover images, so their `thumbnailUrl` and `coverUrl` are `null`.

### Crediting the data {#attribution}

We didn't create this data, so we must say where it came from.

**db/BOOKS-ATTRIBUTION.md**
```markdown
# Book data

- `books.json` is derived from goodbooks-10k by Zygmunt Zając
  (https://github.com/zygmuntz/goodbooks-10k), licensed CC BY-SA 4.0
  (https://creativecommons.org/licenses/by-sa/4.0/). Changes: the 1,000 most-rated books; titles,
  authors and years cleaned; multiple authors joined into one field. This derived data is shared
  under CC BY-SA 4.0.
- Cover images (thumbnailUrl, coverUrl) belong to their publishers; they are linked, not copied.
```

The original dataset is shared under the [CC BY-SA 4.0][cc by-sa] licence.
*BY* means anyone who uses it must credit the author.
*SA* (share alike) means anything we make from it must be shared under the same licence.
We changed the data, so the attribution says what we changed, and our version is shared under the same licence.

> Crediting your sources is good practice anyway.
> With data like this, it's also a condition of using it at all.

> The cover images belong to their publishers.
> We only link to them, we don't copy them into our project.

Nothing uses the data yet, so our website is unchanged.

## A books view and controller {#books-controller}

This will feel familiar.
Every page so far has a view and a [controller][controllers], and the books page is no different.

Create **books.js** in the **server/views** folder.

**server/views/books.js**
```JavaScript
export function booksView() {
    return `
        <section>
            <header>
                <h1>Books</h1>
                <p>All the books in Bookshelves.</p>
            </header>
        </section>
    `;
}
```

Then create **books.js** in the **server/controllers** folder.

**server/controllers/books.js**
```JavaScript
import { render } from "../tools/render.js";
import { booksView } from "../views/books.js";

export function booksController() {
    return render(booksView, { title: "Books" });
}
```

Click [http://localhost:8000/books][books].
It's still our not-found page, with `404 Not Found` in the **network** panel.

Nothing calls `booksController` yet.
Our request handler in **server/main.js** decides which controller to call, and it doesn't know about the books page.

## Routing to the books page {#route}

Update **server/main.js** to import the new controller and call it for `/books`.

**server/main.js**
```JavaScript{hl_lines="2 20-22"}
import { homeController } from "./controllers/home.js";
import { booksController } from "./controllers/books.js";
import { staticController } from "./controllers/static.js";
import { notFoundController } from "./controllers/notFound.js";

export function server(request) {
    const url = new URL(request.url);
    const ctx = { request };

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

Like every controller, `booksController` is passed our [context object][ctx].
It doesn't need anything from it yet, so it doesn't declare it.

Refresh [http://localhost:8000/books][books].

```http
HTTP/1.1 200 OK
content-type: text/html
content-length: 1123
date: Thu, 29 Oct 2026 14:33:37 GMT
```

The tab says **Books | Bookshelves** and the page has a heading.
[http://localhost:8000/][localhost] and [http://localhost:8000/nope][nope] are just as they were.

### Another if {#if-chain}

Look at the `if` statements in **server/main.js**.
Every page we add needs another one, and they all look the same.
This is [technical debt][technical debt], and it grows with every page.
The router placeholder is a reminder that we will pay it off when we add a router.

## A books model {#models}

Our page has no books yet.
The books are in **db/books.json**, and the view needs them.

We could import the file into our controller, or even into our view.
Instead, we will give our data a module of its own.
Create a folder called **models** inside the **server** folder, with a file called **books.js**.

**server/models/books.js**
```JavaScript
import books from "../../db/books.json" with { type: "json" };

// Placeholder: this will need to change when we add search
export function getBooks() {
    return books;
}
```

We import the JSON file just as we did when we [imported the books][import books] in the terminal, so `books` is an array of objects.
`getBooks` returns them.

> Another placeholder.
> When we add search, `getBooks` will need to find only the matching books.

### The M in MVC {#mvc}

**server/models/books.js** is a *model*.
Models are the M in [model-view-controller][mvc views].
We already have [views][mvc views] which present information and [controllers][mvc controllers] which handle requests.
A model looks after our data, and the rest of our code asks the model rather than reading the data itself.

Just like views and controllers, a model is a set of related functions grouped into a module.

## Passing data to the view {#passing-data}

`render` calls the view, so `render` must pass the books on.

Update **server/tools/render.js** so that `render` takes a `data` option and gives it to the view.

**server/tools/render.js**
```JavaScript{hl_lines="4 6"}
import { layout } from "../views/layout.js";

// Placeholder: this will need to change when we add flash messages and sign-in
export function render(view, { status, title, data } = {}) {
    const headers = new Headers({ "content-type": "text/html" });
    return new Response(layout(view({ data }), { title }), { headers, status });
}
```

`data` is another property in our [options object][options].
`render` now calls `view({ data })`, so every view receives an object with a `data` property.

> Our home and not-found views take no arguments, so they simply ignore it.
> Refresh [http://localhost:8000/][localhost] to check.

Now update **server/controllers/books.js** to get the books from the model and pass them to `render`.

**server/controllers/books.js**
```JavaScript{hl_lines="1 6-7"}
import { getBooks } from "../models/books.js";
import { render } from "../tools/render.js";
import { booksView } from "../views/books.js";

export function booksController() {
    const books = getBooks();
    return render(booksView, { title: "Books", data: { books } });
}
```

The controller asks the model for the books with `getBooks`, and passes them to `render` inside `data`.

Finally, update **server/views/books.js** to use the books.

**server/views/books.js**
```JavaScript{hl_lines="1 7-9 14-18"}
export function booksView({ data: { books } }) {
    return `
        <section>
            <header>
                <h1>Books</h1>
            </header>
            <ul>
                ${books.map(bookItem).join("")}
            </ul>
        </section>
    `;
}

function bookItem(book) {
    return `
        <li>${book.title}</li>
    `;
}
```

`booksView({ data: { books } })` uses [destructuring][destructuring] twice.
It takes the `data` property from the argument, and then the `books` property from `data`.
So inside the view, `books` is our array of books.

`bookItem` takes one book and returns a list item containing its title.

[`map`][array map] calls a function on every item in an array and gives us a new array of the results.
We pass `bookItem` itself to `map`, without brackets, just as we [passed views to render][first-class function].
So `books.map(bookItem)` is an array of strings, one for each book.

> `bookItem` isn't exported.
> It is only used by the books view, so it stays in this module.

[`join`][array join] joins the strings in an array into one string.
We give it `""` so that nothing goes between them.

> Without `join`, the template literal would turn the array into a string itself, with a comma between each item.

Refresh [http://localhost:8000/books][books].
You should see a very long bulleted list of titles, all 1,000 of them.
View the page source (**Ctrl + U**) and look at the list.
Here's the start of it.

```html {linenos=false}
            <ul>
                
        <li>The Hunger Games</li>
    
        <li>Harry Potter and the Sorcerer's Stone</li>
    
        <li>Twilight</li>
```

Each `<li>` is one string from `map`, joined with nothing between them.
The extra lines and spaces come from the template in `bookItem`, and the browser ignores them.

All three parts of [model-view-controller][mvc] now work together.
The controller asks the model for the books, and gives them to the view.
The view doesn't know where the books came from, and the model doesn't know anything about HTML.

{{<figure src="images/books-list.webp" alt="A Bookshelves page headed Books, with a long bulleted list of book titles" caption="The books page shows every title as one long bulleted list.">}}

## The first page of books {#page-size}

A thousand books is far too many for one page.
For now, we will show only the first few.

Update **server/controllers/books.js** to ask for a fixed number of books.

**server/controllers/books.js**
```JavaScript{hl_lines="5 8"}
import { getBooks } from "../models/books.js";
import { render } from "../tools/render.js";
import { booksView } from "../views/books.js";

const PAGE_SIZE = 16;

export function booksController() {
    const books = getBooks(PAGE_SIZE);
    return render(booksView, { title: "Books", data: { books } });
}
```

`PAGE_SIZE` is the number of books on a page, set in one place at the top of the module.
Writing a constant like this in capitals is a common convention for a fixed value.

Then update **server/models/books.js** so that `getBooks` takes a `limit`.

**server/models/books.js**
```JavaScript{hl_lines="4-5"}
import books from "../../db/books.json" with { type: "json" };

// Placeholder: this will need to change when we add search
export function getBooks(limit) {
    return books.slice(0, limit);
}
```

[`slice`][slice] gives us a new array with only the first `limit` books, just as it did in the terminal.

The model decides *how* to find the books, and the controller decides *how many* it wants.

Refresh [http://localhost:8000/books][books].
The page now lists the first 16 titles.

## A grid of books {#grid}

A long bulleted list isn't a good way to browse books.
We want them side by side, in as many columns as fit the window.

Give the list a `class` in **server/views/books.js**.

**server/views/books.js**
```JavaScript{hl_lines="7"}
export function booksView({ data: { books } }) {
    return `
        <section>
            <header>
                <h1>Books</h1>
            </header>
            <ul class="grid">
                ${books.map(bookItem).join("")}
            </ul>
        </section>
    `;
}

function bookItem(book) {
    return `
        <li>${book.title}</li>
    `;
}
```

Then add a `.grid` rule to the end of **public/styles.css**.

**public/styles.css**
```css{linenostart=83}
/* auto-fill makes as many columns as fit, each at least 15rem wide */
.grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(15rem, 1fr));
    gap: 1rem;
    padding: 0;
    list-style: none;
}
```

We have used a grid before, for the [rows of our page][styles].
This time we set the columns.

- [*grid-template-columns*][grid-template-columns] describes the columns.
- [`repeat(auto-fill, …)`][repeat] makes as many columns as will fit.
- [`minmax(15rem, 1fr)`][minmax] makes each column at least *15rem* wide, and shares out any space left over equally.
- *gap* puts space between the books.
- `padding: 0` and `list-style: none` remove the indent and bullets that a list has *by default*.

Refresh [http://localhost:8000/books][books] and change the width of the window.
The number of columns changes to fit.
In the picture there are three, with Catching Fire alone on the last row.

> `class="grid"` describes the layout, not the books.
> Any list we want to show as a grid can use it.

{{<figure src="images/books-grid.webp" alt="The books page with the 16 titles in as many 15rem columns as fit the window, three in this picture, with Catching Fire alone on the last row" caption="The books page now shows the titles in a grid.">}}

## A card for each book {#book-card}

Each book will be a *card*, a box with the book's details in it.

Update **server/views/books.js**, replacing `bookItem` with a `bookCard` function.

**server/views/books.js**
```JavaScript{hl_lines="8 14 16-21"}
export function booksView({ data: { books } }) {
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
    return `
        <li class="card">
            <div>
                <h2>${book.title}</h2>
                <p>${book.author} (${book.year})</p>
            </div>
        </li>
    `;
}
```

It has the same shape as `bookItem`, with a `class` on the `<li>` and the author and year added.
`booksView` now maps the books with `bookCard`.

> Like `bookItem`, `bookCard` is a local function, not exported.
> The books page is the only place that uses it.

Each card has a heading, so let's give `<h2>` elements the same weight as `<h1>` near the top of **public/styles.css**.
They share the comment about headings.

**public/styles.css**
```css{linenostart=15 hl_lines="2-3"}
/* Headings are a little heavier than the light body text */
h1,
h2 {
    font-weight: 400;
}
```

Then add a `.card` rule to the end of the file.

**public/styles.css**
```css{linenostart=93}
/* A card is a bordered box with rounded corners */
.card {
    border: 1px solid;
    border-radius: 0.5rem;

    > div:last-child {
        padding: 0.5rem;
    }

    /* Headings and paragraphs in a card have no margins and stay at the text size */
    h2,
    p {
        margin: 0;
        font-size: 1em;
    }
}
```

[`> div:last-child`][last-child] is nested inside `.card`, like the `& > header` in our `body` rule.
It means the last `<div>` directly inside a card, which holds the text.
The headings and paragraphs inside a card lose their margins and are all the same size, as the comment says.

Refresh [http://localhost:8000/books][books].
Each book is now a bordered box with its title, author and year.

{{<figure src="images/books-cards.webp" alt="The books page with each book in a bordered box showing its title, author and year, and only the top edge of Catching Fire visible at the bottom" caption="Each book is now a card with its title, author and year.">}}

## Covers {#cover}

Books are easier to find by their covers.
Add an [`<img>`][img] to the card in **server/views/books.js**.

**server/views/books.js**
```JavaScript{hl_lines="17"}
export function booksView({ data: { books } }) {
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
    return `
        <li class="card">
            <img class="cover" src="${book.coverUrl}" alt="" width="200" height="300" loading="lazy">
            <div>
                <h2>${book.title}</h2>
                <p>${book.author} (${book.year})</p>
            </div>
        </li>
    `;
}
```

- `class="cover"` lets us style book covers.
- `alt=""` tells screen readers the image is decorative, because the title is right next to it.
- `width` and `height` tell the browser the shape of the image before it arrives, so the page doesn't jump about as covers load.
- [`loading="lazy"`][lazy loading] waits to load a cover until it is near the visible part of the page. We will [watch it at work][lazy demo] once our cards are finished.

> Some books in **books.json** have no cover, so their `coverUrl` is `null`.
> None of our first 16 books are like that, so we can leave them for now.

Add an `img` rule to **public/styles.css**, after the `nav` rule.

**public/styles.css**
```css{linenostart=84}
/* Pictures never overflow their box */
img {
    max-inline-size: 100%;
}
```

Pictures can be wider than the box they are in.
This rule stops any image from overflowing its box.

Then add `overflow: clip` to `.card`, and a `.cover` rule after it.

**public/styles.css**
```css{linenostart=98 hl_lines="5 18-25"}
/* A card is a bordered box with rounded corners */
.card {
    border: 1px solid;
    border-radius: 0.5rem;
    overflow: clip; /* the cover is cut to the rounded corners */

    > div:last-child {
        padding: 0.5rem;
    }

    /* Headings and paragraphs in a card have no margins and stay at the text size */
    h2,
    p {
        margin: 0;
        font-size: 1em;
    }
}

/* A book cover is always 2:3, cropped to fit */
.cover {
    inline-size: 100%;
    block-size: auto;
    aspect-ratio: 2 / 3;
    object-fit: cover;
}
```

- [*aspect-ratio*][aspect-ratio] makes every cover the same shape, *2 / 3*, however big it is.
- [*object-fit*][object-fit] with *cover* fills that shape, cropping the picture if it isn't quite the right shape.
- `overflow: clip` on the card cuts off the corners of the cover, so they follow the card's rounded border.

Refresh [http://localhost:8000/books][books].
Each card now has its cover at the top, the full width of the card.

{{<figure src="images/books-covers.webp" alt="The top of the books page, with each card showing a large cover above the title, author and year" caption="Each card now has its cover at the top.">}}

## A grid inside each card {#card-grid}

Our cards are tall.
We will put the cover and the text side by side, which needs a grid inside each card.

Start with the grid itself.
Update the `.card` rule in **public/styles.css**.

**public/styles.css**
```css{linenostart=98 hl_lines="1 3 7 9 11-13"}
/* A card is a bordered box with rounded corners. Each kind of card sets its own grid */
.card {
    display: grid;
    border: 1px solid;
    border-radius: 0.5rem;
    overflow: clip; /* the cover is cut to the rounded corners */
    overflow-wrap: anywhere; /* a very long word breaks, rather than squashing the cover */

    /* The text is spread from the top of the card to the bottom */
    > div:last-child {
        display: grid;
        align-content: space-between;
        gap: 0.5rem;
        padding: 0.5rem;
    }

    /* Headings and paragraphs in a card have no margins and stay at the text size */
    h2,
    p {
        margin: 0;
        font-size: 1em;
    }
}
```

- `display: grid` makes each card a grid. We haven't set any columns, so it has one column and the cover sits above the text, just as before.
- [*overflow-wrap*][overflow-wrap] with *anywhere* lets a very long word break, rather than making its column wider and squashing the cover.
- The text `<div>` is a grid too. [*align-content*][align-content] with *space-between* puts the title at the top of the card and the author at the bottom.

Refresh [http://localhost:8000/books][books].
The cover is still above the text, because each card still has one column.
The only difference is a little more space between the title and the author, from the new *gap*.

Different kinds of card will want different columns, so each kind sets its own.
Add a `.card.book` rule to the end of **public/styles.css**.

**public/styles.css**
```css{linenostart=130}
/* A book card: the cover in a narrow column, the text beside it */
.card.book {
    grid-template-columns: 2fr 3fr;
}
```

`.card.book` matches elements with both classes.
The two columns are measured in [`fr`][fr], a share of the space, so the cover gets two parts and the text gets three.

Add the `book` class to the card in **server/views/books.js**.

**server/views/books.js**
```JavaScript{hl_lines="16"}
export function booksView({ data: { books } }) {
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
    return `
        <li class="card book">
            <img class="cover" src="${book.coverUrl}" alt="" width="200" height="300" loading="lazy">
            <div>
                <h2>${book.title}</h2>
                <p>${book.author} (${book.year})</p>
            </div>
        </li>
    `;
}
```

Refresh [http://localhost:8000/books][books].
Each card now has its cover on the left and its title, author and year on the right.
Our books page is finished, for now.

> Make the window narrow and wide again.
> The cards and the columns all adjust together.

{{<figure src="images/books-finished.webp" alt="The finished books page, with all 16 cards showing a cover on the left and the title, author and year on the right, Catching Fire alone on the last row, and the footer visible" caption="The finished books page, with each cover beside its title, author and year.">}}

### Lazy loading {#lazy-loading}

Now that our cards are finished, let's watch our covers' `loading="lazy"` at work.
Open the developer tools (**F12**) on the **network** panel and click the **Img** filter, so that it only lists images.

Make the page narrow enough for a single column of books.
If your window won't go that narrow, drag the edge of the developer tools across, or open the device toolbar (**Ctrl + Shift + M**) and choose a phone.

Reload [http://localhost:8000/books][books].
Only the first few covers are requested.
Now scroll down slowly and watch the **network** panel.
Each cover is requested just before it comes into view.

> The browser starts loading a lazy image while it is still some distance below the window, so it's usually ready by the time you get there.
> With two or more columns, our 16 books fit near enough to the top that the browser loads nearly all the covers straight away.

## Conclusion

We added a books page, our first page to show data.
We followed the same pattern as before, a view, a controller and one more `if` in **server/main.js**.
We wrote our first model, the M in model-view-controller, which reads the books from **db/books.json**.
`render` gained a `data` option, so the controller can pass the books from the model to the view.
We turned the first 16 books into HTML with `map` and `join`, and laid out each book as a card, with a grid inside a grid.

We also added some debt.
Our `if` chain is longer.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="4-6 12 16-17 21"}
my-app/
├── .vscode/
│   └── settings.json
├── db/
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
│   ├── models/
│   │   └── books.js
│   ├── tools/
│   │   └── render.js
│   ├── views/
│   │   ├── books.js
│   │   ├── home.js
│   │   ├── layout.js
│   │   └── notFound.js
│   └── main.js
├── tasks/
│   └── server.js
├── deno.json
└── deno.lock
```

## Challenge: Experiment with the books page

- In the `.card.book` rule, try `1fr 1fr` or `1fr 3fr` instead of `2fr 3fr`. Which looks best at different window widths?
- In the `.grid` rule, change *15rem* to *10rem* or *25rem* and watch the number of columns.
- In the **Elements** panel, select a card and untick `display: grid` on `.card`, and then `object-fit: cover` on `.cover`. What does each one do?
- Change `PAGE_SIZE` to `100` and refresh [http://localhost:8000/books][books]. How long does the page take to scroll through? Do any cards look wrong?

> Put **public/styles.css** and `PAGE_SIZE` back to the versions above before moving on.

Finally, something to think about.
Our model reads every book from **db/books.json**, and then throws most of them away.
Where else could our books live, and what would we need to change if we moved them?

[previous exercise]: views-and-controllers/controllers/
[localhost]: http://localhost:8000/
[books data]: javascript-fundamentals/books-data/
[goodbooks]: https://github.com/zygmuntz/goodbooks-10k
[cc by-sa]: https://creativecommons.org/licenses/by-sa/4.0/
[controllers]: views-and-controllers/controllers/#controllers
[books]: http://localhost:8000/books
[ctx]: views-and-controllers/controllers/#ctx
[nope]: http://localhost:8000/nope
[technical debt]: views-and-controllers/controllers/#technical-debt
[import books]: javascript-fundamentals/books-data/#import
[mvc views]: views-and-controllers/views/#mvc
[mvc controllers]: views-and-controllers/controllers/#mvc
[mvc]: sqlite-and-models/books-page/#mvc
[options]: views-and-controllers/controllers/#options
[destructuring]: views-and-controllers/controllers/#title
[array map]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map
[array join]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/join
[first-class function]: views-and-controllers/controllers/#view-functions
[slice]: javascript-fundamentals/books-data/#slice
[styles]: views-and-controllers/views/#styles
[grid-template-columns]: https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/grid-template-columns
[repeat]: https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Values/repeat
[minmax]: https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Values/minmax
[last-child]: https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Selectors/:last-child
[img]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/img
[lazy loading]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/img#loading
[lazy demo]: sqlite-and-models/books-page/#lazy-loading
[aspect-ratio]: https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/aspect-ratio
[object-fit]: https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/object-fit
[overflow-wrap]: https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/overflow-wrap
[align-content]: https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/align-content
[fr]: https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Values/flex_value

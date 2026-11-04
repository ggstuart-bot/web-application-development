---
type: exercise
title: A shared layout and views
weight: 6
author: Dr Graeme Stuart
---

In this exercise we will make our not-found page look like the rest of our site.
Along the way we will split our HTML into a shared *layout* and a *view* for each page, so that we can add a header and footer to every page at once.

<!--more-->

> You should have a server which sends the home page for `/` and a `404 Not Found` response for anything else from the [previous exercise][previous exercise].
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

We have one problem to solve: our plain-text 404 looks like a different site.
Fixing it properly will lead us to the first part of a well-known way of organising web applications.
By the end of this exercise you should be able to:

- Return a complete HTML page with a `404 Not Found` status
- Remove repeated code by *refactoring*, without changing what the browser sees
- Write a `layout` function which wraps any page content in our shared HTML document
- Explain what a *view* is and where views fit in the model-view-controller pattern
- Add a header and footer to every page by changing a single function

## Setup

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Open [http://localhost:8000/][localhost] with the developer tools (**F12**) open on the **network** panel.
You should see our home page with its icon and stylesheet.

Now click [http://localhost:8000/books][books].
We get the words **Not found** in the browser's default font.
No stylesheet, no icon in the tab, and no way back to the home page.
It looks like a different site.

## An HTML not-found page {#html-404}

There is nothing stopping us sending an HTML document with a `404 Not Found` status.
The simplest thing we can do is copy the home page document and change its body.

Update **server/main.js** to replace the plain-text fallback with a complete document.

**server/main.js**
```JavaScript{hl_lines="36-54"}
import { serveDir } from "@std/http";

export function server(request) {
    const url = new URL(request.url);

    // Placeholder: this will need to change when we add middleware
    console.log(request.method, url.pathname);

    // Placeholder: this will need to change when we add a router
    if (url.pathname.startsWith("/assets/")) {
        return serveDir(request, {
            fsRoot: "public",
            urlRoot: "assets",
        });
    }
    if (url.pathname === "/") {
        // Placeholder: this will need to change when we add HTML views
        const headers = new Headers({ "content-type": "text/html" });
        const document = `
            <!DOCTYPE html>
            <html lang="en">
                <head>
                    <meta charset="utf-8">
                    <title>Bookshelves</title>
                    <link rel="stylesheet" href="/assets/styles.css">
                    <link rel="icon" href="/assets/icon.svg">
                </head>
                <body>
                    <h1>Welcome to Bookshelves</h1>
                    <p>Organise your books with bookshelves.</p>
                </body>
            </html>
        `;
        return new Response(document, { headers });
    }

    // Placeholder: this will need to change when we add HTML views
    const headers = new Headers({ "content-type": "text/html" });
    const document = `
        <!DOCTYPE html>
        <html lang="en">
            <head>
                <meta charset="utf-8">
                <title>Bookshelves</title>
                <link rel="stylesheet" href="/assets/styles.css">
                <link rel="icon" href="/assets/icon.svg">
            </head>
            <body>
                <h1>Not found</h1>
                <p>The requested resource was not found.</p>
            </body>
        </html>
    `;
    return new Response(document, { headers, status: 404 });
}
```

Only the `<h1>` and `<p>` in the body are different from the home page.
We still pass `status: 404`, and now we also send our `content-type` header.

> We copied the HTML views placeholder too, because these lines will be replaced in the same way.

Click [http://localhost:8000/books][books] again.
Now we get a proper page, with our font, our stylesheet and the icon in the tab.
Select the **books** row in the **network** panel.

{{<figure src="images/not-found-styled.webp" alt="The not-found page with the heading Not found, in the same font and spacing as the home page" caption="Our not-found page is now a proper HTML page, with our stylesheet.">}}

```http
HTTP/1.1 404 Not Found
content-type: text/html
content-length: 450
date: Wed, 28 Oct 2026 12:15:44 GMT
```

The status is still `404 Not Found`, but the `content-type` is now `text/html`.
The terminal shows the browser asking for the stylesheet and the icon, just as it does for the home page.

```plaintext {linenos=false}
GET /books
GET /assets/styles.css
[2026-10-28 12:17:08] [GET] /assets/styles.css 200
GET /assets/icon.svg
[2026-10-28 12:17:08] [GET] /assets/icon.svg 200
```

> The browser doesn't care about the status when it displays a page.
> It renders whatever HTML we send.
> The status is for the browser (and for search engines and other software) to know that this page doesn't exist.

### Spot the repetition

Look at **server/main.js** again.
Apart from two lines in the body, the two documents are identical.

This is not great.
If we want to change anything they share, such as adding a header or another stylesheet, we will have to change both.
Every new page will add another copy, and sooner or later one of them will be forgotten.

> We saw exactly this problem in the [Hello world][hello world upgrade] exercise, where the header and footer had to be copied into every HTML file.

## A layout function {#layout}

The parts of the document every page shares are the *layout*.
The part that changes from page to page is the *content*.
So let's write a function which takes the content and wraps it in the layout.

Create a folder called **views** inside the **server** folder, with a file called **layout.js**.

**server/views/layout.js**
```JavaScript
// Placeholder: this will need to change when we add flash messages, sign-in and nav highlighting
export function layout(content) {
    const document = `
        <!DOCTYPE html>
        <html lang="en">
            <head>
                <meta charset="utf-8">
                <title>Bookshelves</title>
                <link rel="stylesheet" href="/assets/styles.css">
                <link rel="icon" href="/assets/icon.svg">
            </head>
            <body>
            ${content}
            </body>
        </html>
    `;
    return document;
}
```

The function takes the `content` as an argument and puts it inside the `<body>` with `${content}`.
This is the same template literal we have been using all along, with a value dropped into it.

> The layout will need more than just the content later on, when we add flash messages, sign-in and nav highlighting.
> This line will change when we add them.

Now we can use our new function in **server/main.js**.
Each page only needs to provide its content.

**server/main.js**
```JavaScript{hl_lines="2 20-24 29-33"}
import { serveDir } from "@std/http";
import { layout } from "./views/layout.js";

export function server(request) {
    const url = new URL(request.url);

    // Placeholder: this will need to change when we add middleware
    console.log(request.method, url.pathname);

    // Placeholder: this will need to change when we add a router
    if (url.pathname.startsWith("/assets/")) {
        return serveDir(request, {
            fsRoot: "public",
            urlRoot: "assets",
        });
    }
    if (url.pathname === "/") {
        // Placeholder: this will need to change when we add HTML views
        const headers = new Headers({ "content-type": "text/html" });
        const content = `
            <h1>Welcome to Bookshelves</h1>
            <p>Organise your books with bookshelves.</p>
        `;
        return new Response(layout(content), { headers });
    }

    // Placeholder: this will need to change when we add HTML views
    const headers = new Headers({ "content-type": "text/html" });
    const content = `
        <h1>Not found</h1>
        <p>The requested resource was not found.</p>
    `;
    return new Response(layout(content), { headers, status: 404 });
}
```

We [export][export] `layout` from **server/views/layout.js** and import it into **server/main.js**.
This isn't new.
**tasks/server.js** has been importing our `server` function from **server/main.js** since [our first web server][server task].

> The path `"./views/layout.js"` is relative to **server/main.js**, so `./` means "the folder this module is in".
> In **tasks/server.js** we used `"../server/main.js"`, where `../` means "the folder above".

Refresh both [http://localhost:8000/][localhost] and [http://localhost:8000/books][books].
**Nothing should change.**
Both pages look exactly as they did, and the **network** panel shows the same statuses.
That's exactly what we want, it shows our change worked.

### Refactoring {#refactoring}

This is an example of [*refactoring*][refactoring].
We are not adding new functionality, we are reorganising the way the code is structured.
The document now exists in one place, so a change to the layout will change every page.

> A refactor should leave the behaviour exactly as it was.
> That's why we checked both pages, and their statuses, before moving on.

## Views {#views}

Our request handler in **server/main.js** still contains HTML.
Its job is to decide what to send back, it shouldn't need to know what the page says.
So let's move each page's content into a function of its own.

Add two more files to the **views** folder.

**server/views/home.js**
```JavaScript
export function homeView() {
    return `
        <section>
            <header>
                <h1>Welcome to Bookshelves</h1>
                <p>Organise your books with bookshelves.</p>
            </header>
        </section>
    `;
}
```

**server/views/notFound.js**
```JavaScript
export function notFoundView() {
    return `
        <section>
            <header>
                <h1>Not found</h1>
                <p>The requested resource was not found.</p>
            </header>
        </section>
    `;
}
```

These are *views*.
A view is just a function that returns a fragment of HTML which can be embedded into the layout.
Each one builds the content for one page, and nothing else.

> We have also wrapped each page's content in a `<section>` with its own `<header>`, which groups a heading with its introduction.
> This is the only change you might spot in the browser, the spacing between the heading and the paragraph is a little smaller.

Now update **server/main.js** to import the views and use them.

**server/main.js**
```JavaScript{hl_lines="3-4 20 22 25 27"}
import { serveDir } from "@std/http";
import { layout } from "./views/layout.js";
import { homeView } from "./views/home.js";
import { notFoundView } from "./views/notFound.js";

export function server(request) {
    const url = new URL(request.url);

    // Placeholder: this will need to change when we add middleware
    console.log(request.method, url.pathname);

    // Placeholder: this will need to change when we add a router
    if (url.pathname.startsWith("/assets/")) {
        return serveDir(request, {
            fsRoot: "public",
            urlRoot: "assets",
        });
    }
    if (url.pathname === "/") {
        // Placeholder: this will need to change when we add controllers
        const headers = new Headers({ "content-type": "text/html" });
        return new Response(layout(homeView()), { headers });
    }

    // Placeholder: this will need to change when we add controllers
    const headers = new Headers({ "content-type": "text/html" });
    return new Response(layout(notFoundView()), { headers, status: 404 });
}
```

Our request handler is much shorter.
It still decides which page to send, sets the headers and the status, and builds the `Response`.
But it no longer contains any HTML.
For each page it calls a view to get the content, and passes that to `layout` to get the whole document.

> The HTML views placeholders have done their job and are gone.
> Setting the headers and building the `Response` is still the request handler's job, but it won't be for long.
> These lines are now marked for when we add *controllers*, which we will meet soon.

Refresh both pages again.
Apart from the slightly tighter spacing, they should look the same as before.

### Model, view, controller {#mvc}

Views are the first part of a well-known pattern for organising applications, called [*model-view-controller*][mvc] or [*MVC*][mvc wikipedia].

- **Models** look after our data. We don't have any data yet.
- **Views** present information to the user. In our app, a view is a function that returns HTML.
- **Controllers** handle a request, using models and views to build the response. For now, our request handler is doing this job.

Each part has one job, so each part can change without breaking the others.
From now on, every page in our app will have its own view.

> Notice how the code is organised.
> A view is a function, and related functions are grouped into modules.
> All our views live in the **views** folder, so we always know where to look for the HTML.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="8-12"}
my-app/
├── .vscode/
│   └── settings.json
├── public/
│   ├── icon.svg
│   └── styles.css
├── server/
│   ├── views/
│   │   ├── home.js
│   │   ├── layout.js
│   │   └── notFound.js
│   └── main.js
├── tasks/
│   └── server.js
├── deno.json
└── deno.lock
```

## A header and footer {#header-footer}

Now for the payoff.
Every page is built by the same `layout` function, so we can add a header and footer to every page by changing that one function.

Update **server/views/layout.js**.

**server/views/layout.js**
```JavaScript{hl_lines="9 14-20 22-27"}
// Placeholder: this will need to change when we add flash messages, sign-in and nav highlighting
export function layout(content) {
    const document = `
        <!DOCTYPE html>
        <html lang="en">
            <head>
                <meta charset="utf-8">
                <title>Bookshelves</title>
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
    return document;
}
```

We have added three things around our `${content}`.

1. A `<header>` with a link to the home page, showing our icon and the name of the site, plus a `<nav>` with a link to the **books** page.
1. A `<main>` element to hold the content from each view.
1. A `<footer>` with a short copyright notice.

> The icon `<img>` has an empty [`alt`][alt] attribute because the word **Bookshelves** is right next to it.
> An empty `alt` tells screen readers the image is decorative, so they don't read the name twice.

> The new [viewport][viewport] `<meta>` element tells mobile browsers to use the real width of the screen, rather than pretending to be a wide desktop screen.
> Without it, our header would be tiny on a phone.

### Styling the header and footer {#styles}

Replace the contents of **public/styles.css** with the following.

**public/styles.css**
```css {hl_lines="1 4-5 18-29 36-77"}
/* The accent is a custom property, so it is set once and used everywhere */
:root {
    color-scheme: light dark; /* light or dark page colours, from the OS setting */
    --hue: 20; /* change the hue (0 to 360) to pick any accent colour */
    --highlight: oklch(62% 0.2 var(--hue));
}

body {
    margin: 0;
    font-family: system-ui;
    font-weight: 300;
}

h1 {
    font-weight: 400;
}

/* Links keep the text colour, with an accent underline */
a {
    color: inherit;
    text-decoration-color: var(--highlight);
    text-decoration-thickness: 2px;
}

/* The header and footer are dark panels with light text */
.secondary {
    background: #333;
    color: white;
}

/* Padding keeps everything off the edges of the window */
body > * {
    padding: 0.5rem 1rem;
}

/* Three rows: header and footer take what they need, main takes the rest */
body {
    display: grid;
    grid-template-rows: auto 1fr auto;
    min-block-size: 100vh;

    & > header,
    & > footer {
        display: flex;
        flex-wrap: wrap;
        justify-content: space-between;
        align-items: center;
        gap: 1rem;
    }
}

/* The logo and site name sit side by side */
.brand {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    font-size: 1.5em;
    img {
        block-size: 1em;
    }
}

/* The icon is drawn black in light mode, so invert it to show on the dark header */
@media (prefers-color-scheme: light) {
    .brand img {
        filter: invert(1);
    }
}

/* Links in a nav sit in a centred row */
nav {
    display: flex;
    flex-wrap: wrap;
    justify-content: center;
    align-items: center;
    gap: 0.5rem;
}
```

Most of this is revision.
The `body` grid and the flex `header` and `footer` work just like the layout we built in the [Hello world][hello world upgrade] exercise.
The `secondary` and `brand` class names in our layout are what these rules select.

A couple of things are new.

- `--hue` and `--highlight` are [*custom properties*][custom properties], values we set once and use with `var()`. Here, the accent colour has just one job, the underline on our links.
- Our icon is drawn black in light mode and white in dark mode, to suit the page. But our header is always dark, so in light mode a black icon would disappear. The [`prefers-color-scheme`][prefers-color-scheme] media query applies `filter: invert(1)` only in light mode, which turns the black icon white. See [*filter*][filter].

Click [http://localhost:8000/][localhost] and then [http://localhost:8000/books][books].
**Both pages have changed**, and we only changed one function.
They share the same header and footer, and only the content in between is different.

{{<figure src="images/shared-layout.webp" alt="The home page and the not-found page, one above the other, both with the same dark header and footer" caption="The home page and the not-found page now share a header and footer.">}}

Click the **books** link in the header.
We get our not-found page, with a `404 Not Found` status in the **network** panel.
That's correct, we haven't built the books page yet.
Click the icon or the word **Bookshelves** to go back home.

A pretty nice result.

> At the end of [Hello world][hello world upgrade] we said this problem would be solved when we construct our HTML dynamically on the server using JavaScript.
> Now it has been.

## Conclusion

We have turned our plain-text 404 into a page which looks like the rest of our site, with a `404 Not Found` status.
We refactored the repeated document into a `layout` function and moved each page's content into a view.
This is the *view* part of model-view-controller: a view is a function which returns HTML, and our views live together in the **views** folder.
Because every page uses the same layout, we added a header and footer to every page by changing one function.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="6 8-12"}
my-app/
├── .vscode/
│   └── settings.json
├── public/
│   ├── icon.svg
│   └── styles.css
├── server/
│   ├── views/
│   │   ├── home.js
│   │   ├── layout.js
│   │   └── notFound.js
│   └── main.js
├── tasks/
│   └── server.js
├── deno.json
└── deno.lock
```

## Challenge: Experiment with the layout

- Change `--hue` in **public/styles.css** to pick a different accent colour, and watch the underline on every page change.
- Switch your operating system between light and dark mode and look at the icon in the header. Then, in the **Elements** panel, untick the `filter` rule to see why it is there.
- Add a second link to the `<nav>` in **server/views/layout.js**, for example to `/about`. It appears on every page. Where does it take you, and what status do you get?

> Make sure **public/styles.css** and **server/views/layout.js** are back to the versions above before moving on.

### An about page

Try the [about page challenge][about challenge] from the previous exercise again, now that we have views.

- Create **server/views/about.js** with an `aboutView`, based on the not-found view.
- Import it into **server/main.js** and add another `if` which sends `layout(aboutView())`, just like the home page.
- Check that [http://localhost:8000/about][about] is `200 OK` in the **network** panel.
- If you like, add a link to it in the `<nav>` in **server/views/layout.js**. Leave the link out of your count, so that it compares with the other exercises.
- Count the files you touched and the lines you added or changed.

Compare with your previous count.
Think about the number of lines, the effort involved and the quality of the code, and draw your own conclusions.

> **Remove your about page before moving on.**
> Delete **server/views/about.js** and undo your changes to **server/main.js** (and **server/views/layout.js**, if you added a link), so that your code matches the starting point of the next exercise.

Finally, something to think about.
Our request handler still has an `if` for each page, and repeats the same `headers` line for each one.
What would **server/main.js** look like with twenty pages?

[previous exercise]: deno-server/missing-pages/
[localhost]: http://localhost:8000/
[books]: http://localhost:8000/books
[hello world upgrade]: welcome/hello-world/#upgrading-the-website
[export]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/export
[server task]: deno-server/deno-serve/#the-server-task
[refactoring]: https://en.wikipedia.org/wiki/Code_refactoring
[mvc]: https://developer.mozilla.org/en-US/docs/Glossary/MVC
[mvc wikipedia]: https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller
[alt]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/img#alt
[viewport]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/meta/name/viewport
[custom properties]: https://developer.mozilla.org/en-US/docs/Web/CSS/Guides/Cascading_variables/Using_custom_properties
[prefers-color-scheme]: https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/At-rules/@media/prefers-color-scheme
[filter]: https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/filter
[about challenge]: deno-server/missing-pages/#about-challenge
[about]: http://localhost:8000/about

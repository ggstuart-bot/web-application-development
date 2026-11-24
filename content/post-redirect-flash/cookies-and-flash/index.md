---
type: exercise
title: Cookies and flash
weight: 2
author: Dr Graeme Stuart
---

In this exercise we will tell the user what just happened.
After a shelf is created, the next page will show a short message, **Shelf 'poetry' created**, that slides out of the way.
To carry the message across the redirect, we will use our first cookie.

<!--more-->

> You should have the new shelf form, `createShelfController` and the `get` and `post` shortcuts from the [previous exercise][previous exercise].
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

When we create a shelf, we land on its page with no sign that anything happened.
A short message would help, but the page we land on is fetched by a brand new request, after the redirect.
By the end of this exercise you should be able to:

- Share one `Headers` object between everything that handles a request
- Set a cookie with the `Set-Cookie` header, and explain what `HttpOnly` does
- Read a cookie from a request, and delete it with the response
- Carry a *flash message* across a redirect, and show it on the next page only
- Make a status message that screen readers announce and keyboards can reach
- Animate an element with `@keyframes`, unless the visitor prefers reduced motion

## One set of headers {#ctx-headers}

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Our flash message will travel in a cookie, and a cookie is set with a response header.
At the moment, `render` and `redirect` each make their own `Headers` object, so nothing else can add a header to the response.
We will make one `Headers` object for each request, and keep it on `ctx`.

Update `server` in **server/main.js**.

**server/main.js**
```JavaScript {hl_lines="28-29"}
import { homeController } from "./controllers/home.js";
import { bookController, booksController } from "./controllers/books.js";
import {
    createShelfController,
    shelfController,
    shelvesController,
} from "./controllers/shelves.js";
import { staticController } from "./controllers/static.js";
import { Router } from "./tools/router.js";
import { withLogs } from "./middleware/logging.js";
import { withErrors } from "./middleware/errors.js";

const app = new Router();

app.use(withLogs);
app.use(withErrors);

app.get("/assets/*", staticController);
app.get("/", homeController);
app.get("/books", booksController);
app.get("/books/:bookId", bookController);
app.get("/shelves", shelvesController);
app.post("/shelves", createShelfController);
app.get("/shelves/:shelfId", shelfController);

export function server(request) {
    const url = new URL(request.url);
    const headers = new Headers();
    const ctx = { request, url, data: {}, headers };
    return app.handle(ctx);
}
```

Every request now gets an empty [`Headers`][headers] object on `ctx.headers`.

Update **server/tools/render.js** to use it.

**server/tools/render.js**
```JavaScript {hl_lines="5-6"}
import { layout } from "../views/layout.js";

// Placeholder: this will need to change when we add flash messages and sign-in
export function render(view, ctx) {
    const { headers, status } = ctx;
    headers.set("content-type", "text/html");
    const content = view(ctx);
    const html = layout(content, ctx);
    return new Response(html, { headers, status });
}
```

`render` takes the headers from `ctx`, and [`set`][headers set] adds the content type to them.

> The placeholder says that `render` will need to change when we add flash messages.
> We will change it later in this exercise.

Update **server/tools/redirect.js** in the same way.

**server/tools/redirect.js**
```JavaScript {hl_lines="1-3"}
export function redirect(location, ctx) {
    const { headers } = ctx;
    headers.set("location", location);
    return new Response(null, { headers, status: 303 });
}
```

`redirect` needs `ctx` to reach the headers, so it takes `ctx` as well as the location.
Update `createShelfController` in **server/controllers/shelves.js** to pass it.

**server/controllers/shelves.js**
```JavaScript {hl_lines="19"}
import { createShelf, getShelfById, getShelves } from "../models/shelves.js";
import { addThumbnails, getBooksByShelfId } from "../models/shelfBooks.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { NotFoundError } from "../tools/errors.js";
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
    const shelf = createShelf({ title: formData.get("title") });
    return redirect(`/shelves/${shelf.id}`, ctx);
}

export function shelfController(ctx) {
    const { params: { shelfId } } = ctx;
    const shelf = getShelfById(Number(shelfId));
    if (!shelf) {
        throw new NotFoundError(`Shelf ${shelfId} not found`);
    }
    ctx.data.shelf = shelf;
    ctx.data.title = shelf.title;
    ctx.data.books = getBooksByShelfId(shelf.id);
    return render(shelfView, ctx);
}
```

Nothing has changed for the user.
Create a shelf from [http://localhost:8000/shelves][shelves], and you still land on its page.
The difference is that anything that runs during a request can now add a header to the response, and our flash cookie will be the first thing that does.

## A flash cookie {#set-flash}

HTTP has no memory.
Each request is answered on its own, and the server doesn't know that the `GET` after a redirect came from the same browser as the `POST` before it.
If the `POST` wants to leave a message for the next page, the browser has to carry it.

That's what a [cookie][cookies] is for.

1. A response includes a [`Set-Cookie`][set-cookie] header, with a name and a value.
2. The browser stores the cookie.
3. From then on, the browser sends it back to the same site in the `Cookie` header of every request.

The `@std/http` package that gives us `serveDir` also has a [cookie module][std cookie], with functions for setting, reading and deleting cookies.
Create a new file, **server/tools/flash.js**.

**server/tools/flash.js**
```JavaScript {hl_lines="1-12"}
import { setCookie } from "@std/http/cookie";

export function setFlash(ctx) {
    const { headers, flash } = ctx;
    if (!flash) return;
    setCookie(headers, {
        name: "flash",
        value: encodeURIComponent(flash),
        path: "/",
        httpOnly: true,
    });
}
```

`setFlash` sends the message in `ctx.flash` as a cookie, using [`setCookie`][set cookie fn] from `@std/http/cookie`.
If there's no message, it does nothing.

- The cookie's `name` is `flash`.
- Cookie values can't hold every character that a shelf title can, such as a semicolon. [`encodeURIComponent`][encode] replaces those characters with codes like `%20`, the same codes we see in URLs.
- `path: "/"` means the browser will send the cookie back with a request for any page on our site, not just the one that set it.
- `httpOnly: true` adds [`HttpOnly`][http only], which hides the cookie from any JavaScript running in the page. Only the server will ever read it.

`redirect` should send the flash cookie, if there is a message.
Update **server/tools/redirect.js**.

**server/tools/redirect.js**
```JavaScript {hl_lines="1-2 5"}
import { setFlash } from "./flash.js";

export function redirect(location, ctx) {
    const { headers } = ctx;
    setFlash(ctx);
    headers.set("location", location);
    return new Response(null, { headers, status: 303 });
}
```

`setFlash` adds the cookie to the same `ctx.headers` that the redirect sends, which is why we shared them.

A controller leaves a message by putting it on `ctx.flash`.
Update `createShelfController` in **server/controllers/shelves.js**.

**server/controllers/shelves.js**
```JavaScript {hl_lines="19"}
import { createShelf, getShelfById, getShelves } from "../models/shelves.js";
import { addThumbnails, getBooksByShelfId } from "../models/shelfBooks.js";
import { redirect } from "../tools/redirect.js";
import { render } from "../tools/render.js";
import { NotFoundError } from "../tools/errors.js";
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
    const shelf = createShelf({ title: formData.get("title") });
    ctx.flash = `Shelf '${shelf.title}' created`;
    return redirect(`/shelves/${shelf.id}`, ctx);
}

export function shelfController(ctx) {
    const { params: { shelfId } } = ctx;
    const shelf = getShelfById(Number(shelfId));
    if (!shelf) {
        throw new NotFoundError(`Shelf ${shelfId} not found`);
    }
    ctx.data.shelf = shelf;
    ctx.data.title = shelf.title;
    ctx.data.books = getBooksByShelfId(shelf.id);
    return render(shelfView, ctx);
}
```

`createShelf` gave us the new shelf, so the message can include its `title`.

Open [http://localhost:8000/shelves][shelves] with the **network** panel open, with **preserve log** ticked, and create a shelf called **poetry**.
Nothing looks different, but click the `POST` request and look at its response headers.

```plaintext {linenos=false}
location: /shelves/4
set-cookie: flash=Shelf%20'poetry'%20created; HttpOnly; Path=/
```

The spaces have been encoded as `%20`.
Now click the `GET` request for `/shelves/4`, and look at its *request* headers.
The browser sent the cookie straight back.

```plaintext {linenos=false}
cookie: flash=Shelf%20'poetry'%20created
```

You can see the cookie the browser is storing too.
Open the **application** panel (**storage** in Firefox), and choose **cookies**, then `http://localhost:8000`.
The `flash` cookie is there, with a tick in its **HttpOnly** column.

Nothing reads the cookie yet, so the message never shows up on the page.
Nothing deletes it either, so the browser sends it with every request.
Click around the site and check the request headers in the **network** panel.

## Showing the message {#get-flash}

The page we land on should read the cookie, show the message and delete the cookie, so that the message only shows once.
Add `getFlash` to **server/tools/flash.js**.

**server/tools/flash.js**
```JavaScript {hl_lines="1 13-21"}
import { deleteCookie, getCookies, setCookie } from "@std/http/cookie";

export function setFlash(ctx) {
    const { headers, flash } = ctx;
    if (!flash) return;
    setCookie(headers, {
        name: "flash",
        value: encodeURIComponent(flash),
        path: "/",
        httpOnly: true,
    });
}

export function getFlash(ctx) {
    const { request, headers } = ctx;
    const { flash } = getCookies(request.headers);
    if (flash) {
        ctx.data.flash = decodeURIComponent(flash);
        deleteCookie(headers, "flash", { path: "/" });
    }
}
```

- [`getCookies`][get cookies] reads the `Cookie` header of a request, and gives us an object with a property for each cookie. We only want `flash`.
- If there is a flash cookie, [`decodeURIComponent`][decode] turns the `%20` codes back into the original characters, and we put the message on `ctx.data.flash`, with the rest of the page's data.
- [`deleteCookie`][delete cookie] adds a `Set-Cookie` header to our response that tells the browser to throw the cookie away. The `path` must match the one we set it with.

Every page is built by `render`, so that's where we read the flash.
Update **server/tools/render.js**.

**server/tools/render.js**
```JavaScript {hl_lines="1 7"}
import { getFlash } from "./flash.js";
import { layout } from "../views/layout.js";

export function render(view, ctx) {
    const { headers, status } = ctx;
    headers.set("content-type", "text/html");
    getFlash(ctx);
    const content = view(ctx);
    const html = layout(content, ctx);
    return new Response(html, { headers, status });
}
```

`render` calls `getFlash` before it builds the page, so the message is on `ctx.data` when the view and layout need it.
The placeholder has gone, because `render` is finished.

The message belongs on every page, so the layout will show it.
Update **server/views/layout.js**.

**server/views/layout.js**
```JavaScript {hl_lines="3 26 40-43"}
import { html } from "../tools/html.js";

// Placeholder: this will need to change when we add sign-in
export function layout(content, ctx) {
    const { data, url } = ctx;
    const title = data.title ? `${data.title} | Bookshelves` : "Bookshelves";

    return html`
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
                        ${navLink("/shelves", "shelves", url.pathname)}
                    </nav>
                </header>
                ${flashMessage(data.flash)}
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

function flashMessage(flash) {
    return flash ? html`<p id="flash" role="status" tabindex="0">${flash}</p>` : "";
}

// A header link, marked as the current page when it matches the path exactly
function navLink(href, content, path) {
    return html`<a href="${href}" aria-current="${
        href === path ? "page" : "false"
    }">${content}</a>`;
}
```

- `flashMessage` gives us a paragraph with the message, or an empty string when there isn't one, using the [conditional operator][conditional operator].
- The message goes in through our `html` helper, so a shelf title with tags in it is escaped, just like everywhere else.
- [`role="status"`][status role] tells screen readers that this is a status message, so they announce it without the user having to find it.
- [`tabindex="0"`][tabindex] lets a keyboard user reach the message with the **Tab** key, although it's not a link or a form control.

> The placeholder still says the layout will change when we add sign-in.

Go to [http://localhost:8000/shelves][shelves] and create a shelf called **poetry**.
You land on its page, with **Shelf 'poetry' created** just under the header.

Look at the `GET` request for the shelf in the **network** panel.
Its response deletes the cookie.

```plaintext {linenos=false}
set-cookie: flash=; Path=/; Expires=Thu, 01 Jan 1970 00:00:00 GMT
```

There's no way to delete a cookie directly, so `deleteCookie` sets it again, empty, with an expiry date long in the past.
The browser throws it away.

Refresh the page.
The message has gone, and the **application** panel shows no `flash` cookie.
A flash message is shown once, on the very next page.

## Logging the flash {#flash-logs}

A flash message crosses two requests.
It would help to see it in the terminal on both.
Update **server/middleware/logging.js**.

**server/middleware/logging.js**
```JavaScript {hl_lines="3 9-10"}
import { UnexpectedError } from "../tools/errors.js";

// Placeholder: this will need to change when we add sessions
export async function withLogs(ctx, next) {
    const { url, request } = ctx;
    console.log("\n============================");
    console.log(`${request.method} ${url.pathname}`);
    const response = await next(ctx);
    if (ctx.flash) console.log("[flash sent]", ctx.flash);
    if (ctx.data.flash) console.log("[flash received]", ctx.data.flash);
    if (ctx.error instanceof UnexpectedError) console.log("[error]", ctx.error);
    else if (ctx.error) console.log("[error]", ctx.error.message);
    console.log("RESPONSE", response.status);
    console.log("============================");

    return response;
}
```

- `ctx.flash` is the message a controller set, which is sent with the response.
- `ctx.data.flash` is the message `getFlash` read from the request.

Both are logged after `await next(ctx)`, because they are only set once the controller has run.

> The placeholder still says the logging will change when we add sessions.

Create a shelf called **poetry**, and watch the terminal.

```plaintext {linenos=false}

============================
POST /shelves
[flash sent] Shelf 'poetry' created
RESPONSE 303
============================

============================
GET /shelves/4
[flash received] Shelf 'poetry' created
RESPONSE 200
============================
```

We can follow the message through the redirect: sent with the `303`, and received by the next page.

## Styling the flash {#flash-styles}

The message is just a paragraph under the header, and it pushes the page down.
We will make it stand out, keep it out of the way, and let it slide off to the side after a few seconds.

First, add `#flash` to the rule that fills [the current nav link][nav styles], near the top of **public/styles.css**, and add a `:focus-visible` rule after it.

**public/styles.css**
```css {linenostart="29", hl_lines="1-2 8-12"}
/* The flash message and the current page's nav link are filled with the accent */
#flash,
nav a[aria-current="page"] {
    background: var(--highlight);
    color: contrast-color(var(--highlight)); /* black or white, whichever is easier to read */
}

/* Anything you can tab to gets an accent outline */
:focus-visible {
    outline: 2px solid var(--highlight);
    outline-offset: 2px;
}
```

- The flash message is filled with the accent colour, just like the current page's nav link. The comment now mentions both.
- The [`:focus-visible`][focus visible] pseudo-class matches whatever has the keyboard focus, when the browser thinks the user should see where it is. The flash is the first thing we can tab to that isn't a link or a form control, so now anything we can tab to gets an accent outline.

Then add these rules after the `form.secondary` rule.

**public/styles.css**
```css {linenostart="213", hl_lines="1-24"}
/* Fixed, so the page never moves. Hover or focus brings it back */
#flash {
    position: fixed;
    inset-block-end: 4rem;
    inset-inline-end: 0;
    margin: 0;
    padding: 0.5rem 1rem;
    /* Only animate when the visitor hasn't asked for reduced motion */
    @media (prefers-reduced-motion: no-preference) {
        animation: slide-away 0.5s 3s forwards; /* wait 3s, slide for 0.5s, then stay put */
    }
    /* Stopping the animation brings the message back */
    &:hover,
    &:focus {
        animation: none;
    }
}

/* After 3 seconds it slides right, leaving a sliver showing */
@keyframes slide-away {
    to {
        translate: calc(100% - 1rem);
    }
}
```

- [`position: fixed`][position] takes the message out of the page and fixes it to the window. It's `4rem` up from the bottom and right against the right-hand edge. It no longer pushes the page down, and it stays put when we scroll.
- [`@keyframes`][keyframes] defines an animation called `slide-away`. It only has a `to` state, so it moves from wherever the message starts to `translate: calc(100% - 1rem)`, which slides it to the right by its own width, less `1rem`. A sliver is left showing.
- [`animation`][animation] runs `slide-away` on the message. It waits `3s`, slides for `0.5s`, and `forwards` keeps it where the animation ends.
- The animation is inside a [`prefers-reduced-motion`][reduced motion] media query, so it only runs for visitors who haven't asked their system for less motion.
- Hovering over the message, or focusing it, sets `animation: none`. Without the animation the message goes back to where it started, so the user can read it again.

Create a shelf called **poetry**.
The message appears near the bottom right, filled with the accent.

{{<figure src="images/flash.webp" alt="The poetry shelf page, with Shelf 'poetry' created in a red box fixed near the bottom right of the window" caption="The flash message appears near the bottom right.">}}

After three seconds it slides to the right, leaving a sliver showing.
Hover over the sliver, and the message comes back.
Press **Tab** until the message has the focus, and it comes back with an accent outline.

{{<figure src="images/flash-away.webp" alt="The same page a few seconds later, with only a red sliver of the message showing at the right-hand edge of the window" caption="After three seconds, only a sliver is left.">}}

## Conclusion

A redirect starts a new request, so a message for the next page has to travel in a cookie.
`createShelfController` puts its message on `ctx.flash`, and `redirect` sends it as an `HttpOnly` cookie, on the response headers that every part of a request now shares.
On the next page, `render` reads the cookie, deletes it, and the layout shows the message once, in a paragraph that screen readers announce and keyboards can reach.
`withLogs` shows the message on both requests, and our stylesheet fixes it to the window and slides it away.

For now, only creating a shelf leaves a message.
Every action we add from now on can leave one too, just by setting `ctx.flash` before it redirects.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="30"}
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
│   │   ├── shelves.js
│   │   └── static.js
│   ├── middleware/
│   │   ├── errors.js
│   │   └── logging.js
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
│   │   └── router.js
│   ├── views/
│   │   ├── partials/
│   │   │   ├── bookCard.js
│   │   │   ├── cardGrid.js
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

## Challenge: Following a cookie

These are questions to investigate, not changes to keep.

- Create a shelf called `<em>poetry</em>`. How does the flash message show the tags? Which part of our code makes that happen?
- In the developer tools, open the **rendering** panel (in Chrome, it's under **more tools**) and set **prefers-reduced-motion** to **reduce**. Create a shelf. What changes?
- Our `getFlash` deletes the cookie whenever it shows the message. What would happen if it didn't? Comment out the `deleteCookie` line, create a shelf and click around the site.
- Create a shelf called `poetry; extra=yes`, with the **network** panel open, and look at the `set-cookie` header. Then take `encodeURIComponent` out of `setFlash` and create a shelf called **poetry**. What do you see, what does the terminal say, and was the shelf created?

<details>
<summary>Why we encode the value</summary>

A `Set-Cookie` header uses semicolons to separate the cookie from its settings, such as `HttpOnly` and `Path=/`, and the rules for cookie values don't allow spaces, quotes, commas or semicolons.
With `encodeURIComponent`, the header reads `flash=Shelf%20'poetry%3B%20extra%3Dyes'%20created; HttpOnly; Path=/`, so the whole message stays in the value, and `decodeURIComponent` turns it back on the next page.

Without it, `setCookie` refuses the value and throws an error, because of the space in **Shelf 'poetry' created**.
`withErrors` catches it, so you get our **Server error** page, and the terminal shows the error with its cause, that the cookie `cannot contain character ' '`.
The shelf *was* created, because `createShelf` ran before `redirect`.
Refresh the shelves page to see it.

</details>

> If you tried any changes, undo them, and run `deno task db:reset`, so that your code and data match the starting point of the next exercise.

[previous exercise]: post-redirect-flash/form-that-posts/
[shelves]: http://localhost:8000/shelves
[headers]: https://developer.mozilla.org/en-US/docs/Web/API/Headers
[headers set]: https://developer.mozilla.org/en-US/docs/Web/API/Headers/set
[cookies]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Cookies
[set-cookie]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie
[std cookie]: https://jsr.io/@std/http/doc/cookie
[set cookie fn]: https://jsr.io/@std/http/doc/cookie/~/setCookie
[get cookies]: https://jsr.io/@std/http/doc/cookie/~/getCookies
[delete cookie]: https://jsr.io/@std/http/doc/cookie/~/deleteCookie
[encode]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/encodeURIComponent
[decode]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/decodeURIComponent
[http only]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie#httponly
[conditional operator]: user-input-and-security/search/#title
[status role]: https://developer.mozilla.org/en-US/docs/Web/Accessibility/ARIA/Reference/Roles/status_role
[tabindex]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Global_attributes/tabindex
[nav styles]: router-class/one-context/#nav-styles
[focus visible]: https://developer.mozilla.org/en-US/docs/Web/CSS/:focus-visible
[position]: https://developer.mozilla.org/en-US/docs/Web/CSS/position
[keyframes]: https://developer.mozilla.org/en-US/docs/Web/CSS/@keyframes
[animation]: https://developer.mozilla.org/en-US/docs/Web/CSS/animation
[reduced motion]: https://developer.mozilla.org/en-US/docs/Web/CSS/@media/prefers-reduced-motion

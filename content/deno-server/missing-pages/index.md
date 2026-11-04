---
type: exercise
title: Handling missing pages
weight: 5
author: Dr Graeme Stuart
---

In this exercise we will stop our server pretending that every page exists.
We will send the home page only for `/` and a `404 Not Found` response for anything else.

<!--more-->

> You should have a working server which serves an icon and a stylesheet from the [previous exercise][previous exercise].
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

This is a short exercise with one problem to solve: our server answers every path with the home page.
We will start by exploring all the responses our server can generate, and then fix the problem.
By the end of this exercise you should be able to:

- Use the **network** panel to explore the responses our server generates
- Use an `if` statement and `===` to send a response for one path only
- Explain why a function that reaches its end without returning causes a `500 Internal Server Error`
- Return a `404 Not Found` response for any path we don't recognise

## Setup

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Open [http://localhost:8000/][localhost] with the developer tools (**F12**) open on the **network** panel.
You should see our home page with its icon and stylesheet.

## Exploring our responses

Before we change anything, let's explore all the responses our server can currently generate.
Keep the **network** panel open and check the status and `content-type` of each request.

### Pages

Our app will eventually have a page listing all the books.
It doesn't exist yet, so let's see what happens if we visit it.

Click [http://localhost:8000/books][books].

We get our home page.
Select the **books** row in the **network** panel.
The status is `200 OK`.

```http
HTTP/1.1 200 OK
content-type: text/html
content-length: 463
date: Wed, 28 Oct 2026 09:21:50 GMT
```

Try [http://localhost:8000/anything/you/like][anything] too.
Same page, same status.
In fact, every path that doesn't start with `/assets/` gets our home page, exactly as [http://localhost:8000/][localhost] does.

### Static files

Paths that start with `/assets/` are different, they are passed to `serveDir`.

- [http://localhost:8000/assets/icon.svg][assets icon] is `200 OK` with a `content-type` of `image/svg+xml`.
- [http://localhost:8000/assets/styles.css][assets styles] is `200 OK` with a `content-type` of `text/css; charset=UTF-8`.

`serveDir` finds each file in the **public** folder and sends it with the right `content-type`.
It will deliver any file we put in that folder.

### A missing file

Now ask for a file that isn't there.
Click [http://localhost:8000/assets/missing.css][missing].

The browser shows **Not Found** and the **network** panel shows a `404 Not Found` status.

{{<figure src="images/servedir-not-found.webp" alt="A blank page with the words Not Found in small plain text" caption="`serveDir` sends a plain-text Not Found for a missing file.">}}

```http
HTTP/1.1 404 Not Found
Content-Type: text/plain;charset=UTF-8
content-length: 9
date: Wed, 28 Oct 2026 09:21:51 GMT
```

The terminal explains what happened.

```plaintext {linenos=false}
GET /assets/missing.css
No such file or directory (os error 2): stat 'public/missing.css'
[2026-10-28 09:21:51] [GET] /assets/missing.css 404
```

`serveDir` looked in the **public** folder for **missing.css** and didn't find it.
So it told the browser, with a [`404 Not Found`][404] response.
This is exactly the right thing to do.

### Missing pages

Our application pages should do the same.
At the moment our server is telling the browser (and anyone else who asks) that every path is a real page.
That isn't true and it's confusing for our users.
It also means a typo in a link would never be noticed.

So the overall picture is this.

1. We serve the same home page for every path except those under `/assets/`.
1. `serveDir` delivers any requested file from the **public** folder.
1. Requests for missing files get a `404 Not Found` response.
1. Requests for missing pages should also get a `404 Not Found` response, rather than the home page.

The rest of this exercise fixes the last point.

## Only the home page at `/`

We want to send the home page only when the path is exactly `/`.
Update **server/main.js** to wrap the home page code in an `if` statement.

**server/main.js**
```JavaScript{hl_lines="16 35"}
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
}
```

Only lines 16 and 35 are new.
Everything between them is just indented by one more level, because it is now inside the `if` block.

> The HTML views placeholder has moved inside the `if`, along with the code it marks.
> The router placeholder now covers both `if` statements.
> When we add a *router*, it will decide which code handles each path and both of them will be replaced.

> We compare the strings with [`===`][strict equality] (*strict equality*), just as we used `!==` when [serving the whole folder][serve dir].
> It is only `true` if both values have the same type and the same value.
> The looser [`==`][equality] converts the values to the same type first, so `"1" == 1` is `true`.
> There are a few places where `==` is useful.
> For example, `value == null` is `true` when `value` is either `null` or `undefined`.
> Otherwise, we will stick to `===`.

The home page at [http://localhost:8000/][localhost] should still work.
Now click [http://localhost:8000/books][books] again.

The browser shows **Internal Server Error**.
In the **network** panel, the status is `500 Internal Server Error`.

```http
HTTP/1.1 500 Internal Server Error
content-length: 21
date: Tue, 27 Oct 2026 17:47:43 GMT
```

The terminal has an error message.

```plaintext {linenos=false}
GET /books
TypeError: Return value from serve handler must be a response or a promise resolving to a response
    at finishOrReturnNative (ext:deno_http/00_serve.ts:1:11743)
    at finishOrReturnMaybePromise (ext:deno_http/00_serve.ts:1:13573)
    at nativeMapped (ext:deno_http/00_serve.ts:1:13832)
    at nativeDispatch (ext:deno_http/00_serve.ts:1:17522)
```

> The `at ...` lines are deno's own code, so they won't help us much.
> The first line is the important one.

### What went wrong? {#implicit-undefined}

Neither `if` matched `/books`, so our function reached its last line without returning anything.
A JavaScript function that ends without a `return` statement returns [`undefined`][undefined].
`Deno.serve` needs a `Response` (or a promise of one) and `undefined` isn't one, so it complains.

Deno still has to send *something* back to the browser.
So it sends a [`500 Internal Server Error`][500] response, which means the server failed because of a problem in its own code.
It isn't the browser's fault, it's ours.

**This is a bug in our code, not something to catch and hide.**
The fix is to make sure our function always returns a response.

## A fallback response {#not-found}

If the request didn't match any of our `if` statements, the path is one we don't recognise.
Add a final `return` statement to send a `404 Not Found` response.

**server/main.js**
```JavaScript{hl_lines="36"}
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
    return new Response("Not found", { status: 404 });
}
```

> The router placeholder covers this line too.
> When we add a router, it will also decide what to do when nothing matches.

This is a *fallback*.
Code only reaches it if every `if` above was skipped, so every path now gets a response.

Click [http://localhost:8000/books][books] again.
This time the browser shows **Not found** and the **network** panel shows a `404 Not Found` status.

```http
HTTP/1.1 404 Not Found
Content-Type: text/plain;charset=UTF-8
content-length: 9
date: Tue, 27 Oct 2026 17:47:46 GMT
```

> We didn't set a `content-type` header, so the response constructor added `text/plain` for us, as it did for our first hello world.

Try a few more paths and check the status of each in the **network** panel.
We should now have both kinds of 404.

- [http://localhost:8000/][localhost] should still be `200 OK`.
- [http://localhost:8000/anything/you/like][anything] should be `404 Not Found`.
- [http://localhost:8000/favicon.ico][favicon.ico] should be `404 Not Found` too.
- [http://localhost:8000/assets/missing.css][missing] should still be `404 Not Found`, from `serveDir`.

The `/favicon.ico` 404 is fine.
Our document links to `/assets/icon.svg`, so the browser doesn't need to ask for `/favicon.ico` any more.

> Look closely at the text of the two 404 responses.
> The capital **F** in **Not Found** is a clue: that one comes from `serveDir`, and **Not found** comes from our fallback.

## Conclusion

We have made sure our server only sends the home page for `/`.
We saw that a request handler which reaches its end without returning causes a `500 Internal Server Error`, and that this was our bug.
We fixed it with a fallback that returns a `404 Not Found` response for any path we don't recognise.

Our project still looks like this.
Only **server/main.js** has changed.

```plaintext {linenos=false}
my-app/
├── .vscode/
│   └── settings.json
├── public/
│   ├── icon.svg
│   └── styles.css
├── server/
│   └── main.js
├── tasks/
│   └── server.js
├── deno.json
└── deno.lock
```

## Challenge: Experiment with status codes

- Visit [http://localhost:8000/?search=deno][search]. Which `if` handles it, and why? Hint: log `url.search` as well as `url.pathname`.
- Temporarily add a second page at `/temporary` with another `if`, and check its status in the **network** panel.
  Now visit [http://localhost:8000/temporary/][temporary slash], with a slash at the end.
  Why does it get a 404?

> Make sure **server/main.js** is back to the version above before moving on.

### An about page {#about-challenge}

Every time we change the way our code is organised, we will set the same small challenge: add an about page.
Doing the same job each time lets us compare.

- Add an about page at [http://localhost:8000/about][about] with another `if` in **server/main.js**. Copy the home page `if` and change the path, the heading and the paragraph.
- Check that it is `200 OK` in the **network** panel.
- Count how many files you touched and how many lines you added or changed.

Keep a note of your count, and of how the job felt.
We will compare them with the same challenge in later exercises.

> **Remove your about page before moving on.**
> Undo your changes to **server/main.js**, so that your code matches the starting point of the next exercise.

Finally, something to think about.
Our 404 response is just the words **Not found** in plain text.
It doesn't have our stylesheet, our icon or a link back to the home page.
How could we make the not-found page look like the rest of the site?

[previous exercise]: deno-server/static-files/
[localhost]: http://localhost:8000/
[books]: http://localhost:8000/books
[anything]: http://localhost:8000/anything/you/like
[assets icon]: http://localhost:8000/assets/icon.svg
[assets styles]: http://localhost:8000/assets/styles.css
[missing]: http://localhost:8000/assets/missing.css
[404]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/404
[strict equality]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Strict_equality
[serve dir]: deno-server/static-files/#serve-dir
[equality]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Equality
[undefined]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/undefined
[500]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status/500
[favicon.ico]: http://localhost:8000/favicon.ico
[search]: http://localhost:8000/?search=deno
[temporary slash]: http://localhost:8000/temporary/
[about]: http://localhost:8000/about

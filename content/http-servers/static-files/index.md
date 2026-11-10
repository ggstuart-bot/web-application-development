---
type: exercise
title: Handling static files
weight: 4
author: Dr Graeme Stuart
---

In this exercise we will give our page an icon and a stylesheet.
We will use the deno standard library to serve these *static files* straight from disk, without letting the browser read our code.

<!--more-->

> You should have a working server which returns a complete HTML document from the [previous exercise][previous exercise].
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

Primarily, we will be exploring how a single page leads the browser to make several requests, and how to answer some of them with files.
By the end of this exercise you should be able to:

- Use the URL API to read the path of an incoming request
- Explain why the browser makes requests we never asked for, such as icons and stylesheets
- Add a dependency from JSR to **deno.json** with `deno add`, and explain what **deno.lock** is for
- Serve static files with `serveDir`, using `fsRoot` and `urlRoot` to keep our code private
- Link an icon and a stylesheet from our HTML document

## Setup

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Open [http://localhost:8000/][localhost] with the developer tools (**F12**) open on the **network** panel.

### A welcome message

*Bookshelves* deserves a better welcome than hello world.
Update the body of our document in **server/main.js**.

**server/main.js**
```JavaScript{hl_lines="15-16"}
export function server(request) {
    // Placeholder: this will need to change when we add middleware
    console.log(request.method, request.url);

    // Placeholder: this will need to change when we add HTML views
    const headers = new Headers({ "content-type": "text/html" });
    const document = `
        <!DOCTYPE html>
        <html lang="en">
            <head>
                <meta charset="utf-8">
                <title>Bookshelves</title>
            </head>
            <body>
                <h1>Welcome to Bookshelves</h1>
                <p>Organise your books with bookshelves.</p>
            </body>
        </html>
    `;
    return new Response(document, { headers });
}
```

Refresh the page and you should see the new heading and a short paragraph.

{{<figure src="images/welcome-unstyled.webp" alt="A plain page with the heading Welcome to Bookshelves and the line Organise your books with bookshelves, in the browser's default font" caption="Our new heading and paragraph, in the browser's default style.">}}

## Which file was requested? {#url-pathname}

In the [previous exercise][request logging] we saw that the browser requests `/favicon.ico` by itself.
The log shows the full URL, but all we really care about is the path.

The [URL constructor][url constructor] turns a URL string into an object with a property for each part of the URL.
Update **server/main.js** to create a `URL` object and log its [pathname][url pathname] instead.

**server/main.js**
```JavaScript{hl_lines="2 5"}
export function server(request) {
    const url = new URL(request.url);

    // Placeholder: this will need to change when we add middleware
    console.log(request.method, url.pathname);

    // Placeholder: this will need to change when we add HTML views
    const headers = new Headers({ "content-type": "text/html" });
    const document = `
        <!DOCTYPE html>
        <html lang="en">
            <head>
                <meta charset="utf-8">
                <title>Bookshelves</title>
            </head>
            <body>
                <h1>Welcome to Bookshelves</h1>
                <p>Organise your books with bookshelves.</p>
            </body>
        </html>
    `;
    return new Response(document, { headers });
}
```

Refresh the page and look at the terminal.

```plaintext {linenos=false}
GET /
GET /favicon.ico
```

> You may not get the same result.
> The browser cache can mean the icon is only requested occasionally.

Much easier to read.
We will use `url.pathname` to decide what to send back.

But what are we sending back right now?
If your browser shows a **favicon.ico** row in the **network** panel, select it and look at the **response** tab.
Either way, you can visit [http://localhost:8000/favicon.ico][favicon.ico] directly.

We get our HTML document.
The browser asked for an icon and we sent it a web page.
It can't use that, so the tab has no icon.

> Every path gets the same document.
> Try [http://localhost:8000/anything/you/like][anything].

## Adding an icon {#linked-resources}

Rather than relying on the browser to guess, we can tell it where our icon is.

Save {{<a href="images/icon.svg" download="icon.svg" >}}this icon{{< /a >}} as **icon.svg** in the root of the project.

Now add a [`<link>`][link element] element to the `<head>` of our document.

**server/main.js**
```JavaScript{hl_lines="15"}
export function server(request) {
    const url = new URL(request.url);

    // Placeholder: this will need to change when we add middleware
    console.log(request.method, url.pathname);

    // Placeholder: this will need to change when we add HTML views
    const headers = new Headers({ "content-type": "text/html" });
    const document = `
        <!DOCTYPE html>
        <html lang="en">
            <head>
                <meta charset="utf-8">
                <title>Bookshelves</title>
                <link rel="icon" href="/icon.svg">
            </head>
            <body>
                <h1>Welcome to Bookshelves</h1>
                <p>Organise your books with bookshelves.</p>
            </body>
        </html>
    `;
    return new Response(document, { headers });
}
```

The [`rel`][rel] attribute says what the linked file is for.
In this case, `icon` means the file is the page's icon.

This is an important idea.
Our server only ever receives one request when we visit a page, but the browser reads the document and *makes more requests by itself* for anything it links to.
Icons, stylesheets, scripts and images are all requested this way.

Refresh the page and check the terminal.

```plaintext {linenos=false}
GET /
GET /icon.svg
```

> Icons are cached aggressively.
> If you still see `/favicon.ico`, try a hard refresh with **Ctrl + Shift + R**.

The browser now asks for `/icon.svg` instead of `/favicon.ico`.
But visit [http://localhost:8000/icon.svg][icon.svg root] and you will see we still send our HTML document.
The file is sitting right there in our project, but our server doesn't know how to send it.

## Serving files with **serveDir**

Files like our icon are *static files*.
They are not generated dynamically by the server, they are stored on disk as complete files.
To handle these requests, all we need to do is grab the correct file and build it into a response, with the right `content-type`.

We could write this ourselves, but the deno standard library already has a [serveDir][servedir] function which does exactly this.

### Adding a dependency {#jsr-imports}

The standard library is published on [JSR][jsr], a registry of JavaScript packages.
We can add the [@std/http module][std http] to our application as a dependency using this command in the terminal.

```bash {linenos=false}
deno add jsr:@std/http
```

It should very quickly complete the download and you should see a change to the **deno.json** configuration file.

**deno.json**
```json{hl_lines="6-8"}
{
    "tasks": {
        "serve": "deno -A tasks/server.js",
        "dev": "deno --watch -A tasks/server.js"
    },
    "imports": {
        "@std/http": "jsr:@std/http@^1.1.4"
    }
}
```

> Your version number may be different.

The new [`"imports"`][imports] property maps the short name `"@std/http"` onto a versioned package on JSR.
Our code can now import from `"@std/http"` and deno will know where to find it.

A **deno.lock** file has also appeared.
This is expected.
It contains the precise versions of all nested dependencies so the project can easily be installed on another machine.
It needs to be committed into version control along with **deno.json**.

> Don't edit **deno.lock** by hand, deno looks after it.

### Serving the whole folder {#serve-dir}

Now we can import `serveDir` and use it.
Update **server/main.js** so that any request for a path other than `/` is passed to `serveDir`.

**server/main.js**
```JavaScript{hl_lines="1 9-12"}
import { serveDir } from "@std/http";

export function server(request) {
    const url = new URL(request.url);

    // Placeholder: this will need to change when we add middleware
    console.log(request.method, url.pathname);

    // Placeholder: this will need to change when we add a router
    if (url.pathname !== "/") {
        return serveDir(request);
    }

    // Placeholder: this will need to change when we add HTML views
    const headers = new Headers({ "content-type": "text/html" });
    const document = `
        <!DOCTYPE html>
        <html lang="en">
            <head>
                <meta charset="utf-8">
                <title>Bookshelves</title>
                <link rel="icon" href="/icon.svg">
            </head>
            <body>
                <h1>Welcome to Bookshelves</h1>
                <p>Organise your books with bookshelves.</p>
            </body>
        </html>
    `;
    return new Response(document, { headers });
}
```

When we call `serveDir(request)` it looks for a file matching the request path and builds a response for it.
By default it looks in the folder we started deno from, the root of our project.
If the path isn't `/`, we [return][if] its response straight away and the rest of the function never runs.

> Another placeholder.
> Checking the path with an `if` is fine for now, but every new page or type of file would need another `if`.
> When we add a *router*, it will decide which code handles each path and this `if` will be replaced.

Refresh the page.
You should see our icon in the browser tab.

> Again, you might need a hard refresh (**Ctrl + Shift + R**).

{{<figure src="images/tab-icon.webp" alt="The top of the browser window, with our Bookshelves icon and title in the tab above the welcome heading" caption="Our icon now appears in the browser tab.">}}

Check the terminal.

```plaintext {linenos=false}
GET /
GET /icon.svg
[2026-10-27 16:21:40] [GET] /icon.svg 200
```

The extra line comes from `serveDir`, which prints its own log line for each file request.
In the **network** panel, the **icon.svg** request now has a `content-type` of `image/svg+xml`, which `serveDir` set for us.

> Notice that our `server` function now returns two different kinds of thing.
> For the home page it returns a `Response`, but `serveDir` returns a [Promise][promise] of a `Response`, because reading a file from disk takes time.
> `Deno.serve` is happy with either, it waits for the promise to finish and sends the response.
> We will look at promises properly later.

### A security problem {#exposing-source}

Everything works, so what's the problem?

Visit [http://localhost:8000/deno.json][deno.json path].
Then try [http://localhost:8000/server/main.js][main.js path].

**Our code is readable by anyone who can reach our server.**

{{<figure src="images/main-js-exposed.webp" alt="The browser showing the code from server/main.js as plain text" caption="Anyone can read our server code at /server/main.js.">}}

`serveDir` is doing exactly what we asked, serving the whole project folder.
That includes our configuration and all our server code.
In a real application this could expose passwords, database files and the details of how our security works.

We need to keep static files in a folder of their own and only serve that folder.

## A public folder {#public-folder}

Create a folder called **public** in which all static files can be stored and move **icon.svg** into it.

```plaintext {linenos=false, hl_lines="4-5 11"}
my-app/
├── .vscode/
│   └── settings.json
├── public/
│   └── icon.svg
├── server/
│   └── main.js
├── tasks/
│   └── server.js
├── deno.json
└── deno.lock
```

Now update **server/main.js** so that only paths starting with `/assets/` are served, from the **public** folder.
We also need to update the icon link.

**server/main.js**
```JavaScript{hl_lines="10-14 25"}
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

    // Placeholder: this will need to change when we add HTML views
    const headers = new Headers({ "content-type": "text/html" });
    const document = `
        <!DOCTYPE html>
        <html lang="en">
            <head>
                <meta charset="utf-8">
                <title>Bookshelves</title>
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
```

We pass `serveDir` a configuration object which specifies
1. `fsRoot` - the location on the filesystem to look for the requested file.
1. `urlRoot` - the url root to remove from the path before looking.

> Notice that we are requesting `/assets/icon.svg` but the file is located at **public/icon.svg**.
> The request path doesn't need to match the folder name.

Refresh the page (with a hard refresh if necessary).
The icon should still be there, and the second request has changed from `GET /favicon.ico` at the start of this exercise to `GET /assets/icon.svg`.
You can also open [http://localhost:8000/assets/icon.svg][assets icon] to see the icon on its own.

```plaintext {linenos=false}
GET /
GET /assets/icon.svg
[2026-10-27 16:21:43] [GET] /assets/icon.svg 200
```

Now visit [http://localhost:8000/deno.json][deno.json path] and [http://localhost:8000/server/main.js][main.js path] again.
They don't start with `/assets/`, so we get our HTML document.
Our code is no longer exposed.

> Our tasks use `-A`, which already grants `--allow-read`, so deno lets `serveDir` read any file.
> A production system might be more explicit about which folders can be read, for example `--allow-read=public`.
> That would add a layer of [protection][permissions] over our code files, even if we made a mistake in our server code.

## Adding a stylesheet {#stylesheet}

Now that we can serve static files, adding a stylesheet is easy.
We use the same `<link>` element as in the [Hello world][hello world] exercise, this time with `rel="stylesheet"`.

**server/main.js**
```JavaScript{hl_lines="25"}
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
```

Create a stylesheet **public/styles.css** with some very simple styles.

**public/styles.css**
```css
:root {
    color-scheme: light dark; /* light or dark page colours, from the OS setting */
}

body {
    margin: 0;
    font-family: system-ui;
    font-weight: 300;
}

h1 {
    font-weight: 400;
}

/* Padding keeps everything off the edges of the window */
body > * {
    padding: 0.5rem 1rem;
}
```

There is nothing new here, these are the same kind of rules we wrote in the [Hello world][hello world] exercise.

Refresh the page.
The heading and paragraph should have a new font and some space around them.
In the terminal, you should see the browser request **styles.css** as well as the icon.
You can open [http://localhost:8000/assets/styles.css][assets styles] to see the stylesheet itself.

```plaintext {linenos=false}
GET /
GET /assets/styles.css
[2026-10-27 16:19:48] [GET] /assets/styles.css 200
GET /assets/icon.svg
[2026-10-27 16:19:48] [GET] /assets/icon.svg 200
```

> If your computer is set to dark mode, the page will be dark.
> This isn't a bug.
> The [*color-scheme*][color-scheme] property tells the browser our page works in both *light* and *dark*, so it uses the default colours for whichever your operating system prefers.
> We haven't written a single colour.
> The icon adapts in the same way, so it is white on a dark tab.

A pretty nice result.

{{<figure src="images/welcome-styled.webp" alt="The welcome page with a new font and some space around the heading and paragraph" caption="With our stylesheet, the page has a new font and some space around it.">}}

## Conclusion

We have seen that a single page can lead the browser to make several requests, for the icon and the stylesheet as well as the document.
We used the URL API to read the path of each request.
We added `serveDir` from the deno standard library and used it to serve static files from a **public** folder at `/assets/`, without exposing our code.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="4-6 12"}
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

## Challenge: Experiment with static files

Try a few experiments with `serveDir` and the stylesheet.

- Visit [http://localhost:8000/assets/nope.css][nope] and look at the status code in the **network** panel.
- Add a picture of your own to the **public** folder and view it in the browser by its `/assets/` path.
- Change the styles in **public/styles.css** and switch your operating system between light and dark mode.

> Make sure **public/styles.css** is back to the version above before moving on.
> You can remove any extra files you added to **public**.

Finally, something to think about.
Visit [http://localhost:8000/deno.json][deno.json path] once more.
We get our home page, with a `200 OK` status.
Is that really the right response for a path that doesn't exist?

[previous exercise]: http-servers/deno-serve/
[localhost]: http://localhost:8000/
[request logging]: http-servers/deno-serve/#request-logging
[url constructor]: https://developer.mozilla.org/en-US/docs/Web/API/URL/URL
[url pathname]: https://developer.mozilla.org/en-US/docs/Web/API/URL/pathname
[favicon.ico]: http://localhost:8000/favicon.ico
[anything]: http://localhost:8000/anything/you/like
[link element]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/link
[rel]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Attributes/rel
[icon.svg root]: http://localhost:8000/icon.svg
[servedir]: https://jsr.io/@std/http/doc/~/serveDir
[jsr]: https://jsr.io/
[std http]: https://jsr.io/@std/http
[imports]: https://docs.deno.com/runtime/fundamentals/modules/#managing-third-party-modules-and-libraries
[if]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/if...else
[promise]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise
[deno.json path]: http://localhost:8000/deno.json
[main.js path]: http://localhost:8000/server/main.js
[permissions]: https://docs.deno.com/runtime/fundamentals/security/#permissions
[hello world]: welcome/hello-world/
[color-scheme]: https://developer.mozilla.org/en-US/docs/Web/CSS/Reference/Properties/color-scheme
[nope]: http://localhost:8000/assets/nope.css
[assets icon]: http://localhost:8000/assets/icon.svg
[assets styles]: http://localhost:8000/assets/styles.css

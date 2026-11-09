---
type: exercise
title: Adding a second page
weight: 220
---

In this exercise, we will upgrade our server to serve two distinct pages.
This will require some routing logic and will create opportunities to avoid repetition when rendering a standard page template.

<!--more-->

> Before you start, make sure everything is committed to your git repository. 
You should have a working **hello world** page from the [previous exercise][previous exercise]. 

## Learning objectives

- Inspecting the request object
- basic routing
- dividing code into [modules]
- the context pattern
- request handlers
- basic view functions
- string interpolation with simple [template literals]



## Understanding the request

We can start by upgrading our server function to receive the incoming request as an argument. 

> The incoming request is passed in by `Deno.serve` but we have been ignoring it until now.

```JavaScript{hl_lines="1-2"}
function server(request) { 
    console.log(request);
    const headers = new Headers({ "content-type": "text/html" }); 
    return new Response("<h1>hello world</h1>", { headers });
}

Deno.serve(server);
```

If you are already running the server, it should restart automatically. 
If not then start the server with your existing *"dev"* task in your terminal.

```bash{linenos=false}
deno task dev
```

Make a request to your server by visiting [the home page] in your browser.

You should see lots of output in your terminal.
Notice that each request is a fairly complex object and there are multiple requests for each page load.

> This is what I saw, yours may be slightly different

```bash{hl_lines=[20, 22, 36, 38, 57, 59]}
Request {
  bodyUsed: false,
  headers: Headers {
    accept: "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7",
    "accept-encoding": "gzip, deflate, br, zstd",
    "accept-language": "en-GB,en-US;q=0.9,en;q=0.8,la;q=0.7",
    "cache-control": "max-age=0",
    connection: "keep-alive",
    host: "localhost:8000",
    "sec-ch-ua": '"Chromium";v="154", "Google Chrome";v="154", "Not A(Brand";v="99"',
    "sec-ch-ua-mobile": "?0",
    "sec-ch-ua-platform": '"Linux"',
    "sec-fetch-dest": "document",
    "sec-fetch-mode": "navigate",
    "sec-fetch-site": "none",
    "sec-fetch-user": "?1",
    "upgrade-insecure-requests": "1",
    "user-agent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/154.0.0.0 Safari/537.36"
  },
  method: "GET",
  redirect: undefined,
  url: "http://localhost:8000/"
}
Request {
  bodyUsed: false,
  headers: Headers {
    "accept-encoding": "gzip, deflate, br, zstd",
    "accept-language": "en-GB,en-US;q=0.9,en;q=0.8,la;q=0.7",
    connection: "keep-alive",
    host: "localhost:8000",
    "sec-fetch-dest": "empty",
    "sec-fetch-mode": "no-cors",
    "sec-fetch-site": "same-origin",
    "user-agent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/154.0.0.0 Safari/537.36"
  },
  method: "GET",
  redirect: undefined,
  url: "http://localhost:8000/.well-known/appspecific/com.chrome.devtools.json"
}
Request {
  bodyUsed: false,
  headers: Headers {
    accept: "image/avif,image/webp,image/apng,image/svg+xml,image/*,*/*;q=0.8",
    "accept-encoding": "gzip, deflate, br, zstd",
    "accept-language": "en-GB,en-US;q=0.9,en;q=0.8,la;q=0.7",
    connection: "keep-alive",
    host: "localhost:8000",
    referer: "http://localhost:8000/",
    "sec-ch-ua": '"Chromium";v="154", "Google Chrome";v="154", "Not A(Brand";v="99"',
    "sec-ch-ua-mobile": "?0",
    "sec-ch-ua-platform": '"Linux"',
    "sec-fetch-dest": "image",
    "sec-fetch-mode": "no-cors",
    "sec-fetch-site": "same-origin",
    "user-agent": "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/154.0.0.0 Safari/537.36"
  },
  method: "GET",
  redirect: undefined,
  url: "http://localhost:8000/favicon.ico"
}
```

We don't need all this information but notice a few things.

Each request contains:

- A method (GET)
- A URL
- The headers we saw in the browser developer tools.

Loading the page once, results in multiple requests:

- "http://localhost:8000/"
- "http://localhost:8000/favicon.ico"

> Again, you may not get the same result. 
> The browser cache can mean some files are only requested occasionally.
> I got a request for "http://localhost:8000/.well-known/appspecific/com.chrome.devtools.json" because I had the developer tools open.
> We can safely ignore this request.

### Extracting the URL

If we are going to make sense of these requests, our server needs to understand what is being requested.
The first step is to grab the URL from the request and look at the pathname component.

> The pathname will distinguish between the root (**'/'**) and e.g. the favicon (**'/favicon.ico'**) without worrying about the host or protocol.

```JavaScript{hl_lines=2}
function server(request) {
    console.log(request.url);    
    const headers = new Headers({ "content-type": "text/html" }); 
    return new Response("<h1>hello world</h1>", { headers });
}

Deno.serve(server);
```

When you refresh the page in the browser, the terminal output should now be much more manageable.

```bash{linenos=false}
http://localhost:8000/
http://localhost:8000/favicon.ico
http://localhost:8000/.well-known/appspecific/com.chrome.devtools.json
```

### Constructing a URL object

If we want to extract the pathname from the URL, we can use the [URL API][url api] to do the work for us.

```JavaScript{hl_lines=[2, 3]}
function server(request) {
    const url = new URL(request.url); 
    console.log(url.pathname);    
    const headers = new Headers({ "content-type": "text/html" }); 
    return new Response("<h1>hello world</h1>", { headers });
}

Deno.serve(server);
```

The output is reduced to the most important detail.

```bash{linenos=false}
/
/.well-known/appspecific/com.chrome.devtools.json
/favicon.ico
```

The URL object contains critical information about the request (e.g. search parameters) and so it becomes part of the context for handling the request. 

## Setting up a template

If we want to serve multiple, more realistic pages from our server then we will need to generate a lot more HTML.
We can start by adapting the example code from a previous exercise.

Create a new file **layout.js** containing a function that just returns our fully-formed HTML document.

```JavaScript{hl_lines="13-14"}
export default function layout() {
    return`
<!DOCTYPE html>
<html>
<head>
    <title>My website</title>
    <link rel="stylesheet" href="/styles.css">
</head>
<body>
    <header>
        <h1>My website</h1>
        <nav>
            <a href="/">home</a>
            <a href="/about">about</a>
        </nav>
    </header>
    <main>        
        <h2>This is an experiment.</h2>
        <p>See the <a href="/about">about</a> page for more information</p>
    </main>
    <footer>
        &copy; My website developers
    </footer>
</body>
</html>`;
}
```
> We made a small adjustment to the link **href** values.
> The string returned is a complete HTML page. 
We will make the function more dynamic later, so we can generate different pages using a similar layout.

Notice we have used `export default`, so we can import this function into our server and return the full page.

```JavaScript{hl_lines=[1, 7, 8]}
import layout from './layout.js';

function server(request) {
    const url = new URL(request.url); 
    console.log(url.pathname);    
    const headers = new Headers({ "content-type": "text/html" }); 
    const html = layout();
    return new Response(html, { headers });
}

Deno.serve(server);
```

Now, when you load the site in the browser, you should see that clicking the links reloads the exact same HTML and that the terminal shows we are receiving new requests each time.

> You should also see requests coming in for the stylesheet **/styles.css**.
> My terminal looks like this after a few clicks.
>
> ```bash{linenos=false}
> /
> /styles.css
> /.well-known/appspecific/com.chrome.devtools.json
> /favicon.ico
> /about
> /styles.css
> /.well-known/appspecific/com.chrome.devtools.json
> /favicon.ico
> /
> /styles.css
> /.well-known/appspecific/com.chrome.devtools.json
> /favicon.ico
> ```
>
> This matches our expectations because the HTML document is loaded and then the browser is trying to load the CSS file and the favicon.

## Create request handlers for each page

The next thing we want to do is serve a different `<main>` element depending on which page was requested.
Our code will branch based on the URL requested.
In order to keep our **main.js** file simple, we will offload the main tasks to the layout function and a simple view function for each page.

### Embed dynamic content in the page

Update **layout.js** so we require content for the page as the only argument.

```JavaScript{hl_lines=[1, 18]}
export default function layout(content) {
    return`
<!DOCTYPE html>
<html>
<head>
    <title>My website</title>
    <link rel="stylesheet" href="/styles.css">
</head>
<body>
    <header>
        <h1>My website</h1>
        <nav>
            <a href="/">home</a>
            <a href="/about">about</a>
        </nav>
    </header>
    <main>        
    ${content}
    </main>
    <footer>
        &copy; My website developers
    </footer>
</body>
</html>`;
}
```

Now we can create a variety of similar pages using the same function.

### Create some **views**

Our simple view functions are just functions that return HTML fragments that can be embedded into the main layout.
Create a folder called **views** with the following two files.

**views/home.js**
```JavaScript
export default function homeView() { 
    return `
    <section>
        <h2>Home</h2>
        <p>This is the home page</p>
    </section>
    `
}
```

**views/about.js**

```JavaScript
export default function aboutView() { 
    return `
    <section>
        <h2>About</h2>
        <p>This is the about page</p>
    </section>
    `
}
```

Finally, we can add a simple pair of conditional statements to fork the request handling within **main.js**.

```JavaScript{hl_lines=["10-20"]}
import layout from './layout.js';
import aboutView from "./views/about.js";
import homeView from "./views/home.js";

function server(request) {
    const url = new URL(request.url); 
    console.log(url.pathname);    
    const headers = new Headers({ "content-type": "text/html" }); 

    if (url.pathname == "/") {
        const content = homeView();
        const html = layout(content);
        return new Response(html, { headers });
    }

    if (url.pathname == "/about") {
        const content = aboutView();
        const html = layout(content);
        return new Response(html, { headers });
    }
}

Deno.serve(server);
```

The result should be that your site now has two distinct pages. 
Requests for the home page and for the about page are handled differently with slightly different results.

However, you may also notice that you are getting errors in the terminal.
```bash
TypeError: Return value from serve handler must be a response or a promise resolving to a response
```
This is because now, only the two page requests are returning a Response object. 
The other requests, e.g. for the stylesheet and icon, are being ignored and returning `undefined` by default.

### Add a **notFound** view

We can solve this by creating a new view function for any requests that are not explicitly handled.

**views/notFound.js**
```JavaScript
export default function notFoundView() { 
    return `
    <section>
        <h2>Not Found</h2>
        <p>The requested resource was not found.</p>
    </section>
    `
}
```

We can add this as a fallback in **main.js** after our conditional statements.

```JavaScript{hl_lines="23-25"}
import layout from './layout.js';
import aboutView from "./views/about.js";
import homeView from "./views/home.js";
import notFoundView from "./views/notFound.js";

function server(request) {
    const url = new URL(request.url); 
    console.log(url.pathname);    
    const headers = new Headers({ "content-type": "text/html" }); 

    if (url.pathname == "/") {
        const content = homeView();
        const html = layout(content);
        return new Response(html, { headers });
    }

    if (url.pathname == "/about") {
        const content = aboutView();
        const html = layout(content);
        return new Response(html, { headers });
    }

    const content = notFoundView();
    const html = layout(content);
    return new Response(html, { headers });
}

Deno.serve(server);
```

All the errors are now gone.
Also note that we can visit a missing url such as [http://localhost:8000/missing] and the **Not Found** page will be rendered.

> We have not set the response status code to 404. 
> We will come back to this later.

It is clear that our code includes a lot of repetition.

### A **render** helper function

We will reduce the repetition with a helper function that allows us to generate an HTML response with a given layout and a given view function.

Create a **tools** folder with a file called **render.js**.
This will be the function we call when we want to render an HTML page.


**tools/render.js**
```JavaScript
export default function render(view, layout) { 
    const headers = new Headers({ "content-type": "text/html" }); 
    const content = view();
    const html = layout(content);
    return new Response(html, { headers });    
}
```

It takes two arguments, both of which are functions.

This allows us to greatly simplify the code in **main.js**.

```JavaScript{hl_lines=[12, 16, 19]}
import layout from './layout.js';
import render from "./tools/render.js";
import aboutView from "./views/about.js";
import homeView from "./views/home.js";
import notFoundView from "./views/notFound.js";

function server(request) {
    const url = new URL(request.url); 
    console.log(url.pathname);    

    if (url.pathname == "/") {
        return render(homeView, layout);
    }

    if (url.pathname == "/about") {
        return render(aboutView, layout);
    }

    return render(notFoundView, layout);
}

Deno.serve(server);
```


Notice that the code reads quite well. 
We are rendering a specific view using the main layout function.
We choose a different view depending on the circumstances.

### Controllers

Our application is pretty simple, we check the incoming request and render a different view depending on the URL requested.
We could easily expand this approach to include many simple static pages.

Although our render function is great for rendering HTML pages, we may want to do more.

> e.g. interact with a database before generating a response.

Even in our existing application we don't just need to render HTML pages, we also need to set the status code.
When the page isn't found, we should be returning a 404 status code as part of the response.

We will add a controller layer into our application for this kind of operation.
Our controller functions will be entirely responsible for handling a request from beginning to end.

Our **main.js** module will remain responsible for routing, i.e. choosing which controller to invoke. 

> We are iterating towards a clean architecture.

Create a folder **controllers** with three controller functions.

The home and about controllers are very simple, they decide to render the appropriate view.

**controllers/home.js**
```JavaScript
import layout from "../layout.js";
import render from "../tools/render.js";
import homeView from "../views/home.js";

export function homeController() { 
    return render(homeView, layout);
}
```

**controllers/about.js**
```JavaScript
import layout from "../layout.js";
import render from "../tools/render.js";
import aboutView from "../views/about.js";

export function aboutController() { 
    return render(aboutView, layout);
}
```

The notFound controller can pass a bit more information to the render function so we can set the status code.

**controllers/notFound.js**
```JavaScript
import layout from "../layout.js";
import render from "../tools/render.js";
import notFoundView from "../views/notFound.js";

export function notFoundController() { 
    return render(notFoundView, layout, { status: 404 });
}
```

The render function needs to integrate this new information.

```JavaScript
export default function render(view, layout, { status } = {}) {
    const headers = new Headers({ "content-type": "text/html" });
    const content = view();
    const html = layout(content);
    return new Response(html, { headers, status });
}
```

> We are [destructuring] the third argument.
> We would get an error if the argument was undefined as we cannot destructure undefined.
> To handle this, we provide an empty object as a default value. 
> With this default, when no value is provided, the status variable simply becomes undefined.

Now we can simply call the controller functions and everything is done.

**main.js**

```JavaScript{hl_lines=["1-3", 10, 14, 17]}
import { aboutController } from "./controllers/about.js";
import { homeController } from "./controllers/home.js";
import { notFoundController } from "./controllers/notFound.js";

function server(request) {
    const url = new URL(request.url);
    console.log(url.pathname);

    if (url.pathname == "/") {
        return homeController()
    }

    if (url.pathname == "/about") {
        return aboutController();
    }

    return notFoundController();
}

Deno.serve(server);
```

The controllers themselves represent *things our application can do*.

If you made the changes to all files (make sure they are all saved) then the site should now work and if you visit [a missing page][http://localhost:8000/missing], the notFound view should be rendered with a 404 status code.

![The status code is properly set](/images/notFound.png)

> Notice that the **styles.css** request is receiving this HTML response which confuses the browser, but the 404 probably helps. 
> We will fix this in the next exercise.

## The context pattern

Now we have controllers and views we can introduce a simple version of *the context pattern* as a core part of our server architecture. 
For our purposes, this means we will create an object which acts as the context for handling a request.
It will contain all the information we need to generate the response and it will eventually pass through many functions and be updated along the way as we gather more information about the request.

What this means in practice is that our render function can actually take in the context object as its only argument.
Everything it needs to access, will be available by destructuring the context object.

**tools/render.js**
```JavaScript{hl_lines=[1, 2]}
export default function render(ctx) {
    const { view, layout, status } = ctx;
    const headers = new Headers({ "content-type": "text/html" });
    const content = view();
    const html = layout(content);
    return new Response(html, { headers, status });
}
```

Now, our controllers will also take in the same context object.
To maintain our working system, we need to make sure there is a view and a layout property set on the context.
Optionally, we can also set a status code.

> Let's assume a default layout is already set.

**controllers/home.js**
```JavaScript {hl_lines=[4, 5, 6]}
import render from "../tools/render.js";
import homeView from "../views/home.js";

export function homeController(ctx) {
    ctx.view = homeView;
    return render(ctx);
}
```

**controllers/about.js**
```JavaScript{hl_lines="4-6"}
import render from "../tools/render.js";
import aboutView from "../views/about.js";

export function aboutController(ctx) { 
    ctx.view = aboutView;
    return render(ctx);
}
```

**controllers/notFound.js**
```JavaScript{hl_lines="4-7"}
import render from "../tools/render.js";
import notFoundView from "../views/notFound.js";

export function notFoundController(ctx) {
    ctx.view = notFoundView;
    ctx.status = 404;
    return render(ctx);
}
```
> Controllers can update the context

Now we can initialise the context object in our **main.js** file.

**main.js**
```JavaScript{hl_lines=[4, 8, 12, 16, 19]}
import { aboutController } from "./controllers/about.js";
import { homeController } from "./controllers/home.js";
import { notFoundController } from "./controllers/notFound.js";
import layout from "./layout.js";

function server(request) {
    const url = new URL(request.url);
    const ctx = { request, url, layout };
    console.log(url.pathname);

    if (url.pathname == "/") {
        return homeController(ctx)
    }

    if (url.pathname == "/about") {
        return aboutController(ctx);
    }

    return notFoundController(ctx);
}

Deno.serve(server);
```

## Conclusions

We have built out the simple beginnings of a Model-View-Controller (MVC) architecture in about 100 lines of code.
The code is divided across nine javascript **modules** which are organised in a simple folder structure so we can easily find the code we want to work on.

We have seen that our HTTP server receives HTTP **Request** objects and returns HTTP **Response** objects. 
We have implemented a simple form of **routing** (via a sequence of conditional statements) to delegate the task of handling each request to an appropriate **controller function**.
We implemented a fallback controller for requests which we don't explicitly understand.  

Controller functions receive a context object which contains basic information about the request.
Our controllers use the **render** helper function to return **Response** objects with the correct HTML **content-type** header. 
We have **view helpers** and a **layout function** which handle the details of creating HTML strings.

However, we are receiving requests for supplementary static files such as an icon and a stylesheet which are not being correctly handled.
Our server simply returns an HTML response for these requests stating that the files were not found.

[modules]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules
[template literals]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals
[the home page]: http://localhost:8000
[url api]: https://developer.mozilla.org/en-US/docs/Web/API/URL
[http://localhost:8000/missing]: http://localhost:8000/missing
[destructuring]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring

[previous exercise]: deno-server/deno-serve/

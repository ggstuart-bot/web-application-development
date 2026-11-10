---
type: exercise
title: A simple web server
weight: 3
author: Dr Graeme Stuart
---

In this exercise we will create our first HTTP server using deno and the fetch API.
We will create an HTTP request handler function in a new **server** folder and start it with a task.

<!--more-->

> You will need deno installed and working.
> See the [Deno setup][deno setup] exercise for details.

## Learning objectives

Primarily, we will be exploring the various ways we can use the [Response constructor][response constructor] and the [Headers constructor][headers constructor] to create [Response][response] objects that can be returned to the browser.
By the end of this exercise you should be able to:

- Create a deno project in an empty folder and open it in VS Code with the Deno extension
- Write a request handler that receives a Request and returns a Response
- Give deno permission to use the network, and run the server with the `serve` and `dev` tasks
- Inspect requests and responses in the browser's **network** panel
- Set the body, `content-type` header and status code of a response
- Log incoming requests and spot the automatic favicon request
- Return a complete HTML document with a doctype, a language, a character set and a title

## Setup

This will be the first of a long series of exercises which build towards a complex and fully featured web application.

> By the end of the module we will have *Bookshelves*, an application for sharing shelves of books.
> For now, we will settle for hello world.

### Create a project folder

Create a new, empty folder for this project and open it in VSCode (**File** > **Open Folder...**).

**Make sure you open the folder, not a file.**
If you only open a file, VSCode won't show your project in the **Explorer** panel and the Deno extension will have nowhere to save its settings.

Then initialise a git repository in the folder.

> See [Getting Started with Git and Github][git in vscode] for details.

### Install the Deno Extension {#deno-extension}

Web servers themselves (programmes which listen for HTTP requests) are not built into JavaScript as a language and there is no web standard for them.
However, Deno does provide the non-standard [Deno.serve][deno.serve] API.

Create a **tasks** folder with a new file inside it called **server.js**.
This is where we will eventually start our server.
Type **Deno** into your **tasks/server.js** file and add a dot.

**tasks/server.js**
```js {linenos=false}
Deno.
```

Notice that you don't get any autocomplete suggestions for the Deno namespace.
We need to install and activate the deno language server for this.

1. In VSCode, open the Extensions sidebar (**Ctrl + Shift + X**)
2. Search for **"Deno"**
3. Click the official **Deno** extension (by the Deno Land company) and select **Install**

![deno extension](/images/vscode/deno-extension.png)

### Activate the Deno extension

The [Deno extension][deno extension] adds knowledge of the APIs and coding conventions from the deno environment to VSCode.
However, installing the deno extension doesn't automatically activate it because if you were writing browser-based JavaScript, you would not want the deno extension activated.

Activate it by typing **Ctrl + Shift + P** to open the VSCode command palette and search for **Deno: Enable**.
Select the enable option.

You should see a new file has been created at **.vscode/settings.json** with the following content.

**.vscode/settings.json**
```json
{
    "deno.enable": true
}
```

This indicates that the deno extension is active.
You should now have autocompletion activated.
Try deleting the dot after `Deno` and typing it again, you should see `serve` among the suggestions.

> The `Deno.serve` API **will** work without the extension.
> The extension just provides a better developer experience.

> Remember to remove the `Deno.` line from **tasks/server.js** when you are done.

## Create our first web server

The `Deno.serve` API requires that we pass in a request handler.

> A request handler is a function that takes a standard [Request][request] object as an argument and returns a standard [Response][response] object.
> This is exactly what a web server should do.
> Deno implements the standard [fetch API][fetch api].

### The **server** folder

All the code for our web application will live in a **server** folder.
Create a **server** folder with a file called **main.js**.

We can begin with the simplest possible valid request handler.
All we do is return a blank response, every time.

**server/main.js**
```JavaScript
export function server() {
    return new Response();
}
```

Notice that we [export][export] the function so it can be imported by other modules.

> We will always use *named exports* like this.

### The **server** task

Our project will have several *entry points*, i.e. modules that we execute directly.
We will keep all of them together in the **tasks** folder.
So we need a small entry point which imports our request handler and passes it to `Deno.serve`.

Update **tasks/server.js** as follows.

**tasks/server.js**
```JavaScript
import { server } from "../server/main.js";

Deno.serve(server);
```

`Deno.serve` will spin up an infinite loop, continuously listening for incoming requests and passing each one to our function.

> The **tasks/server.js** module only *starts* the server.
> The request handler itself knows nothing about how it is being run.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="2-7"}
my-app/
├── .vscode/
│   └── settings.json
├── server/
│   └── main.js
└── tasks/
    └── server.js
```

### Before we start

Before we start the server, let's see what the browser does when there is nothing to talk to.

Open a new browser tab, open the developer tools (**F12**) and select the **network** panel.
Now visit [http://localhost:8000/][localhost].

You should see an error page.
In Chrome it says **This site can't be reached** and **localhost refused to connect**.
In the **network** panel, the **localhost** row is red.
Its status is **(failed)** because no response came back.
Select it and you will see `net::ERR_CONNECTION_REFUSED`.

{{<figure src="images/connection-refused.webp" alt="Chrome showing This site can't be reached and localhost refused to connect, with the network panel below showing a red localhost row with the status (failed) net::ERR_CONNECTION_REFUSED" caption="With no server running, the browser can't connect and the request fails.">}}

> Firefox shows **Unable to connect** instead.

> The browser sent its request, but no programme on your computer was listening for it, so nothing replied.

Keep this tab open, we will come back to it.

### Start the server

Execute the programme.

```bash{linenos=false}
deno tasks/server.js
```

You should get the following warning.

```plaintext {linenos=false}
┏ ⚠️  Deno requests net access to "0.0.0.0:8000".
┠─ Requested by `Deno.listen()` API.
┠─ To see a stack trace for this prompt, set the DENO_TRACE_PERMISSIONS environmental variable.
┠─ Learn more at: https://docs.deno.com/go/--allow-net
┠─ Run again with --allow-net to bypass this prompt.
┗ Allow? [y/n/A] (y = yes, allow; n = no, deny; A = allow all net permissions) > 
```

The warning shows that deno wants to get permission from you before it allows the programme to access the network.

#### Give permission to access the network {#permissions}

Type `y` for **"yes, allow"** and press **Enter**, the server should start with this message.

```plaintext {linenos=false}
✅ Granted net access to "0.0.0.0:8000".
Listening on http://0.0.0.0:8000/ (http://localhost:8000/)
```

The server is now running and will continue to run until you close the programme with **Ctrl + C**.

> Deno is *secure by default*.
> A script cannot access the network, the file system or the environment unless we give it [permission][permissions].

#### Give permission via a task {#tasks}

Stop the server with **Ctrl + C**.

Deno can be configured by creating a [**deno.json**][deno.json] file in the root of the project.
One of the important uses of this file is to create [*tasks*][tasks] which are essentially shortcuts for complex commands.
Since we will often need to start our server, we can create a `serve` task to make life easier.

Create a **deno.json** file in the root of the project and add a `serve` task.

**deno.json**
```json
{
    "tasks": {
        "serve": "deno -A tasks/server.js"
    }
}
```

> The `"tasks"` property is an object.
> Each key is the name of a task and each value is the command that will be executed.

The only permission our server actually needs is network access, so `deno --allow-net tasks/server.js` would be enough.
We have used `-A` (short for `--allow-all`) which grants **every** permission.
This keeps our tasks simple as the project grows, but it means the script now has permission to do bad things, so make sure you know what code you are executing.

Now you should be able to start the server like this.

```bash{linenos=false}
deno task serve
```

All this does is look up the command in **deno.json** and execute it.

```plaintext {linenos=false}
Task serve deno -A tasks/server.js
Listening on http://0.0.0.0:8000/ (http://localhost:8000/)
```

Notice, you were not prompted for permission because you gave explicit permission within the command.

### Make a request with the browser

The server should be running again.
So we can see the result by refreshing the tab we opened earlier, at [http://localhost:8000/][localhost].
You should see a completely blank page.
No errors.
Just a blank page.

{{<iframe src="examples/hello-server/blank.html" width="800" height="200">}}{{</iframe>}}

> Visiting the link causes your browser to issue an HTTP GET request which will be received by our server and passed into our function.

To see what happened, we can look a bit deeper.

#### Inspecting the network traffic

Look at the **network** panel again.
This time the request succeeded.
Click on the **localhost** entry and it should look something like this, although yours will be somewhat different.

![the network panel shows a blank response](/images/blank-response.png)

My browser issued the following HTTP request.
Understanding all this is not necessary.

```http {hl_lines=[1, 6]}
GET / HTTP/1.1
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br, zstd
Accept-Language: en-GB,en-US;q=0.9,en;q=0.8,la;q=0.7
Connection: keep-alive
Host: localhost:8000
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: none
Sec-Fetch-User: ?1
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/154.0.0.0 Safari/537.36
```

> Check the "raw" option in the devtools to see this version.

The key points to note are that it is a `GET` request to the path `/` on the host `localhost:8000`.

Our server generated an extremely minimal response.

```http
HTTP/1.1 200 OK
content-length: 0
date: Fri, 02 Oct 2026 13:30:30 GMT
```

We got an **OK** status code with zero content.

### Change the response

Let's give the browser something more interesting by passing a [body][response body] to the [Response constructor][response constructor].

**server/main.js**
```JavaScript {hl_lines=2}
export function server() {
    return new Response("hello world");
}
```

Refresh the browser and... nothing happens?

The problem here is that the server is still running the old version of the code.
In order to update the server with the new code, we need to stop the programme and restart.
You can do this with **Ctrl + C** in the terminal and restarting the server with:

```bash{linenos=false}
deno task serve
```

> Hint: press the up arrow in the terminal to bring up previous commands and press enter to execute them.

However, to make our life easier, we can enable watch mode.

#### Enable watch mode {#watch-mode}

[Watch mode][watch mode] will stop and restart the server if we change a file.
To enable it, we need to add the `--watch` flag.

We can add a new task `dev` for development, in which we switch on the watch flag.

**deno.json**
```json{hl_lines="3-4"}
{
    "tasks": {
        "serve": "deno -A tasks/server.js",
        "dev": "deno --watch -A tasks/server.js"
    }
}
```

> Make sure you add the comma at the end of the serve task.

> Again, the dev task only needs `--allow-net`, i.e. `deno --watch --allow-net tasks/server.js`.

Stop the server with **Ctrl + C**.
Now starting the server in developer mode is easy.

```bash {linenos=false}
deno task dev
```

Any changes we make from now on will cause the server to restart.

```plaintext {linenos=false}
Task dev deno --watch -A tasks/server.js
Watcher Process started.
Listening on http://0.0.0.0:8000/ (http://localhost:8000/)
```

Refresh the browser and you should now see our message.
Watch mode restarts the server, but we still need to refresh the page to send a new request.

> Watch mode follows our imports, so changes to **server/main.js** also cause a restart.

Of course, we can also run the server in production mode with the `serve` task.

#### Hello world!

The HTTP response is a bit more interesting.

```http
HTTP/1.1 200 OK
content-type: text/plain;charset=UTF-8
content-length: 11
date: Fri, 02 Oct 2026 13:56:21 GMT
```

We now have a `content-length` of `11` and a `content-type` of `text/plain;charset=UTF-8`.
These were added automatically.

> The body of the response can be viewed by switching from the **headers** panel (within the network panel) to the **response** panel.
> You should see the raw plaintext string we returned.

{{<figure src="images/response-panel-hello-world.webp" alt="Chrome showing hello world as plain text, with the network panel open on the localhost request and its Response tab showing hello world" caption="The response panel shows the plain text we sent back.">}}

### An HTML response?

We might think we can just give the browser HTML code and it will render a page for us.
Unfortunately, it's not quite as easy as that.
We need to set the `content-type` header to tell the browser how the body of the response should be interpreted.

We can demonstrate this with a minor change.

**server/main.js**
```JavaScript {hl_lines=2}
export function server() {
    return new Response("<h1>hello world</h1>");
}
```

The browser should show that our response body is still being interpreted as plain text.
We can see the heading tags in the browser viewport.

{{<iframe src="examples/hello-server/plaintext.html" width="800" height="200">}}{{</iframe>}}

The HTTP response looks like this.

```http
HTTP/1.1 200 OK
content-type: text/plain;charset=UTF-8
content-length: 20
date: Fri, 02 Oct 2026 14:11:32 GMT
```

Still clearly showing plain text, this time with 20 characters.

### Setting the `content-type` header {#content-type}

For the browser to understand that the response body should be interpreted as an HTML document, we need to set the [`content-type`][content-type] header to `text/html`.
We can use the [headers] API for this.

**server/main.js**
```JavaScript{hl_lines="2-4"}
export function server() {
    // Placeholder: this will need to change when we add HTML views
    const headers = new Headers({ "content-type": "text/html" });
    return new Response("<h1>hello world</h1>", { headers });
}
```

We create a headers object and [initialise][headers init] it with an object literal containing the header data we want.
We then pass the headers object as a property (the "headers" property) on the [options][response options] object we pass into the Response constructor as a second argument after our html string which represents the [body][response body] of the Response.

> Make sure that makes sense to you.
> Passing an object as a "bag of options" is a common JavaScript pattern.
> Study the linked MDN documentation for details.

> The comment marks these lines as a *placeholder*, code we know will be replaced.
> Writing HTML strings directly into our request handler is fine for one heading, but it won't scale to whole pages.
> When we add HTML *views*, functions which build complete documents, a small helper will set the `content-type` header for us and these lines will be replaced.

You should now see that the browser is properly interpreting our HTML fragment as an HTML document.

{{<iframe src="examples/hello-server/hello.html" width="800" height="200">}}{{</iframe>}}

The response we are giving the browser should indicate the correct headers have been used.

```http
HTTP/1.1 200 OK
content-type: text/html
content-length: 20
date: Fri, 02 Oct 2026 14:21:49 GMT
```

### Setting the status code {#status-codes}

We can also set the [status code][status codes] for the response.
Just add a [status][response status] property to the Response options.

**server/main.js**
```JavaScript{hl_lines=4}
export function server() {
    // Placeholder: this will need to change when we add HTML views
    const headers = new Headers({ "content-type": "text/html" });
    return new Response("<h1>hello world</h1>", { headers, status: 404 });
}
```

> We wouldn't normally do this without a good reason, but we are just experimenting.

The page looks exactly the same, but the network panel should show the new status code.

```http
HTTP/1.1 404 Not Found
content-type: text/html
content-length: 20
date: Fri, 02 Oct 2026 14:25:12 GMT
```

> The status code tells the browser (and any other client) what happened.
> The body is just for humans.

{{<figure src="images/network-status-404.webp" alt="Chrome showing the hello world heading, with the network panel open on the localhost request showing Status Code 404 Not Found and Content-Type text/html" caption="The page looks the same, but the network panel shows 404 Not Found.">}}

Remove the status property again before moving on, so our server returns the default `200 OK`.

### Looking at the request {#request-logging}

`Deno.serve` passes the incoming [Request][request] object into our function, but we have been ignoring it until now.
Let's add a `request` parameter and [log][console.log] the request [method][request method] and [URL][request url].

**server/main.js**
```JavaScript{hl_lines="1-3"}
export function server(request) {
    // Placeholder: this will need to change when we add middleware
    console.log(request.method, request.url);

    // Placeholder: this will need to change when we add HTML views
    const headers = new Headers({ "content-type": "text/html" });
    return new Response("<h1>hello world</h1>", { headers });
}
```

> Another placeholder.
> Logging every request is useful, but it isn't really the job of our request handler.
> When we add *middleware*, functions which run on every request before and after our handlers, the logging will move into its own middleware and this line will be replaced.

Refresh the page and look at the terminal.

```plaintext{linenos=false}
GET http://localhost:8000/
GET http://localhost:8000/favicon.ico
```

> You may not get the same result.
> The browser cache can mean some files are only requested occasionally.

Compare the first line with the raw request we saw in the **network** panel.
The browser sent `GET /` with the host in a separate `Host: localhost:8000` header.
The `Request` object puts them together into a full URL for us.

The second line is a request we never asked for.
Browsers automatically request `/favicon.ico`, looking for a small [icon][favicon] to show in the tab.
Depending on your browser, it may not appear in the **network** panel, but our server still receives it.

Loading the page once can result in multiple requests.
Every request receives the same response from our server.
We will deal with this shortly.

### A complete HTML document {#html-document}

Our response body is just a fragment of HTML, a single heading.
Browsers are very forgiving, so this works, but a proper HTML document has a bit more structure.

Update **server/main.js** to build a complete document.

**server/main.js**
```JavaScript{hl_lines="7-20"}
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
                <h1>hello world</h1>
            </body>
        </html>
    `;
    return new Response(document, { headers });
}
```

We have used a [template literal][template literal] (with backticks) so our document can span several lines.
The `content-type` header is unchanged.

You should recognise this structure from the [Hello world][hello world] exercise.

- The [`<!DOCTYPE html>`][doctype] line tells the browser this is a modern HTML document.
- The [`<html>`][html element] element wraps everything and its [`lang`][lang] attribute says the page is in English.
- The [`<head>`][head element] contains information *about* the page, the [character encoding][meta charset] and a [`<title>`][title element].
- The [`<body>`][body element] contains the content we actually see, our heading.

> The `document` variable is just a name we chose for our string.
> In browser-based JavaScript there is a built-in `document` object which represents the page, and we will meet it later.
> It doesn't exist in deno, so there is no clash.

> The spaces and the newline before `<!DOCTYPE html>` come from indenting our template literal.
> They are harmless, the browser ignores whitespace before the doctype.

Refresh the page.
The heading looks the same, but the browser tab now shows **Bookshelves** as the page title.

{{<figure src="images/tab-title-bookshelves.webp" alt="The top of the browser window, with the tab titled Bookshelves above the hello world heading" caption="The tab now shows our page title, Bookshelves.">}}

The response in the **network** panel has the same headers, with a bigger `content-length`.

```http
HTTP/1.1 200 OK
content-type: text/html
content-length: 267
date: Fri, 02 Oct 2026 14:31:05 GMT
```

> The placeholder comment now covers the document too.
> When we add HTML views, building the document will move out of our request handler.

## Conclusion

We have created a **server** folder containing a request handler which receives a `Request` object and returns a `Response` object containing a complete HTML document.
The **tasks/server.js** module imports our request handler and passes it to `Deno.serve`.
We have `serve` and `dev` tasks to run the server.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="2-8"}
my-app/
├── .vscode/
│   └── settings.json
├── server/
│   └── main.js
├── tasks/
│   └── server.js
└── deno.json
```

## Challenge: Experiment with responses

Try a few experiments with the Response and Headers constructors.

- Set the status to 500 to indicate an unknown server error.
- Try various [4xx] codes to indicate client errors
- Return a [JSON][response json] response using the static `Response.json()` method and see how the `content-type` changes.
- Add a custom header of your own to the Headers object and find it in the network panel.

> Make sure your server returns the HTML document above before moving on.

Finally, something to think about.
Every request receives the same response from our server.
How might we send something different for `/books`?

[deno setup]: javascript-fundamentals/deno-setup/
[git in vscode]: extras/git-in-vscode/
[response]: https://developer.mozilla.org/en-US/docs/Web/API/Response
[deno.serve]: https://docs.deno.com/api/deno/~/Deno.serve
[deno extension]: https://docs.deno.com/runtime/reference/vscode/
[request]: https://developer.mozilla.org/en-US/docs/Web/API/Request
[fetch api]: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
[export]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/export
[localhost]: http://localhost:8000/
[permissions]: https://docs.deno.com/runtime/fundamentals/security/
[deno.json]: https://docs.deno.com/runtime/fundamentals/configuration/
[tasks]: https://docs.deno.com/runtime/reference/cli/task/
[response body]: https://developer.mozilla.org/en-US/docs/Web/API/Response/Response#body
[response constructor]: https://developer.mozilla.org/en-US/docs/Web/API/Response/Response
[headers constructor]: https://developer.mozilla.org/en-US/docs/Web/API/Headers/Headers
[watch mode]: https://docs.deno.com/runtime/getting_started/command_line_interface/#reload-on-change-with-watch-mode
[content-type]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Type
[headers]: https://developer.mozilla.org/en-US/docs/Web/API/Headers
[headers init]: https://developer.mozilla.org/en-US/docs/Web/API/Headers/Headers#init
[response options]: https://developer.mozilla.org/en-US/docs/Web/API/Response/Response#options
[status codes]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status
[response status]: https://developer.mozilla.org/en-US/docs/Web/API/Response/Response#status
[console.log]: https://developer.mozilla.org/en-US/docs/Web/API/console/log_static
[request method]: https://developer.mozilla.org/en-US/docs/Web/API/Request/method
[request url]: https://developer.mozilla.org/en-US/docs/Web/API/Request/url
[favicon]: https://developer.mozilla.org/en-US/docs/Glossary/Favicon
[response json]: https://developer.mozilla.org/en-US/docs/Web/API/Response/json_static
[4xx]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status#client_error_responses
[template literal]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals
[hello world]: welcome/hello-world/
[doctype]: https://developer.mozilla.org/en-US/docs/Glossary/Doctype
[html element]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/html
[lang]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Global_attributes/lang
[head element]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/head
[meta charset]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/meta#charset
[title element]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/title
[body element]: https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/body

---
type: exercise
title: A simple web server
author: Dr Graeme Stuart
weight: 4
---

In this exercise we will create our first HTTP server using deno and the fetch API.
We will create an HTTP request handler function that accepts a request object and returns a response object.

<!--more-->

## Learning objectives

Primarily, we will be exploring the various ways we can use the [Response constructor][response constructor] to create [Response][response] objects that can be returned to the browser.
We will also see how to configure tasks in a deno project and how to put together a very simple web server.

## Setup

This will be the first of a long series of exercises which build towards a complex and fully featured web framework.

### Initialise a git repository

Before you start, you should initialise a git repository.

> See [Getting Started with Git and Github]({{< ref "/extras/git-in-vscode" >}}) for details. 


### Install the Deno Extension

Web servers themselves (programmes which listen for HTTP requests) are not built into JavaScript as a language and there is no web standard for them.
However, Deno does provide the non-standard [Deno.serve][deno.serve] API. 

Type **Deno** into your `main.js` file and add a dot.

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

The Deno extension adds knowledge of the APIs and coding conventions from the deno environment to VSCode.
However, installing the deno extension doesn't automatically activate it because if you were writing browser-based JavaScript, you would not want the deno extension activated.

Activate it by typing **Ctrl + Shift + P** to open the VSCode command palette and search for **Deno: Enable**.
Select the enable option.

You should see a new file has been created at *.vscode/settings.json* with the following content.

```json
{
    "deno.enable": true
}
```

This indicates that the deno extension is active.
You should now have autocompletion activated.

> The `Deno.serve` API **will** work without the extension.
> The extension just provides a better developer experience. 

## Create our first web server

The `Deno.serve` API requires that we pass in a request handler.

> A request handler is a function that takes a standard [Request][request] object as an argument and returns a standard [Response][response] object.
> This is exactly what a web server should do.
Deno implements the standard [fetch API][fetch api].

We can begin with the simplest possible valid request handler.
All we do is return a blank response, every time.

```JavaScript
function server() { 
    return new Response();
}

Deno.serve(server);
```

We pass the request handler to `Deno.serve` which will spin up an infinite loop, continuously listening for incoming requests.

### Start the server

Execute the programme.

```bash{linenos=false}
deno main.js
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

The warning shows that deno wants to get permission from you before it allows the programme to access the internet.

#### Give permission to access the internet

Type **y** for **"yes, allow"** and press enter, the server should start with this message.

```plaintext {linenos=false}
✅ Granted net access to "0.0.0.0:8000".
Listening on http://0.0.0.0:8000/ (http://localhost:8000/)
```

The server is now running and will continue to run until you close the programme with **Ctrl + C**.

#### Give permission with a flag

Stop the server with **Ctrl + C** and restart with an additional `--allow-net` flag.

```bash{linenos=false}
deno --allow-net main.js
```

> If you get problems on the lab machines, you can add `--allow-all` instead, just make sure you know what code you are executing because the script now has permission to do bad things.

Notice, you were not prompted for permission because you gave explicit permission within the command.

#### Give permission via a task

Deno can be configured by creating a **deno.json** file in the root of the project.
One of the important uses of this file is to create `tasks` which are essentially shortcuts for complex commands.
Since we will often need to start our server, and the command is already quite long (it will get longer) we can create a `serve` task to make life easier. 

Create a **deno.json** file and add a task we can use for starting the server.

```json
{
    "tasks": {
        "serve": "deno --allow-net main.js"
    }
}
```

Now you should be able to execute the task from the terminal like this.

```bash{linenos=false}
deno task serve
```

All this does is lookup the command in **deno.json** and executes it.

### Make a request with the browser

The server should be running again. 
So we can see the result by visiting [http://localhost:8000/][localhost].
You should see a completely blank page. 
No errors. 
Just a blank page.

{{<iframe src="examples/hello-server/blank.html" width="800" height="200">}}{{</iframe>}}


> Visiting the link causes your browser to issue an HTTP GET request which will be received by our server and passed into our function.

To see what happened, we can look a bit deeper.

#### Inspecting the network traffic

Open up the developer tools in your browser and view the network panel.
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
sec-ch-ua: "Chromium";v="154", "Google Chrome";v="154", "Not A(Brand";v="99"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "Linux"
```

> Check the "raw" option in the devtools to see this version. 

This shows all the core information available to our server.
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

```JavaScript {hl_lines=2}
function server() { 
    return new Response("hello world");
}

Deno.serve(server);
```

Refresh the browser and... nothing happens?

The problem here is that the server is still running the old version of the code. 
In order to update the server with the new code, we need to stop the programme and restart.
You can do this with *Ctrl + C* in the terminal and restarting the server with:

```bash{linenos=false}
deno task serve
```

> Hint: press the up arrow in the terminal to bring up previous commands and press enter to execute them. 

However, to make our life easier, we can enable watch mode.

#### Enable watch mode

Watch mode will stop and restart the server if we change a file.
To enable it, we need to add the `--watch` flag into our task.

In fact, we can add a new task **dev** for development, in which we switch on the watch flag.

```json{hl_lines="3-4"}
{
    "tasks": {
        "serve": "deno --allow-net main.js",
        "dev": "deno --watch --allow-net main.js"
    }
}
```
> Make sure you add the comma at the end of the serve task. 
>
> Again, if you have `--allow-all` activated then you can adapt the example accordingly.

Now starting the server in developer mode is easy.

```bash {linenos=false}
deno task dev
```

Any changes we make from now on will cause the server to restart.

```plaintext {linenos=false}
Task dev deno --watch --allow-net main.js
Watcher Process started.
Listening on http://0.0.0.0:8000/ (http://localhost:8000/)
```

Of course, we can also run the server in production mode with the original task.

#### Hello world!

You should now see our message appears in the browser!

The HTTP response is a bit more interesting.

```http
HTTP/1.1 200 OK
content-type: text/plain;charset=UTF-8
content-length: 11
date: Fri, 02 Oct 2026 13:56:21 GMT
```

We now have a `content-length` of **11** and a `content-type` of **text/plain;charset=UTF-8**.
These were added automatically.

> The body of the response can be viewed by switching from the **headers** panel (within the network panel) to the **response** panel.
You should see the raw plaintext string we returned.

### An HTML response?

We might think we can just give the browser HTML code and it will render a page for us.
Unfortunately, its not quite as easy as that. 
We need to set the **content-type** header to tell the browser how the body of the response should be interpreted.

We can demonstrate this with a minor change.

```JavaScript {hl_lines=2}
function server() { 
    return new Response("<h1>hello world</h1>");
}

Deno.serve(server);
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

### Setting the **content-type** header

For the browser to understand that the response body should be interpreted as an HTML document, we need to set the [**content-type**][content-type] header to `text/html`.
We can use the [headers] API for this.

```JavaScript{hl_lines=[2, 3]}
function server() { 
    const headers = new Headers({ "content-type": "text/html" }); 
    return new Response("<h1>hello world</h1>", { headers });
}

Deno.serve(server);
```

We create a headers object and [initialise][headers init] it with an object literal containing the header data we want.
We then pass the headers object as a property (the "headers" property) on the [options][response options] object we pass into the Response constructor as a second argument after our html string which represents the [body][response body] of the Response.

> Make sure that makes sense to you.
> Passing an object as a "bag of options" is a common JavaScript pattern.
> Study the linked MDN documentation for details.

You should now see that the browser is properly interpreting our HTML fragment as an HTML document.

{{<iframe src="examples/hello-server/hello.html" width="800" height="200">}}{{</iframe>}}


The response we are giving the browser should indicate the correct headers have been used.

```http
HTTP/1.1 200 OK
content-type: text/html
content-length: 20
date: Fri, 02 Oct 2026 14:21:49 GMT
```

## Challenge: Experiment with status codes

We can also set the status code for the response. 
Just add a [status][response status] property to the Response options.

> We wouldn't normally do this without a good reason, but we are just experimenting.

Try setting the status to various codes:
- 500 to indicate an unknown server error.
- [4xx] to indicate various client errors

[response]: https://developer.mozilla.org/en-US/docs/Web/API/Response
[deno.serve]: https://docs.deno.com/api/deno/http-server/#Deno.serve
[request]: https://developer.mozilla.org/en-US/docs/Web/API/Request
[fetch api]: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
[localhost]: http://localhost:8000/
[response body]: https://developer.mozilla.org/en-US/docs/Web/API/Response/Response#body
[response constructor]: https://developer.mozilla.org/en-US/docs/Web/API/Response/Response
[content-type]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Type
[headers]: https://developer.mozilla.org/en-US/docs/Web/API/Headers
[headers init]: https://developer.mozilla.org/en-US/docs/Web/API/Headers/Headers#init
[response options]: https://developer.mozilla.org/en-US/docs/Web/API/Response/Response#options
[response status]: https://developer.mozilla.org/en-US/docs/Web/API/Response/Response#status
[4xx]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status#client_error_responses

---
type: exercise
title: Escape everything
weight: 3
author: Dr Graeme Stuart
---

In this exercise we will see what a user can do with our search box, and then stop them.
We will write an `html` helper that *escapes* every value we put into our pages.

<!--more-->

> You should have a working search from the [previous exercise][previous exercise].
> Your database should hold the 20 books we seeded. If you have been experimenting, run `deno task db:reset` to get back to 20.
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

This exercise is about *security*.
First we will attack our own application, then we will fix it one template at a time.
By the end of this exercise you should be able to:

- Explain how user input in a page can become HTML, and what *cross-site scripting* is
- Explain what escaping does, and why it is needed in text and in attributes
- Write a *tagged template* function that escapes every value it is given
- Use `reduce` to build one string from an array
- Protect every view and the layout with one `html` helper

## What can a search do? {#attacks}

At the end of the last exercise, a search for **"hobbit"**, with quotes, [broke our search box][quotes].
Our code writes whatever the user typed straight into our HTML, in four places: the page title, the heading, the box and the "no matches" message.
Let's see how far that goes.

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

> Only ever try attacks like these on your own application, running on your own computer.

### Stage 1: Formatting {#formatting}

Search for `<strong>Hobbit</strong>`, or visit [this link][bold].

The heading now says **Books matching "Hobbit"**, with **Hobbit** in bold, and so does the "no matches" message.
The browser didn't show our user's text; it read it as HTML, and made a `<strong>` element.

The box and the browser tab still show `<strong>Hobbit</strong>`, tags and all.
The `value` attribute is just text to the browser, and so is everything inside `<title>`.

{{<figure src="images/xss-bold.webp" alt="The books page after a search for Hobbit wrapped in strong tags. The heading reads Books matching “Hobbit” with Hobbit in bold, and so does the No books match message, while the tab title and the search box show the strong tags as text" caption="The heading shows **Hobbit** in bold, but the box still shows the tags.">}}

Try a few more.

- [`<em>Hobbit</em>`][emphasis]
- [`<mark>Hobbit</mark>`][mark]
- [`<s>Hobbit</s>`][strike]

### Stage 2: Breaking the layout {#layout}

Bold text is harmless, but the user can add *any* element.

Visit [`<h1 style=color:red>SALE</h1>`][sale].
Two huge red **SALE** headings appear: one in our heading, and one in the message.

{{<figure src="images/xss-sale.webp" alt="The books page after a search for an h1 element with a red style. Two large red SALE headings appear on the page, one under the Books matching heading and one under the No books match message, while the search box shows the HTML that was typed" caption="Our search has become two red headings.">}}

Try these too.

- [`<img src=/assets/icon.svg width=2000>`][huge image] fills the page with our logo, twice.
- [`<hr><hr><hr>`][rules] adds horizontal lines.

### Stage 3: Fake content {#fake-content}

Now visit this one.

```plaintext {linenos=false}
</h1><p>Site closed. <a href=https://example.com>Log in here</a></p>
```

[Try it][site closed].
The `</h1>` closes our heading early, so the user's paragraph looks like part of our page.
It says **Site closed. Log in here**, and the link goes to [example.com][example], not to us.

{{<figure src="images/xss-site-closed.webp" alt="The books page showing the text Site closed. Log in here, twice, with the mouse over the Log in here link and the browser status bar showing https://example.com" caption="Hovering over the link shows where it really goes.">}}

Here is another one: [`<p>Free books for everyone! <a href=https://example.com>Claim yours</a></p>`][free books].

Now look at the address bar.
The query is part of the URL, so this page can be shared like any other link.
An attacker doesn't need to touch our server.
They only need someone to click a link that starts with *our* address, and the page they see says what the attacker wants.
A fake login page on a real site's address is a classic way to steal passwords.

### Stage 4: Running code {#script}

HTML can include JavaScript, so a user can type that too.

Visit [`<script>alert('oops')</script>`][oops].

The browser shows an alert saying **oops**, and then another, because the script is in the heading *and* in the message.
The page behind the first alert is still blank, because the browser runs a script as soon as it reaches it, before it has drawn the rest of the page.

{{<figure src="images/xss-alert.webp" alt="A browser alert saying localhost:8000 says oops, over a blank grey page, with the script search in the address bar" caption="The browser ran the script from our search.">}}
The browser ran code that a user typed, as part of our page.

Scripts don't need a `<script>` element.
Try [`<img src=x onerror=alert('img')>`][onerror]: the image fails to load, so the browser runs its `onerror` attribute, once for each copy.

This is called [*cross-site scripting*][xss], or XSS.
Because the script arrives in the link and comes straight back in our response, it is *reflected* cross-site scripting.

An alert is harmless, but a real script can do anything our own page can do.
It can read everything on the page, change it, and send requests to our server *as the user*.
In week 4 our users will sign in, and our pages will show their own data.
A script like this would then be acting with their account.

### Stage 5: Not just tags {#contexts}

Our page title looked safe, because the browser doesn't read tags inside `<title>`.
But it does look for the end of the title.

Visit [`</title><script>alert('title')</script>`][title attack].
You get *three* alerts this time.
The `</title>` ends our title, so the script after it runs in the `<head>`, and the tab now says just **Books matching "**.

The box is even easier to break, and it doesn't need `<` or `>` at all.
Remember how a `"` closed our `value` attribute early?

```plaintext {linenos=false}
" autofocus onfocus="alert('value')
```

Visit [this search][value attack].
The box comes back empty, and an alert says **value**.

{{<figure src="images/xss-value.webp" alt="A browser alert saying localhost:8000 says value, over the books page. The search box behind it is empty, and the No books match message shows the attribute text that was typed" caption="Every time the box gets focus, it runs the user’s code.">}}
View the page source (**Ctrl+U**) and find the input.

```html
<input id="q" type="search" name="q" value="" autofocus onfocus="alert('value')">
```

The user's `"` closed our attribute, and the rest of their search became two new attributes.
`autofocus` puts the cursor in the box when the page loads, and `onfocus` then runs their code.

So it isn't enough to remove `<script>`.
Every value we write into our HTML must be made safe for where it goes, in text and in attributes.

## Escaping {#escaping}

The fix is called *escaping*.
Before we write a value into our HTML, we replace each character that means something in HTML with a [*character reference*][character reference], which the browser shows as that character but never reads as HTML.

| Character | Escaped |
|---|---|
| `&` | `&amp;` |
| `<` | `&lt;` |
| `>` | `&gt;` |
| `"` | `&quot;` |
| `'` | `&#39;` |

We have used one already: `&copy;` in our footer shows as ©.

We could escape each value by hand, but sooner or later we would forget one, and one is all an attacker needs.
Instead, we will make escaping the default, with a helper that every template uses.

### The escape function {#std-html}

The deno standard library has an [`escape`][std html] function.
Add it to our project.

```bash {linenos=false}
deno add jsr:@std/html@1
```

```plaintext {linenos=false}
Add jsr:@std/html@1.0.7
```

> As before, **deno.lock** will change too.
> Commit both files as usual.

**deno.json**
```json{hl_lines="11"}
{
    "tasks": {
        "serve": "deno -A tasks/server.js",
        "dev": "deno --watch -A tasks/server.js",
        "db:create": "deno -A tasks/db-create.js",
        "db:seed": "deno -A tasks/db-seed.js",
        "db:reset": "deno task db:create && deno task db:seed"
    },
    "imports": {
        "@db/sqlite": "jsr:@db/sqlite@^0.12.0",
        "@std/html": "jsr:@std/html@^1.0.7",
        "@std/http": "jsr:@std/http@^1.1.4"
    }
}
```

It does exactly what the table above says.

```JavaScript {linenos=false}
escape(`<strong>"Hobbit" & 'friends'</strong>`)
```

```plaintext {linenos=false}
&lt;strong&gt;&quot;Hobbit&quot; &amp; &#39;friends&#39;&lt;/strong&gt;
```

### Tagged templates {#tagged-templates}

All our views build HTML with template literals.
JavaScript lets us put a function name in front of a template literal, like this.

```JavaScript {linenos=false}
html`<p>No books match "${q}"</p>`
```

This is a [*tagged template*][tagged templates].
Instead of building the string itself, JavaScript calls the function, `html`, with the pieces.

- The first argument is an array of the fixed strings: the parts of the template *between* the `${}`.
- Each value from a `${}` comes as one more argument.

For the template above, with `q` set to `<strong>Hobbit</strong>`, the function is given these.

```plaintext {linenos=false}
[ '<p>No books match "', '"</p>' ]
"<strong>Hobbit</strong>"
```

There is always one more string than there are values, because a value always sits *between* two strings.

So a tag function sees our fixed HTML and the values separately.
It can trust the strings, which we wrote, and escape every value, which might have come from anyone.

### An html helper {#html}

Create **html.js** in the **server/tools** folder.

**server/tools/html.js**
```JavaScript
import { escape } from "@std/html";

function toHTML(value) {
    if (value === undefined || value === null) return "";
    return escape(String(value));
}

export function html(strings, ...values) {
    return strings.reduce(
        (output, string, index) => output + string + toHTML(values[index]),
        "",
    );
}
```

`toHTML` turns one value into safe HTML.

- `undefined` and `null` become an empty string, rather than the words "undefined" or "null".
- Anything else is turned into a string with [`String`][string], so a number such as a year works too, and then escaped.

`html` is our tag function.

- `...values` is a [*rest parameter*][rest parameters]. It collects all the values into one array, however many there are, just as the [rest element][rest element] in our router's `chain` collected the remaining handlers.
- [`reduce`][reduce] builds one result from an array. It calls our arrow function once for each string, passing in the result so far as `output`, the current `string` and its `index`. Whatever the function returns becomes the next `output`. The `""` at the end is where `output` starts.
- Each time, we add the string and then the escaped value with the same index. After the last string there is no value, so `values[index]` is `undefined`, and `toHTML` adds nothing.

### Escaping the search box {#search-box}

The search box is where our quote problem started, so let's protect it first.
Update **server/views/partials/searchForm.js**.

**server/views/partials/searchForm.js**
```JavaScript{hl_lines="1 6"}
import { html } from "../../tools/html.js";

// The book search form, shared by the home page and the books page.
// Pass the current search to pre-fill the box
export function searchForm(q = "") {
    return html`
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

The only changes are the import and the word `html` before the template.

Now try the [`value` attack][value attack] again.
There is no alert this time, and the box shows exactly what was typed.
In the page source, every `"` and `'` is now a character reference.

```html
<input id="q" type="search" name="q" value="&quot; autofocus onfocus=&quot;alert(&#39;value&#39;)">
```

But our other templates still aren't escaped: try [`<script>alert('oops')</script>`][oops] and you still get two alerts.

## Escaping the book cards {#cards}

Users don't only reach our pages through the search.
Our book titles and authors come from the database, and one day our users will be adding books.

Update **server/views/partials/bookCard.js**.

**server/views/partials/bookCard.js**
```JavaScript{hl_lines="1 4"}
import { html } from "../../tools/html.js";

export function bookCard(book) {
    return html`
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
Nothing looks different, but view the page source and find **Angels & Demons**.

```html
<h2>Angels &amp; Demons</h2>
```

The `&` is now escaped, which is how it should always have been written.

## HTML inside HTML {#nesting}

Next, let's convert the home page.
Update **server/views/home.js**.

**server/views/home.js**
```JavaScript{hl_lines="1 5"}
import { html } from "../tools/html.js";
import { searchForm } from "./partials/searchForm.js";

export function homeView() {
    return html`
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

Visit [http://localhost:8000/][localhost].
Our search form has gone, and in its place is its HTML, shown as text.

```html
        &lt;search&gt;
            &lt;form method=&quot;GET&quot; action=&quot;/books&quot;&gt;
```

`searchForm()` returns a string, so the home page's `html` escaped it, just like a search.
Our `html` helper can't tell the difference between HTML we built and text a user typed.
We need to mark the output of `html` as *safe*, so that it is only escaped once.

Update **server/tools/html.js**.

**server/tools/html.js**
```JavaScript{hl_lines="3-10 14 19 22"}
import { escape } from "@std/html";

class SafeHTML {
    constructor(value) {
        this.value = value;
    }
    toString() {
        return this.value;
    }
}

function toHTML(value) {
    if (value === undefined || value === null) return "";
    if (value instanceof SafeHTML) return value.value;
    return escape(String(value));
}

export function html(strings, ...values) {
    return new SafeHTML(strings.reduce(
        (output, string, index) => output + string + toHTML(values[index]),
        "",
    ));
}
```

`SafeHTML` is a small [class][classes], like our `Router`.
An object made from it holds a string that we know is safe HTML.

- `html` now wraps its result with `new SafeHTML(...)`.
- In `toHTML`, a `SafeHTML` value is passed through as it is, checked with [`instanceof`][instanceof] just as in our router's error handling. Everything else is still escaped.
- [`toString`][tostring] is a special method. JavaScript calls it whenever an object is used as a string, for example in a plain template literal, so code that expects a string still works.

Refresh [http://localhost:8000/][localhost].
The form is back.

## Arrays of HTML {#arrays}

Now let's convert the books list.
Update **server/views/books.js**.

**server/views/books.js**
```JavaScript{hl_lines="1 8-9"}
import { html } from "../tools/html.js";
import { bookCard } from "./partials/bookCard.js";
import { searchForm } from "./partials/searchForm.js";

export function booksView(ctx) {
    const { data: { title, books, q } } = ctx;
    const results = books.length
        ? html`<ul class="grid">${books.map(bookCard).join("")}</ul>`
        : html`<p>${q ? `No books match "${q}"` : "No books found"}</p>`;

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

Refresh [http://localhost:8000/books][books].
The cards have gone, and their HTML is shown as text instead, starting with `&lt;li class=&quot;card book&quot;&gt;`.

Each `bookCard` is now `SafeHTML`, but `join` turns the whole array into one plain string, and plain strings get escaped.
Rather than joining the cards ourselves, we will let `html` handle arrays.

Update **server/tools/html.js**.

**server/tools/html.js**
```JavaScript{hl_lines="15"}
import { escape } from "@std/html";

class SafeHTML {
    constructor(value) {
        this.value = value;
    }
    toString() {
        return this.value;
    }
}

function toHTML(value) {
    if (value === undefined || value === null) return "";
    if (value instanceof SafeHTML) return value.value;
    if (Array.isArray(value)) return value.map(toHTML).join("\n");
    return escape(String(value));
}

export function html(strings, ...values) {
    return new SafeHTML(strings.reduce(
        (output, string, index) => output + string + toHTML(values[index]),
        "",
    ));
}
```

[`Array.isArray`][is array] checks whether a value is an array.
If it is, we turn each item into HTML with `toHTML` itself, so each card is passed through as safe and anything else is escaped.
Then we join them, with a new line between each one.

Now remove the `join` from **server/views/books.js**.

**server/views/books.js**
```JavaScript{hl_lines="8"}
import { html } from "../tools/html.js";
import { bookCard } from "./partials/bookCard.js";
import { searchForm } from "./partials/searchForm.js";

export function booksView(ctx) {
    const { data: { title, books, q } } = ctx;
    const results = books.length
        ? html`<ul class="grid">${books.map(bookCard)}</ul>`
        : html`<p>${q ? `No books match "${q}"` : "No books found"}</p>`;

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

Refresh [http://localhost:8000/books][books].
The cards are back.

The message is now escaped too.
The plain template literal inside it builds the message as a string, and the `html` around it escapes that string.
Try [`<script>alert('oops')</script>`][oops]: there is only one alert now, from the heading.

## The views {#views}

The heading is in the template that the books view returns.
Update **server/views/books.js** again.

**server/views/books.js**
```JavaScript{hl_lines="11 15"}
import { html } from "../tools/html.js";
import { bookCard } from "./partials/bookCard.js";
import { searchForm } from "./partials/searchForm.js";

export function booksView(ctx) {
    const { data: { title, books, q } } = ctx;
    const results = books.length
        ? html`<ul class="grid">${books.map(bookCard)}</ul>`
        : html`<p>${q ? `No books match "${q}"` : "No books found"}</p>`;

    return html`
        <section>
            <header>
                <h1>${title}</h1>
                ${q ? html`<p><a href="/books">show all books</a></p>` : ""}
            </header>
            ${searchForm(q)}
            ${results}
        </section>
    `;
}
```

The "show all books" link is HTML we wrote, so it uses `html` too, to keep it safe inside the outer template.

The not-found view has no values in it yet, but we will make it safe now, so nobody has to remember later.
Update **server/views/notFound.js**.

**server/views/notFound.js**
```JavaScript{hl_lines="1 4"}
import { html } from "../tools/html.js";

export function notFoundView() {
    return html`
        <section>
            <header>
                <h1>Not found</h1>
                <p>The requested resource was not found.</p>
            </header>
        </section>
    `;
}
```

Try [`<script>alert('oops')</script>`][oops] again.
No alert, and the heading shows the search as text.
But [`</title><script>alert('title')</script>`][title attack] still gives one alert, from the `<head>`.

## The layout {#layout-html}

The last template is the layout, which writes the title.
Update **server/views/layout.js**.

**server/views/layout.js**
```JavaScript{hl_lines="1 8 19-32 40-42"}
import { html } from "../tools/html.js";

// Placeholder: this will need to change when we add flash messages and sign-in
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
}

// A header link, marked as the current page when it matches the path exactly
function navLink(href, content, path) {
    return html`<a href="${href}" aria-current="${
        href === path ? "page" : "false"
    }">${content}</a>`;
}
```

- The layout returns `html` straight away, so we no longer need the `document` variable.
- The `content` from the view is already `SafeHTML`, so it goes in as it is.
- `navLink` uses `html` too. The tidy-up of its line breaks, and of the indentation under `<body>`, is just formatting.

`render` doesn't need to change.
It passes the layout's result to `new Response`, which uses it as a string, through `toString`.

## Trying the attacks again {#again}

Go back through every stage, from [`<strong>Hobbit</strong>`][bold] to the [`value` attack][value attack].

- There are no alerts at all.
- Nothing is bold, red, huge or fake.
- Each search shows as text, exactly as it was typed, in the heading, the message, the box and the tab.

{{<figure src="images/escaped.webp" alt="The books page after a search for Hobbit wrapped in strong tags, with escaping in place. The heading shows the tags as plain text, and the search box and No books match message show the same text" caption="With escaping in place, the search is just text.">}}

The page source for the first search shows why.

```html
                <title>Books matching &quot;&lt;strong&gt;Hobbit&lt;/strong&gt;&quot; | Bookshelves</title>
```

Every character that could have started some HTML has been escaped.

## Conclusion

Anything a user types is *untrusted*.
If we write it into our HTML as it is, they can format our page, change it, put fake content on it or run their own code there.
Because the search lives in the URL, an attacker only needs someone to click a link: this is reflected cross-site scripting.

We fixed it by escaping by default.
Our `html` tag function escapes every value, and marks its own output as safe, so templates can be nested and arrays of cards still work.
Every view, every partial and the layout now use it.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="25"}
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
│   │   ├── html.js
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

## Challenge: Check our defences

These are questions to investigate, not changes to keep.

- Search for `<strong>Hobbit</strong>` and view the page source. Find all four places the search appears. Is each one escaped the same way?
- Two plain template literals are left: the title in **server/views/layout.js** and the message in **server/views/books.js**, and the controller builds `Books matching "${q}"` with one too. Why are they still safe?
- Try your own attacks from each stage. Can you find one that still works?
- What would happen if a book's title in the database contained `<script>`? Which commit protects us from that?

> If you tried any changes, undo them, so that your code matches the starting point of the next exercise.

[previous exercise]: user-input-and-security/search/
[quotes]: user-input-and-security/search/#quotes
[bold]: http://localhost:8000/books?q=%3Cstrong%3EHobbit%3C%2Fstrong%3E
[emphasis]: http://localhost:8000/books?q=%3Cem%3EHobbit%3C%2Fem%3E
[mark]: http://localhost:8000/books?q=%3Cmark%3EHobbit%3C%2Fmark%3E
[strike]: http://localhost:8000/books?q=%3Cs%3EHobbit%3C%2Fs%3E
[sale]: http://localhost:8000/books?q=%3Ch1%20style%3Dcolor%3Ared%3ESALE%3C%2Fh1%3E
[huge image]: http://localhost:8000/books?q=%3Cimg%20src%3D%2Fassets%2Ficon.svg%20width%3D2000%3E
[rules]: http://localhost:8000/books?q=%3Chr%3E%3Chr%3E%3Chr%3E
[site closed]: http://localhost:8000/books?q=%3C%2Fh1%3E%3Cp%3ESite%20closed.%20%3Ca%20href%3Dhttps%3A%2F%2Fexample.com%3ELog%20in%20here%3C%2Fa%3E%3C%2Fp%3E
[free books]: http://localhost:8000/books?q=%3Cp%3EFree%20books%20for%20everyone%21%20%3Ca%20href%3Dhttps%3A%2F%2Fexample.com%3EClaim%20yours%3C%2Fa%3E%3C%2Fp%3E
[example]: https://example.com
[oops]: http://localhost:8000/books?q=%3Cscript%3Ealert%28%27oops%27%29%3C%2Fscript%3E
[onerror]: http://localhost:8000/books?q=%3Cimg%20src%3Dx%20onerror%3Dalert%28%27img%27%29%3E
[title attack]: http://localhost:8000/books?q=%3C%2Ftitle%3E%3Cscript%3Ealert%28%27title%27%29%3C%2Fscript%3E
[value attack]: http://localhost:8000/books?q=%22%20autofocus%20onfocus%3D%22alert%28%27value%27%29
[xss]: https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/XSS
[character reference]: https://developer.mozilla.org/en-US/docs/Glossary/Character_reference
[std html]: https://jsr.io/@std/html/doc/~/escape
[tagged templates]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals#tagged_templates
[string]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/String
[rest parameters]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/rest_parameters
[rest element]: router-class/middleware/#chain
[reduce]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce
[books]: http://localhost:8000/books
[localhost]: http://localhost:8000/
[classes]: router-class/router/#classes
[instanceof]: router-class/router/#catch
[tostring]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/toString
[is array]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/isArray

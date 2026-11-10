---
type: exercise
title: A book search
weight: 4
author: Dr Graeme Stuart
---

In this exercise we will search the books by title or author, then split our code into modules.
The way we split it foreshadows the model-view-controller pattern we will use on the server.

<!--more-->

> You should have **data/books.json** and a working `formatBook` from the [previous exercise][previous exercise].
> If you are starting fresh, copy the **data** folder across and recreate `formatBook` and `formatList` as below.

## Learning objectives

By the end of this exercise you should be able to:

- Read a search query from the command line or from `prompt`
- Filter an array with `filter`, `toLowerCase` and `includes`
- Split a programme into modules with named exports and imports
- Explain how data, logic and output map onto model, view and controller

## A search query {#query}

Update **main.js** so that the search term comes from outside the file.

**main.js**
```JavaScript{hl_lines="9 13-14"}
import books from "./data/books.json" with { type: "json" };

function formatBook(book) {
    const { title, author, year } = book;
    return `${title}\nby ${author} (${year ?? "unknown"})`;
}

function formatList(books) {
    if (books.length === 0) return "No books match.";
    return books.map(formatBook).join("\n------------------------------\n");
}

const query = Deno.args[0] ?? prompt("Search for a book:") ?? "";
console.log(`Searching for "${query}" in ${books.length} books.`);
```

[`Deno.args`][deno args] is an array of the command-line arguments after the script name.
[`prompt`][prompt] asks a question in the terminal and waits for an answer.
If the user cancels, `prompt` returns `null`, so we fall back to an empty string with `??`.

Run it with an argument.

```bash {linenos=false}
deno main.js king
```

```plaintext {linenos=false}
Searching for "king" in 1000 books.
```

Run it with no argument, and type a word when asked.

```bash {linenos=false}
deno main.js
```

```plaintext {linenos=false}
Search for a book: potter
Searching for "potter" in 1000 books.
```

## Filter the books {#filter}

Add a `searchBooks` function and print the matches.

**main.js**
```JavaScript{hl_lines="13-19 22 24-31"}
import books from "./data/books.json" with { type: "json" };

function formatBook(book) {
    const { title, author, year } = book;
    return `${title}\nby ${author} (${year ?? "unknown"})`;
}

function formatList(books) {
    if (books.length === 0) return "No books match.";
    return books.map(formatBook).join("\n------------------------------\n");
}

function searchBooks(books, query) {
    const q = query.toLowerCase();
    return books.filter((book) => {
        return book.title.toLowerCase().includes(q) ||
            book.author.toLowerCase().includes(q);
    });
}

const query = Deno.args[0] ?? prompt("Search for a book:") ?? "";
const matches = searchBooks(books, query);

console.log(`\n==============================`);
console.log(`Search: "${query}" (${matches.length} matches)`);
console.log(`==============================`);
console.log(formatList(matches.slice(0, 5)));
if (matches.length > 5) {
    console.log(`\n…and ${matches.length - 5} more.`);
}
console.log(`==============================`);
```

[`filter`][array filter] keeps the books for which our function returns `true`.
We compare in lower case so that `"King"` and `"king"` match the same books.
[`includes`][includes] is true when the query appears anywhere in the title or author.
The [`||`][logical or] means a book matches if *either* field contains the query.

```bash {linenos=false}
deno main.js king
```

```plaintext {linenos=false}

==============================
Search: "king" (58 matches)
==============================
To Kill a Mockingbird
by Harper Lee (1960)
------------------------------
Mockingjay
by Suzanne Collins (2010)
------------------------------
Breaking Dawn
by Stephenie Meyer (2008)
------------------------------
The Shining
by Stephen King (1977)
------------------------------
Looking for Alaska
by John Green (2005)

…and 53 more.
==============================
```

We only show the first five matches, so a busy terminal stays readable.
Try `deno main.js xyzzy` too; you should see **No books match.**

> The function we pass to `filter` is an [*arrow function*][arrow functions].
> For a single expression you can drop the braces and the `return`, but the longer form is easier to read while we are learning.

## Split into modules {#modules}

**main.js** is doing three different jobs: holding the data, searching, and formatting the output.
As programmes grow, we split them into [modules][modules].

Create three new files, and move each job into one of them.

**data.js**
```JavaScript
import books from "./data/books.json" with { type: "json" };

export { books };
```

**search.js**
```JavaScript
export function searchBooks(books, query) {
    const q = query.toLowerCase();
    return books.filter((book) => {
        return book.title.toLowerCase().includes(q) ||
            book.author.toLowerCase().includes(q);
    });
}
```

**format.js**
```JavaScript
export function formatBook(book) {
    const { title, author, year } = book;
    return `${title}\nby ${author} (${year ?? "unknown"})`;
}

export function formatList(books) {
    if (books.length === 0) return "No books match.";
    return books.map(formatBook).join("\n------------------------------\n");
}
```

[`export`][export] makes a name available to other modules.
We use *named exports*, so each import has to say which names it wants.

Now **main.js** only wires the pieces together.

**main.js**
```JavaScript
import { books } from "./data.js";
import { searchBooks } from "./search.js";
import { formatList } from "./format.js";

const query = Deno.args[0] ?? prompt("Search for a book:") ?? "";
const matches = searchBooks(books, query);

console.log(`\n==============================`);
console.log(`Search: "${query}" (${matches.length} matches)`);
console.log(`==============================`);
console.log(formatList(matches.slice(0, 5)));
if (matches.length > 5) {
    console.log(`\n…and ${matches.length - 5} more.`);
}
console.log(`==============================`);
```

```bash {linenos=false}
deno main.js king
```

The output is unchanged.
That is the point of this step: we reorganised the code without changing what it does.

```plaintext {linenos=false, hl_lines="6-8"}
my-app/
├── data/
│   ├── BOOKS-ATTRIBUTION.md
│   └── books.json
├── data.js
├── format.js
├── main.js
└── search.js
```

### Data, output and logic {#mvc}

Look at what each file is responsible for.

- **data.js** looks after our data. On the server, code like this will live in a *model*.
- **format.js** turns data into something a human can read. On the server, that job belongs to a *view*.
- **main.js** takes the user's input, asks for the right data, and chooses how to show it. On the server, that job belongs to a *controller*.
- **search.js** is the logic of the search. Later, logic like this often sits with the model, or in a small helper of its own.

This is a first look at [*model-view-controller*][mvc glossary].
We will build it properly once we have a web server.
The habit to take forward is already here: give each file one job, and connect them with imports.

## Conclusion

We have created a simple programme that allows us to search a thousand books from the command line.
Our code is split into functions with clear jobs and we have divided our functions into modules so the codebase can be easily grown.
`data.js` holds the books, `search.js` filters them, `format.js` turns them into text, and `main.js` runs the programme.
The same split between data (models), presentation (views), and the overall logic of performing a task (controllers) is an extremely common architecture for web servers.

## Challenge: Stretch the search

- Search for `"rowling"` and for `"Rowling"`. Why do both work?
- Show all matches when there are ten or fewer, and keep the "…and N more" message otherwise.
- Add a default parameter to `formatList`, such as `formatList(books, limit = 5)`, and use it from **main.js**.
- Move `formatBook` into its own file, **formatBook.js**, and import it from **format.js**.

> Keep the four-file structure above before you move on.
> The next unit starts a web server, and we will meet **books.json** again there.

[previous exercise]: javascript-fundamentals/books-data/
[deno args]: https://docs.deno.com/api/deno/~/Deno.args
[prompt]: https://developer.mozilla.org/en-US/docs/Web/API/Window/prompt
[array filter]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/filter
[includes]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/includes
[logical or]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Logical_OR
[arrow functions]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions
[modules]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules
[export]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/export
[mvc glossary]: https://developer.mozilla.org/en-US/docs/Glossary/MVC

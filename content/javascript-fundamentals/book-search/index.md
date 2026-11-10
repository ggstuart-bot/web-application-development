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

- Filter an array with `filter`, `includes` and `toLowerCase`
- Write an arrow function and pass it to another function
- Use `if` to handle an empty result
- Read a search query from the command line or from `prompt`
- Split a programme into modules with named exports and imports
- Explain how data, logic and output map onto model, view and controller

## Filter the books {#filter}

We want only the books that match a search.
Arrays have a [`filter`][array filter] method for exactly this.

Update **main.js**, keeping `formatBook` and `formatList` as they are.

**main.js**
```JavaScript{hl_lines="12-14 16-17"}
import books from "./data/books.json" with { type: "json" };

function formatBook(book) {
    const { title, author, year } = book;
    return `${title}\nby ${author} (${year ?? "unknown"})`;
}

function formatList(books) {
    return books.map(formatBook).join("\n------------------------------\n");
}

function isMatch(book) {
    return book.title.includes("Hobbit");
}

const matches = books.filter(isMatch);
console.log(formatList(matches));
```

[`includes`][includes] is `true` when one string appears anywhere inside another, so `isMatch` returns `true` for any book with "Hobbit" in its title.
`filter` calls `isMatch` on every book and keeps the books for which it returns `true`.
Just like `map`, we pass the function without brackets and let `filter` call it.

```bash {linenos=false}
deno main.js
```

```plaintext {linenos=false}
The Hobbit
by J.R.R. Tolkien (1937)
------------------------------
The Hobbit: Graphic Novel
by Chuck Dixon, J.R.R. Tolkien (1989)
------------------------------
The History of the Hobbit, Part One: Mr. Baggins
by John D. Rateliff, J.R.R. Tolkien (2007)
```

## Arrow functions {#arrow-functions}

`isMatch` is only used in one place, we can write the function right where it is used instead.

**main.js**
```JavaScript{hl_lines="12-14"}
import books from "./data/books.json" with { type: "json" };

function formatBook(book) {
    const { title, author, year } = book;
    return `${title}\nby ${author} (${year ?? "unknown"})`;
}

function formatList(books) {
    return books.map(formatBook).join("\n------------------------------\n");
}

const matches = books.filter((book) => {
    return book.title.includes("Hobbit");
});
console.log(formatList(matches));
```

`(book) => { … }` is an [*arrow function*][arrow functions].
It is a function with no name: `book` is its parameter, and the body is the same as before.
Now the condition sits next to the `filter` that uses it.

> For a single expression you can drop the braces and the `return`, but the longer form is easier to read while we are learning.

```bash {linenos=false}
deno main.js
```

```plaintext {linenos=false}
The Hobbit
by J.R.R. Tolkien (1937)
------------------------------
The Hobbit: Graphic Novel
by Chuck Dixon, J.R.R. Tolkien (1989)
------------------------------
The History of the Hobbit, Part One: Mr. Baggins
by John D. Rateliff, J.R.R. Tolkien (2007)
```

The output hasn't changed.

## A search function {#search-books}

The word "Hobbit" is buried inside our code.
Let's wrap the search in a function, like `formatBook`, and pass the search term in.

**main.js**
```JavaScript{hl_lines="12-16 18-19"}
import books from "./data/books.json" with { type: "json" };

function formatBook(book) {
    const { title, author, year } = book;
    return `${title}\nby ${author} (${year ?? "unknown"})`;
}

function formatList(books) {
    return books.map(formatBook).join("\n------------------------------\n");
}

function searchBooks(books, query) {
    return books.filter((book) => {
        return book.title.includes(query);
    });
}

const query = "Hobbit";
const matches = searchBooks(books, query);
console.log(formatList(matches));
```

`searchBooks` takes two parameters, the books to search and the search term, and returns the matches.
Because our arrow function is written inside `searchBooks`, it can use the `query` parameter.
A separate `isMatch` function outside could not.

```bash {linenos=false}
deno main.js
```

```plaintext {linenos=false}
The Hobbit
by J.R.R. Tolkien (1937)
------------------------------
The Hobbit: Graphic Novel
by Chuck Dixon, J.R.R. Tolkien (1989)
------------------------------
The History of the Hobbit, Part One: Mr. Baggins
by John D. Rateliff, J.R.R. Tolkien (2007)
```

## Search by author {#logical-or}

Change the query to `"Orwell"`.
There are no matches, because Orwell is in the author, not the title.
Update `searchBooks` so it checks both.

**main.js**
```JavaScript{hl_lines="14-15 19"}
import books from "./data/books.json" with { type: "json" };

function formatBook(book) {
    const { title, author, year } = book;
    return `${title}\nby ${author} (${year ?? "unknown"})`;
}

function formatList(books) {
    return books.map(formatBook).join("\n------------------------------\n");
}

function searchBooks(books, query) {
    return books.filter((book) => {
        return book.title.includes(query) ||
            book.author.includes(query);
    });
}

const query = "Orwell";
const matches = searchBooks(books, query);
console.log(formatList(matches));
```

The [`||`][logical or] means a book matches if *either* field contains the query.

```bash {linenos=false}
deno main.js
```

```plaintext {linenos=false}
1984
by George Orwell (1949)
------------------------------
Animal Farm
by George Orwell (1945)
------------------------------
Animal Farm / 1984
by George Orwell (1950)
```

## Ignore capitals {#lower-case}

Change the query to `"orwell"`.
There are no matches again, because `includes` cares about capital letters.
Nobody wants to remember how to capitalise a search.

**main.js**
```JavaScript{hl_lines="13 15-16 20"}
import books from "./data/books.json" with { type: "json" };

function formatBook(book) {
    const { title, author, year } = book;
    return `${title}\nby ${author} (${year ?? "unknown"})`;
}

function formatList(books) {
    return books.map(formatBook).join("\n------------------------------\n");
}

function searchBooks(books, query) {
    const q = query.toLowerCase();
    return books.filter((book) => {
        return book.title.toLowerCase().includes(q) ||
            book.author.toLowerCase().includes(q);
    });
}

const query = "orwell";
const matches = searchBooks(books, query);
console.log(formatList(matches));
```

[`toLowerCase`][to lower case] gives us a lower case copy of a string.
We compare in lower case so that `"Orwell"` and `"orwell"` match the same books.
We only need to lower the query once, so we keep it in `q`.

```bash {linenos=false}
deno main.js
```

```plaintext {linenos=false}
1984
by George Orwell (1949)
------------------------------
Animal Farm
by George Orwell (1945)
------------------------------
Animal Farm / 1984
by George Orwell (1950)
```

## No matches {#no-matches}

Change the query to `"xyzzy"`.
Nothing matches, so `filter` gives us an empty array and our programme prints an empty line.
That looks like a mistake.

**main.js**
```JavaScript{hl_lines="9 21"}
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

const query = "xyzzy";
const matches = searchBooks(books, query);
console.log(formatList(matches));
```

An [`if`][if else] statement only runs its code when the condition is `true`.
The [`===`][strict equality] checks that `books.length` is exactly `0`.
When it is, `formatList` returns early, and the rest of the function doesn't run.

```bash {linenos=false}
deno main.js
```

```plaintext {linenos=false}
No books match.
```

## A search report {#report}

Change the query to `"king"`.
That gives a long list, so let's only show the first five matches, with a heading.

**main.js**
```JavaScript{hl_lines="21 24-31"}
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

const query = "king";
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

Everything here is something we have seen before.
We use template literals to print the heading, and `slice` to show only the first five matches, so a busy terminal stays readable.
This `if` uses braces, which can hold as many lines as we like.
The short form we used in `formatList` only holds a single statement.

```bash {linenos=false}
deno main.js
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

## A search term from the command line {#deno-args}

To search for something else we still have to edit our code.
It would be better to give the search term when we run the programme.

**main.js**
```JavaScript{hl_lines="21"}
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

const query = Deno.args[0];
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

[`Deno.args`][deno args] is an array of the command-line arguments after the script name, so `Deno.args[0]` is the first one.

Run it with an argument.

```bash {linenos=false}
deno main.js king
```

You should see the same report as before.
Try `deno main.js xyzzy` too; you should see **No books match.**

Now run it with no argument.

```bash {linenos=false}
deno main.js
```

```plaintext {linenos=false}
error: Uncaught (in promise) TypeError: Cannot read properties of undefined (reading 'toLowerCase')
    const q = query.toLowerCase();
                    ^
```

There is no first argument, so `Deno.args[0]` is `undefined`, and `undefined` has no `toLowerCase`.

## Ask for a search term {#prompt}

When there is no argument, let's ask for a search term instead.

**main.js**
```JavaScript{hl_lines="21"}
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

We met `??` when we [handled missing years][missing years].
If `Deno.args[0]` is `undefined`, we use the answer from [`prompt`][prompt] instead.
`prompt` asks a question in the terminal and waits for an answer.
If the user cancels, `prompt` returns `null`, so we fall back to an empty string with a second `??`.

Run it with no argument, and type a word when asked.

```bash {linenos=false}
deno main.js
```

```plaintext {linenos=false}
Search for a book: potter

==============================
Search: "potter" (10 matches)
==============================
…
```

Running it with an argument still works as before.

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
[array filter]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/filter
[includes]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/includes
[arrow functions]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions
[logical or]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Logical_OR
[to lower case]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/toLowerCase
[if else]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/if...else
[strict equality]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Strict_equality
[deno args]: https://docs.deno.com/api/deno/~/Deno.args
[missing years]: javascript-fundamentals/books-data/#missing-year
[prompt]: https://developer.mozilla.org/en-US/docs/Web/API/Window/prompt
[modules]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules
[export]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/export
[mvc glossary]: https://developer.mozilla.org/en-US/docs/Glossary/MVC

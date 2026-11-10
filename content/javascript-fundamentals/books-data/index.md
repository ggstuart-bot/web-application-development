---
type: exercise
title: Working with the books data
weight: 3
author: Dr Graeme Stuart
---

In this exercise we will load our books data and write functions that turn each book into a formatted string.
Everything is printed to the terminal; there is no HTML yet.

<!--more-->

> You should have Deno working from the [previous exercise][previous exercise].
> Create a new folder for this project and open it in VSCode.

## Learning objectives

By the end of this exercise you should be able to:

- Import a JSON file into a Deno programme
- Group related values in an object, and take them out with destructuring
- Build multi-line strings with template literals
- Turn an array into one string with `map` and `join`
- Write a function that returns a string, rather than printing it itself

## The books data {#data}

Our app will be about books.
The data we will use later already exists as a JSON file, so we can start practising with it now.

Create a folder called **data**.
Save {{<a href="files/books.json" download="books.json" >}}this list of books{{< /a >}} as **books.json** and {{<a href="files/BOOKS-ATTRIBUTION.txt" download="BOOKS-ATTRIBUTION.md" >}}this attribution file{{< /a >}} as **BOOKS-ATTRIBUTION.md**, both in the **data** folder.

**books.json** is a JSON array with one book on each line.
Each book has a title, an author, a year, and two image URLs.

### Crediting the data {#attribution}

The books come from an open dataset, and we are required to credit it.

Open **data/BOOKS-ATTRIBUTION.md**.

```markdown {linenos=false}
# Book data

- `books.json` is derived from goodbooks-10k by Zygmunt Zając
  (https://github.com/zygmuntz/goodbooks-10k), licensed CC BY-SA 4.0
  (https://creativecommons.org/licenses/by-sa/4.0/). Changes: the 1,000 most-rated books; titles,
  authors and years cleaned; multiple authors joined into one field. This derived data is shared
  under CC BY-SA 4.0.
- Cover images (thumbnailUrl, coverUrl) belong to their publishers; they are linked, not copied.
```

Keep that file next to the data.
We will meet the same files again when we build the database.

```plaintext {linenos=false, hl_lines="2-4"}
my-app/
├── data/
│   ├── BOOKS-ATTRIBUTION.md
│   └── books.json
└── …
```

## Import the books {#import}

Create **main.js** in the project root.

**main.js**
```JavaScript
import books from "./data/books.json" with { type: "json" };

console.log(books.length);
console.log(books[0]);
```

We import the JSON file directly.
The [`with { type: "json" }`][import attributes] part tells deno to read the file as JSON, so `books` is an array of objects.

Run it.

```bash {linenos=false}
deno main.js
```

```plaintext {linenos=false}
1000
{
  title: "The Hunger Games",
  author: "Suzanne Collins",
  year: 2008,
  thumbnailUrl: "https://images.gr-assets.com/books/1447303603m/2767052.jpg",
  coverUrl: "https://images.gr-assets.com/books/1447303603l/2767052.jpg"
}
```

We have 1,000 books.
The first one is an object with five properties.

## Format one book {#format-book}

`console.log` of an object is useful for debugging, but it is not nice to read.
Let's write a function that turns a book into a short description.

Update **main.js**.

**main.js**
```JavaScript{hl_lines="3-8 10-11"}
import books from "./data/books.json" with { type: "json" };

function formatBook(book) {
    const { title, author, year } = book;
    return `${title}
by ${author} (${year})`;
}

const first = books[0];
console.log(formatBook(first));
```

[`function`][functions] declares a named function.
Inside it we take `title`, `author` and `year` out of the book with [destructuring][destructuring].
The [template literal][template literals] (backticks) can span more than one line, and `${…}` drops a value into the string.

Run it.

```bash {linenos=false}
deno main.js
```

```plaintext {linenos=false}
The Hunger Games
by Suzanne Collins (2008)
```

`formatBook` *returns* a string.
It does not print anything itself.
That means we can use the same function later to build a longer report, or to send HTML, without changing it.

## A list of books {#format-list}

We want several books, with a line between them.

Add a `formatList` function and use it for the first three books.

**main.js**
```JavaScript{hl_lines="9-11 13-16"}
import books from "./data/books.json" with { type: "json" };

function formatBook(book) {
    const { title, author, year } = book;
    return `${title}
by ${author} (${year})`;
}

function formatList(books) {
    return books.map(formatBook).join("\n------------------------------\n");
}

const sample = books.slice(0, 3);
console.log("==============================");
console.log(formatList(sample));
console.log("==============================");
```

[`map`][array map] calls `formatBook` on every book and gives us an array of strings.
[`join`][array join] joins those strings into one, with a separator between them.
[`slice`][array slice] gives us just the first three books, so the terminal stays readable.

```bash {linenos=false}
deno main.js
```

```plaintext {linenos=false}
==============================
The Hunger Games
by Suzanne Collins (2008)
------------------------------
Harry Potter and the Sorcerer's Stone
by J.K. Rowling (1997)
------------------------------
Twilight
by Stephenie Meyer (2005)
==============================
```

The `====` lines are just characters in a string, like the separators we will later use when our server logs each request.

> Try `books.slice(0, 10)` or `books` on its own.
> Put it back to `3` before moving on.

## Missing years {#missing-year}

Some books have no year.
Their `year` is `null`.
Book 267 in our list is one of them.

Update `formatBook` so that a missing year becomes `"unknown"`, then print that book.

**main.js**
```JavaScript{hl_lines="4-5"}
import books from "./data/books.json" with { type: "json" };

function formatBook(book) {
    const { title, author, year } = book;
    return `${title}
by ${author} (${year ?? "unknown"})`;
}

function formatList(books) {
    return books.map(formatBook).join("\n------------------------------\n");
}

console.log(formatBook(books[266]));
```

[`??`][nullish coalescing] is the *nullish coalescing* operator.
If `year` is `null` (or `undefined`), we use `"unknown"`.
Otherwise we use `year`.

Run it.

```bash {linenos=false}
deno main.js
```

```plaintext {linenos=false}
Twilight: The Complete Illustrated Movie Companion
by Mark Cotta Vaz (unknown)
```

> Arrays are numbered from zero, so `books[266]` is the 267th book.

## Conclusion

We loaded 1,000 books from a JSON file and wrote two small functions that turn them into readable text.
`formatBook` returns a string for one book, and `formatList` joins many of them with a separator.
The same data file will feed our database later.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="2-4 6"}
my-app/
├── data/
│   ├── BOOKS-ATTRIBUTION.md
│   └── books.json
└── main.js
```

## Challenge: Experiment with the report

- Change the separator to `"\n****\n"` or a blank line.
- In `formatBook`, put the year on its own line.
- Pass `books.filter((book) => book.year < 1950)` into `formatList` instead of `sample`. How many are there?
- Use a [ternary][conditional operator] instead of `??` to show `"unknown"`.

> Put **main.js** back to the version above before moving on.

[previous exercise]: javascript-fundamentals/deno-setup/
[import attributes]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import/with
[functions]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Functions
[destructuring]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring
[template literals]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals
[array map]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map
[array join]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/join
[array slice]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/slice
[nullish coalescing]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing
[conditional operator]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Conditional_operator

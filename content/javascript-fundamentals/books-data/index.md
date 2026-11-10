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
- Read the properties of an object, and take them out with destructuring
- Build strings with `+`, and then more clearly with template literals
- Write a function that returns a string, rather than printing it itself
- Turn an array into one string with `slice`, `map` and `join`
- Replace a missing value with `??`

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

## Properties of a book {#properties}

`console.log` of an object is useful for debugging, but it is not nice to read.
We only want some of the properties, so let's take them out of the object one at a time.

Update **main.js**.

**main.js**
```JavaScript{hl_lines="3-6"}
import books from "./data/books.json" with { type: "json" };

const book = books[0];
console.log(book.title);
console.log(book.author);
console.log(book.year);
```

We give the first book a name, `book`, so we don't have to write `books[0]` every time.
A dot followed by a property name, such as `book.title`, gives us the value of that [property][property accessors].

Run it.

```bash {linenos=false}
deno main.js
```

```plaintext {linenos=false}
The Hunger Games
Suzanne Collins
2008
```

## Joining strings {#concatenation}

Three separate lines are not much of a description.
We want one line, with some words of our own in between the values.

**main.js**
```JavaScript{hl_lines="4"}
import books from "./data/books.json" with { type: "json" };

const book = books[0];
console.log(book.title + " by " + book.author + " (" + book.year + ")");
```

When either side is a string, [`+`][concatenation] joins the two sides together into one string.
The number `2008` is turned into a string for us.

```bash {linenos=false}
deno main.js
```

```plaintext {linenos=false}
The Hunger Games by Suzanne Collins (2008)
```

It works, but it is hard to read.
The string is chopped into pieces, and it is easy to lose a space or a quote mark in among all the `+` signs.

## Template literals {#template-literals}

A [template literal][template literals] lets us write the whole string in one piece.

**main.js**
```JavaScript{hl_lines="4"}
import books from "./data/books.json" with { type: "json" };

const book = books[0];
console.log(`${book.title}\nby ${book.author} (${book.year})`);
```

The template literal (backticks) defines the output string and `${…}` drops dynamic values into the string.
Compare it with the previous version.
The text between the values reads exactly as it will be printed, so the spaces and brackets are easy to see.

> Notice we include a newline character `\n` in the template, so the author goes on a second line.
> In fact, template literals can be split over multiple lines like this.
> ```JavaScript {hl_lines="2-3"}
> const book = books[0];
> console.log(`${book.title}
> by ${book.author} (${book.year})`);
> ```
> We use the newline character to avoid the confusing indentation.

```bash {linenos=false}
deno main.js
```

```plaintext {linenos=false}
The Hunger Games
by Suzanne Collins (2008)
```

## Destructuring {#destructuring}

Our template still says `book.` three times.
That is noise; the interesting part is the three values.

**main.js**
```JavaScript{hl_lines="4-5"}
import books from "./data/books.json" with { type: "json" };

const book = books[0];
const { title, author, year } = book;
console.log(`${title}\nby ${author} (${year})`);
```

The curly brackets on the left take `title`, `author` and `year` out of `book`, and make a `const` for each of them.
This is called [destructuring][destructuring].
It does the same job as writing `const title = book.title;` and so on, in one line.

Now the template reads almost exactly like the output.

```bash {linenos=false}
deno main.js
```

```plaintext {linenos=false}
The Hunger Games
by Suzanne Collins (2008)
```

The output hasn't changed, only the code.

> We could destructure `books[0]` directly, with `const { title, author, year } = books[0];`.
> We keep the `book` variable because it makes the next step easier.

## A reusable function {#format-book}

Our code describes the first book only.
To describe another book, we would have to copy those lines and change `books[0]`.
Let's write a function that turns any book into a short description.

**main.js**
```JavaScript{hl_lines="3-6 8-9"}
import books from "./data/books.json" with { type: "json" };

function formatBook(book) {
    const { title, author, year } = book;
    return `${title}\nby ${author} (${year})`;
}

console.log(formatBook(books[0]));
console.log(formatBook(books[1]));
```

[`function`][functions] declares a named function.
`book` is now a *parameter*: each time we call `formatBook`, it is given whichever book we pass in.
The body is the two lines we already had, except that the template is returned instead of printed.

Run it.

```bash {linenos=false}
deno main.js
```

```plaintext {linenos=false}
The Hunger Games
by Suzanne Collins (2008)
Harry Potter and the Sorcerer's Stone
by J.K. Rowling (1997)
```

`formatBook` *returns* a string.
It does not print anything itself.
That means we can use the same function later to build a longer report, or to send HTML, without changing it.

## A few books {#slice}

Calling `formatBook` once for each book by hand won't get us very far with 1,000 books.
Let's take the first three and loop over them.

**main.js**
```JavaScript{hl_lines="8-11"}
import books from "./data/books.json" with { type: "json" };

function formatBook(book) {
    const { title, author, year } = book;
    return `${title}\nby ${author} (${year})`;
}

const sample = books.slice(0, 3);
for (const book of sample) {
    console.log(formatBook(book));
}
```

[`slice`][array slice] gives us a new array with just the first three books, so the terminal stays readable.
The `for...of` loop is the same one we used in the [previous exercise][previous exercise].

```bash {linenos=false}
deno main.js
```

```plaintext {linenos=false}
The Hunger Games
by Suzanne Collins (2008)
Harry Potter and the Sorcerer's Stone
by J.K. Rowling (1997)
Twilight
by Stephenie Meyer (2005)
```

The books run into each other.
We could print a separator inside the loop, but then we would get one after the last book too.
And, just like `formatBook`, we would rather build a string than print as we go.

## Mapping books to strings {#map}

Arrays have a [`map`][array map] method, which calls a function on every item and gives us a new array of the results.

**main.js**
```JavaScript{hl_lines="9"}
import books from "./data/books.json" with { type: "json" };

function formatBook(book) {
    const { title, author, year } = book;
    return `${title}\nby ${author} (${year})`;
}

const sample = books.slice(0, 3);
console.log(sample.map(formatBook));
```

Notice that we pass `formatBook` without brackets.
We are not calling it ourselves; we are handing the function to `map`, and `map` calls it once for each book.

```bash {linenos=false}
deno main.js
```

```plaintext {linenos=false}
[
  "The Hunger Games\nby Suzanne Collins (2008)",
  "Harry Potter and the Sorcerer's Stone\nby J.K. Rowling (1997)",
  "Twilight\nby Stephenie Meyer (2005)"
]
```

We have an array of three strings.
Because we logged the whole array, deno shows each string in quotes, with the newline character as `\n`.

## A list of books {#format-list}

We want one string, with a line between the books.

Add a `formatList` function and use it for the first three books.

**main.js**
```JavaScript{hl_lines="8-10 13-15"}
import books from "./data/books.json" with { type: "json" };

function formatBook(book) {
    const { title, author, year } = book;
    return `${title}\nby ${author} (${year})`;
}

function formatList(books) {
    return books.map(formatBook).join("\n------------------------------\n");
}

const sample = books.slice(0, 3);
console.log("==============================");
console.log(formatList(sample));
console.log("==============================");
```

[`join`][array join] joins the strings from `map` into one, with a separator *between* them, so there is no separator after the last book.
`formatList` returns a string, just like `formatBook`.

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
```JavaScript{hl_lines="5"}
import books from "./data/books.json" with { type: "json" };

function formatBook(book) {
    const { title, author, year } = book;
    return `${title}\nby ${author} (${year ?? "unknown"})`;
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
[property accessors]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Property_accessors
[concatenation]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Addition#string_concatenation
[template literals]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Template_literals
[destructuring]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring
[functions]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Functions
[array slice]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/slice
[array map]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/map
[array join]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/join
[nullish coalescing]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing
[conditional operator]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Conditional_operator

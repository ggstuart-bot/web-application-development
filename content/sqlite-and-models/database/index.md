---
type: exercise
title: A books database
weight: 9
author: Dr Graeme Stuart
---

In this exercise we will move our books into a SQLite database.
Our books page will look the same, but the books will come from the database, sorted by title.

<!--more-->

> You should have a books page showing the first 16 books from the [previous exercise][previous exercise].
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

Our books model reads every book from a JSON file, and then throws most of them away.
In this exercise we will store the books in a database, and ask it for only the books we need.
By the end of this exercise you should be able to:

- Add the `@db/sqlite` package and explain why its tasks need `-A`
- Create a table with tasks that we can run again whenever we need a fresh database
- Seed the table from a JSON file
- Write model functions with prepared statements and named parameters
- Change where our data lives without changing our controllers or views
- Sort rows with `ORDER BY`, and show a default value when some data are missing

## Setup

Start the server in development mode.

```bash {linenos=false}
deno task dev
```

Open [http://localhost:8000/books][books].
You should see the first 16 books, starting with **The Hunger Games**.

## Adding SQLite {#sqlite}

[SQLite][sqlite] is a small database which keeps everything in a single file.
We will use the [@db/sqlite][db sqlite] package from JSR to work with it.

We added a package in the same way when we [served static files][jsr imports].
This time we ask for a particular version, so that everyone gets the same one.

```bash {linenos=false}
deno add jsr:@db/sqlite@0.12
```

```plaintext {linenos=false}
Add jsr:@db/sqlite@0.12.0
```

> You will also see a long list of downloads, and **deno.lock** will change.
> Commit both files as usual.

**deno.json**
```json{hl_lines="7"}
{
    "tasks": {
        "serve": "deno -A tasks/server.js",
        "dev": "deno --watch -A tasks/server.js"
    },
    "imports": {
        "@db/sqlite": "jsr:@db/sqlite@^0.12.0",
        "@std/http": "jsr:@std/http@^1.1.4"
    }
}
```

## Opening the database {#db}

Create **db.js** in the **server/models** folder, next to our books model.

**server/models/db.js**
```JavaScript
import { Database } from "@db/sqlite";

export const db = new Database("db/app.db");
```

This opens the database file **db/app.db**, creating it if it doesn't exist.
We export `db` so that other modules can use it.

> The database is opened once, the first time any module imports **db.js**.
> Every module that imports it after that gets the same `db`.

The path **db/app.db** is relative to the folder we run deno from.
So our database tasks must always run from the project root.
`deno task` takes care of this for us, because it runs tasks from the folder containing **deno.json**.

> If you see `SqliteError: 14: unable to open database file`, you are probably running a script from the wrong folder.
> Use `deno task` instead.

The database will be built from **db/books.json**, so we can always make it again.
We don't want it in our git repository.
Create a file called **.gitignore** in the root of the project.

**.gitignore**
```plaintext
db/app.db
```

A [**.gitignore**][gitignore] file lists files which git should ignore.
**db/app.db** won't appear in the **source control** panel.

## Creating the table {#create-table}

A database stores data in *tables*.
Each table has *columns*, and each book will be a *row*.
Create **tasks/db-create.js** to create a **books** table.

**tasks/db-create.js**
```JavaScript
import { db } from "../server/models/db.js";

db.exec("DROP TABLE IF EXISTS books");

db.exec(`
    CREATE TABLE books (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT NOT NULL,
        author TEXT NOT NULL,
        year INTEGER,
        thumbnailUrl TEXT,
        coverUrl TEXT
    )
`);
```

`db.exec` runs some SQL.
The [`CREATE TABLE`][create table] statement lists the columns and their types.

- `id` is an `INTEGER PRIMARY KEY`, a unique number for each book. `AUTOINCREMENT` means the database chooses it for us.
- `title` and `author` are `TEXT NOT NULL`, so every book must have them.
- `year`, `thumbnailUrl` and `coverUrl` are allowed to be empty (`NULL`).

`DROP TABLE IF EXISTS books` deletes the table first, along with all its data.
So we can run this task again whenever we want to start from scratch.

Add a task for it to **deno.json**.

**deno.json**
```json{hl_lines="4-5"}
{
    "tasks": {
        "serve": "deno -A tasks/server.js",
        "dev": "deno --watch -A tasks/server.js",
        "db:create": "deno -A tasks/db-create.js"
    },
    "imports": {
        "@db/sqlite": "jsr:@db/sqlite@^0.12.0",
        "@std/http": "jsr:@std/http@^1.1.4"
    }
}
```

Run it.

```bash {linenos=false}
deno task db:create
```

```plaintext {linenos=false}
Task db:create deno -A tasks/db-create.js
Downloading https://github.com/denodrivers/sqlite3/releases/download/0.12.0/libsqlite3.so
```

A file called **app.db** has appeared in the **db** folder.

> SQLite isn't written in JavaScript.
> The first time it runs, @db/sqlite downloads a SQLite library for your operating system (that's the **Downloading** line, and the file name will depend on your system) and calls it directly using [FFI][ffi].
> That needs several [permissions][permissions]: the network, the file system and FFI itself.
> `-A` allows them all, just as it does for our server tasks.

## Creating books {#models}

Now we need some code to put books into the database.
That's a job for our [books model][models].
Add a `createBook` function to **server/models/books.js**.

**server/models/books.js**
```JavaScript{hl_lines="2-10"}
import books from "../../db/books.json" with { type: "json" };
import { db } from "./db.js";

export function createBook({ title, author, year, thumbnailUrl, coverUrl }) {
    return db.prepare(`
        INSERT INTO books(title, author, year, thumbnailUrl, coverUrl)
        VALUES (:title, :author, :year, :thumbnailUrl, :coverUrl)
        RETURNING id, title, author, year, thumbnailUrl, coverUrl
    `).get({ title, author, year, thumbnailUrl, coverUrl });
}

// Placeholder: this will need to change when we add search
export function getBooks(limit) {
    return books.slice(0, limit);
}
```

The model imports `db` from **db.js**, next to it in the **models** folder.

`createBook` takes a book and inserts it into the **books** table.
We take the book's properties out with [destructuring][destructuring], just as we did with the options for `render`.

`db.prepare` creates a [*prepared statement*][prepared statement] from our SQL.
The `:title`, `:author` and so on are *named parameters*.
`.get()` runs the statement, filling in each parameter with the matching property of the object we pass it.

> We never paste values into the SQL string ourselves.
> The database keeps the values separate from the SQL, so a book called `"); DROP TABLE books; --` is just a strange title.
> Pasting values into SQL is how [SQL injection][sql injection] happens, and we will come back to it.

[`RETURNING`][returning] gives us back the new row, including the `id` the database chose.
We list the columns we want, rather than asking for everything, so we only get what we ask for, even if the table gains more columns later.

`getBooks` still reads **db/books.json**, so the books page is unchanged.

## Seeding the database {#seed}

Filling a new database with starting data is called *seeding*.
Create **tasks/db-seed.js**.

**tasks/db-seed.js**
```JavaScript
import books from "../db/books.json" with { type: "json" };
import { createBook } from "../server/models/books.js";

for (const book of books) {
    createBook(book);
}
```

We import **books.json** just as our model does, so `books` is an array of objects.
Then we loop over the books with [`for...of`][for of] and create each one with our model.

Add a `db:seed` task, and a `db:reset` task which runs `db:create` and then `db:seed`.

**deno.json**
```json{hl_lines="5-7"}
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
        "@std/http": "jsr:@std/http@^1.1.4"
    }
}
```

Run the seed task.

```bash {linenos=false}
deno task db:seed
```

```plaintext {linenos=false}
Task db:seed deno -A tasks/db-seed.js
```

No news is good news.
Now run it again.

```bash {linenos=false}
deno task db:seed
```

Our table now has 2,000 rows, every book twice.
Seeding adds to whatever is already there.

That's what `db:reset` is for.
It drops the table, creates it again and seeds it once.

> Running `db:create` twice is safe, because it drops the table and creates it again.
> However many times we run it, we end up with the same empty table.
> An operation like this is called [*idempotent*][idempotent].
> `db:seed` is not idempotent, running it twice gave us 2,000 rows.
> That's why `db:reset` runs `db:create` before `db:seed`, so the whole reset is safe to repeat.
> Idempotency is an important concept in HTTP, and we will come across it again later.

```bash {linenos=false}
deno task db:reset
```

```plaintext {linenos=false}
Task db:reset deno task db:create && deno task db:seed
Task db:create deno -A tasks/db-create.js
Task db:seed deno -A tasks/db-seed.js
```

We are back to 1,000 books.

> `&&` runs the second command only if the first one succeeded.

The books are in the database now, but the books page still reads them from **db/books.json**.

## Reading the books from the database {#get-books}

Update `getBooks` in **server/models/books.js** to ask the database for the books.

**server/models/books.js**
```JavaScript{hl_lines="13-16"}
import { db } from "./db.js";

export function createBook({ title, author, year, thumbnailUrl, coverUrl }) {
    return db.prepare(`
        INSERT INTO books(title, author, year, thumbnailUrl, coverUrl)
        VALUES (:title, :author, :year, :thumbnailUrl, :coverUrl)
        RETURNING id, title, author, year, thumbnailUrl, coverUrl
    `).get({ title, author, year, thumbnailUrl, coverUrl });
}

// Placeholder: this will need to change when we add search
export function getBooks(limit) {
    return db.prepare(`
        SELECT id, title, author, year, thumbnailUrl, coverUrl FROM books
        LIMIT :limit
    `).all({ limit });
}
```

[`SELECT`][select] reads rows from the **books** table, and `LIMIT :limit` asks for no more than `limit` rows.
`.all()` fills in the `:limit` parameter from the object we pass it, and returns the rows as an array of objects.

The model no longer imports **db/books.json**, and `getBooks` keeps its search placeholder.

> Make sure you have run `deno task db:reset`, so that the database has the books in it.

Refresh [http://localhost:8000/books][books].
The page looks just as it did, with the same 16 books.

Our books controller and our views haven't changed at all.
They still call `getBooks(PAGE_SIZE)` and get back an array of books, so they don't know or care that the books now come from a database.

**This is the payoff of keeping our data in a model.**
We changed where the data lives in one place, and nothing else had to change.

## Sorting by title {#order-by}

A database doesn't promise to return rows in any particular order.
We happen to get our books in the order we seeded them, but that could change.

Add [`ORDER BY title`][order by] to the query in **server/models/books.js**, before `LIMIT`.

**server/models/books.js**
```JavaScript{hl_lines="15"}
import { db } from "./db.js";

export function createBook({ title, author, year, thumbnailUrl, coverUrl }) {
    return db.prepare(`
        INSERT INTO books(title, author, year, thumbnailUrl, coverUrl)
        VALUES (:title, :author, :year, :thumbnailUrl, :coverUrl)
        RETURNING id, title, author, year, thumbnailUrl, coverUrl
    `).get({ title, author, year, thumbnailUrl, coverUrl });
}

// Placeholder: this will need to change when we add search
export function getBooks(limit) {
    return db.prepare(`
        SELECT id, title, author, year, thumbnailUrl, coverUrl FROM books
        ORDER BY title
        LIMIT :limit
    `).all({ limit });
}
```

The database sorts all the books by title, and then gives us the first 16.

Refresh [http://localhost:8000/books][books].
The page now starts at the beginning of the alphabet, and the same books appear every time.

> Numbers and punctuation come before letters, so the first book is **'Salem's Lot**, followed by **11/22/63**, **1776** and **1984**.

But some of the cards are broken.
**1st to Die**, **2001: A Space Odyssey**, **A Beautiful Mind**, **A Bend in the Road** and **A Child Called "It"** have no cover, just a broken image.

{{<figure src="images/broken-covers.webp" alt="The books page sorted by title, starting at 'Salem's Lot, where the cards for 1st to Die, 2001: A Space Odyssey, A Beautiful Mind and A Bend in the Road show a broken image where the cover should be" caption="Sorting by title brings some books with no cover onto the page.">}}

We noted when we [added covers][covers] that some books have no cover.
None of the first 16 books in **books.json** were like that, but sorting has brought five of them onto our page.
View the page source (**Ctrl + U**) and find **1st to Die**.

```html {linenos=false}
<img class="cover" src="null" alt="" width="200" height="300" loading="lazy">
```

Its `coverUrl` is `null`, and the template literal has turned that into the text `null`.
The browser asks for a cover at **null**, which doesn't exist.

## A default cover {#default-cover}

A book with no cover should still look like a book.

Save {{<a href="files/default-cover.svg" download="default-cover.svg" >}}this default cover{{< /a >}} as **default-cover.svg** in the **public** folder.
Click [http://localhost:8000/assets/default-cover.svg][default cover] to check that it is served.

Now update **server/views/books.js** to use it when a book has no cover.

**server/views/books.js**
```JavaScript{hl_lines="1-2 17 20"}
const DEFAULT_COVER = "/assets/default-cover.svg";

export function booksView({ data: { books } }) {
    return `
        <section>
            <header>
                <h1>Books</h1>
            </header>
            <ul class="grid">
                ${books.map(bookCard).join("")}
            </ul>
        </section>
    `;
}

function bookCard(book) {
    const cover = book.coverUrl ?? DEFAULT_COVER;
    return `
        <li class="card book">
            <img class="cover" src="${cover}" alt="" width="200" height="300" loading="lazy">
            <div>
                <h2>${book.title}</h2>
                <p>${book.author} (${book.year})</p>
            </div>
        </li>
    `;
}
```

`DEFAULT_COVER` is the address of our default cover, a constant in capitals like [`PAGE_SIZE`][page size].
It isn't exported, because only this module uses it.

`??` is the [nullish coalescing operator][nullish coalescing], which we used for missing years in the terminal.
If `book.coverUrl` is `null` (or `undefined`), `cover` is `DEFAULT_COVER`.
Otherwise, `cover` is `book.coverUrl`.

Refresh [http://localhost:8000/books][books].
Every card now has a cover.

{{<figure src="images/default-cover.webp" alt="The same books page, where 1st to Die, 2001: A Space Odyssey, A Beautiful Mind and A Bend in the Road now show a plain grey book as their cover" caption="Books without a cover now get the default one.">}}

## Conclusion

We have added SQLite to our project and created a **books** table with a task we can run again at any time.
We seeded it from our JSON file, and changed our books model to read from the database instead.
We used prepared statements and named parameters, so the rest of our code never needs to write SQL.
Our controller and views didn't change at all.
Then we sorted the books by title, and gave the books with no cover a default one.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="5 9 20 30-31 33"}
my-app/
├── .vscode/
│   └── settings.json
├── db/
│   ├── app.db
│   ├── BOOKS-ATTRIBUTION.md
│   └── books.json
├── public/
│   ├── default-cover.svg
│   ├── icon.svg
│   └── styles.css
├── server/
│   ├── controllers/
│   │   ├── books.js
│   │   ├── home.js
│   │   ├── notFound.js
│   │   └── static.js
│   ├── models/
│   │   ├── books.js
│   │   └── db.js
│   ├── tools/
│   │   └── render.js
│   ├── views/
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

> **db/app.db** is in the folder, but not in git.

## Challenge: Experiment with the database

- In `getBooks`, temporarily add `WHERE year < 1950` after `FROM books` and refresh [http://localhost:8000/books][books]. Try other conditions, such as `WHERE author = 'J.K. Rowling'`.
- Try `ORDER BY year` or `ORDER BY author` instead of `ORDER BY title`. What does `ORDER BY title DESC` do?
- Open **db/books.json** and look for another book with no cover images.

> Make sure **server/models/books.js** is back to the version above before moving on.

Finally, something to think about.
Our books page only ever shows the first 16 books.
How could a visitor see the other 984?

[previous exercise]: sqlite-and-models/books-page/
[books]: http://localhost:8000/books
[sqlite]: https://www.sqlite.org/
[db sqlite]: https://jsr.io/@db/sqlite
[jsr imports]: http-servers/static-files/#jsr-imports
[gitignore]: https://git-scm.com/docs/gitignore
[create table]: https://www.sqlite.org/lang_createtable.html
[ffi]: https://docs.deno.com/runtime/fundamentals/ffi/
[permissions]: http-servers/deno-serve/#permissions
[models]: sqlite-and-models/books-page/#models
[destructuring]: views-and-controllers/controllers/#title
[prepared statement]: https://en.wikipedia.org/wiki/Prepared_statement
[sql injection]: https://en.wikipedia.org/wiki/SQL_injection
[returning]: https://www.sqlite.org/lang_returning.html
[for of]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...of
[idempotent]: https://developer.mozilla.org/en-US/docs/Glossary/Idempotent
[select]: https://www.sqlite.org/lang_select.html
[order by]: https://www.sqlite.org/lang_select.html#the_order_by_clause
[covers]: sqlite-and-models/books-page/#cover
[default cover]: http://localhost:8000/assets/default-cover.svg
[page size]: sqlite-and-models/books-page/#page-size
[nullish coalescing]: javascript-fundamentals/books-data/#missing-year

---
type: exercise
title: A books database
weight: 8
author: Dr Graeme Stuart
---

In this exercise we will create a SQLite database and fill it with books.
Our website won't change at all, but we will be able to list the books in the terminal.

<!--more-->

> You should have a home page and a not-found page, each with its own controller, from the [previous exercise][previous exercise].
> Before you start, make sure everything is committed to your git repository.

## Learning objectives

*Bookshelves* needs books.
In this exercise we will store them in a database, ready to show them on the books page in the next exercise.
By the end of this exercise you should be able to:

- Add the `@db/sqlite` package and explain why its tasks need `-A`
- Create a table with tasks that we can run again whenever we need a fresh database
- Seed the table from a JSON file, and explain why we credit the source of the data
- Write a *model* with prepared statements and named parameters
- Explain where models fit in model-view-controller

## The data {#data}

We need some books.
We will use a list of the 1,000 most-rated books on Goodreads, from a public dataset called [goodbooks-10k][goodbooks].

Create a folder called **db** in the root of the project.
Save {{<a href="files/books.json" download="books.json" >}}this list of books{{< /a >}} as **books.json** and {{<a href="files/BOOKS-ATTRIBUTION.txt" download="BOOKS-ATTRIBUTION.md" >}}this attribution file{{< /a >}} as **BOOKS-ATTRIBUTION.md**, both in the **db** folder.

**books.json** is a JSON array with one book on each line.
Here's the first one.

**db/books.json**
```json {linenos=false}
{ "title": "The Hunger Games", "author": "Suzanne Collins", "year": 2008, "thumbnailUrl": "https://images.gr-assets.com/books/1447303603m/2767052.jpg", "coverUrl": "https://images.gr-assets.com/books/1447303603l/2767052.jpg" }
```

Each book has a `title`, an `author` and a `year`, plus links to a small and a large cover image.
Some books have no cover images, so their `thumbnailUrl` and `coverUrl` are `null`.

### Crediting the data {#attribution}

We didn't create this data, so we must say where it came from.

**db/BOOKS-ATTRIBUTION.md**
```markdown
# Book data

- `books.json` is derived from goodbooks-10k by Zygmunt Zając
  (https://github.com/zygmuntz/goodbooks-10k), licensed CC BY-SA 4.0
  (https://creativecommons.org/licenses/by-sa/4.0/). Changes: the 1,000 most-rated books; titles,
  authors and years cleaned; multiple authors joined into one field. This derived data is shared
  under CC BY-SA 4.0.
- Cover images (thumbnailUrl, coverUrl) belong to their publishers; they are linked, not copied.
```

The original dataset is shared under the [CC BY-SA 4.0][cc by-sa] licence.
*BY* means anyone who uses it must credit the author.
*SA* (share alike) means anything we make from it must be shared under the same licence.
We changed the data, so the attribution says what we changed, and our version is shared under the same licence.

> Crediting your sources is good practice anyway.
> With data like this, it's also a condition of using it at all.

> The cover images belong to their publishers.
> We only link to them, we don't copy them into our project.

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

Create a folder called **models** inside the **server** folder, with a file called **db.js**.

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

The database is built from **books.json**, so we can always make it again.
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

## A books model {#models}

Now we need some code to put books into the database.
Create **server/models/books.js**.

**server/models/books.js**
```JavaScript
import { db } from "./db.js";

export function createBook({ title, author, year, thumbnailUrl, coverUrl }) {
    return db.prepare(`
        INSERT INTO books(title, author, year, thumbnailUrl, coverUrl)
        VALUES (:title, :author, :year, :thumbnailUrl, :coverUrl)
        RETURNING id, title, author, year, thumbnailUrl, coverUrl
    `).get({ title, author, year, thumbnailUrl, coverUrl });
}
```

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

### The M in MVC {#mvc}

**server/models/books.js** is a *model*.
Models are the M in [model-view-controller][mvc views].
We already have [views][mvc views] which present information and [controllers][mvc controllers] which handle requests.
A model looks after our data, and the rest of our code asks the model rather than talking to the database itself.

Just like views and controllers, a model is a set of related functions grouped into a module.

## Seeding the database {#seed}

Filling a new database with starting data is called *seeding*.
Create **tasks/db-seed.js**.

**tasks/db-seed.js**
```JavaScript
import books from "../db/books.json" with { type: "json" };
import { createBook } from "../server/models/books.js";

// Temporary: seed only 20 books until we introduce paging
for (const book of books.slice(0, 20)) {
    createBook(book);
}
```

We import **books.json** directly.
The [`with { type: "json" }`][import attributes] part tells deno to read the file as JSON, so `books` is an array of objects.
Then we loop over the books with [`for...of`][for of] and create each one with our model.

`books.slice(0, 20)` gives us only the first 20 books.
This is a deliberate, temporary limit.
Later, when we seed all 1,000, we will see why our pages need *paging*.

> The `Temporary:` comment marks a line we will change, like our placeholders.

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

Our table now has 40 rows, every book twice.
Seeding adds to whatever is already there.

That's what `db:reset` is for.
It drops the table, creates it again and seeds it once.

> Running `db:create` twice is safe, because it drops the table and creates it again.
> However many times we run it, we end up with the same empty table.
> An operation like this is called [*idempotent*][idempotent].
> `db:seed` is not idempotent, running it twice gave us 40 rows.
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

We are back to 20 books.

> `&&` runs the second command only if the first one succeeded.

## Listing the books {#get-books}

We can put books in, but we can't see them yet.
Add a `getBooks` function to **server/models/books.js**.

**server/models/books.js**
```JavaScript{hl_lines="11-16"}
import { db } from "./db.js";

export function createBook({ title, author, year, thumbnailUrl, coverUrl }) {
    return db.prepare(`
        INSERT INTO books(title, author, year, thumbnailUrl, coverUrl)
        VALUES (:title, :author, :year, :thumbnailUrl, :coverUrl)
        RETURNING id, title, author, year, thumbnailUrl, coverUrl
    `).get({ title, author, year, thumbnailUrl, coverUrl });
}

// Placeholder: this will need to change when we add search
export function getBooks() {
    return db.prepare(`
        SELECT id, title, author, year, thumbnailUrl, coverUrl FROM books
    `).all();
}
```

[`SELECT`][select] reads rows from a table, and `.all()` returns every row as an array of objects.

> Another placeholder.
> When we add search, `getBooks` will need to find only the matching books.

Our server doesn't use the model yet.
For now, create a small task to list the books in the terminal.

**tasks/books.js**
```JavaScript
import { getBooks } from "../server/models/books.js";

for (const book of getBooks()) {
    console.log(book.id, book.title, book.author, book.year);
}
```

Add it to **deno.json**.

**deno.json**
```json{hl_lines="7-8"}
{
    "tasks": {
        "serve": "deno -A tasks/server.js",
        "dev": "deno --watch -A tasks/server.js",
        "db:create": "deno -A tasks/db-create.js",
        "db:seed": "deno -A tasks/db-seed.js",
        "db:reset": "deno task db:create && deno task db:seed",
        "books": "deno -A tasks/books.js"
    },
    "imports": {
        "@db/sqlite": "jsr:@db/sqlite@^0.12.0",
        "@std/http": "jsr:@std/http@^1.1.4"
    }
}
```

**This task is temporary.**
In the next exercise we will show the books on the books page, and then we will delete **tasks/books.js** and remove the `books` task from **deno.json**.

Run it.

```bash {linenos=false}
deno task books
```

```plaintext {linenos=false}
Task books deno -A tasks/books.js
1 The Hunger Games Suzanne Collins 2008
2 Harry Potter and the Sorcerer's Stone J.K. Rowling 1997
3 Twilight Stephenie Meyer 2005
4 To Kill a Mockingbird Harper Lee 1960
5 The Great Gatsby F. Scott Fitzgerald 1925
6 The Fault in Our Stars John Green 2012
7 The Hobbit J.R.R. Tolkien 1937
8 The Catcher in the Rye J.D. Salinger 1951
9 Pride and Prejudice Jane Austen 1813
10 Angels & Demons Dan Brown 2000
11 The Diary of a Young Girl Anne Frank 1947
12 1984 George Orwell 1949
13 Divergent Veronica Roth 2011
14 Animal Farm George Orwell 1945
15 Harry Potter and the Prisoner of Azkaban J.K. Rowling 1999
16 Catching Fire Suzanne Collins 2009
17 The Kite Runner Khaled Hosseini 2003
18 The Girl with the Dragon Tattoo Stieg Larsson 2005
19 Harry Potter and the Chamber of Secrets J.K. Rowling 1998
20 The Fellowship of the Ring J.R.R. Tolkien 1954
```

Our 20 books, each with the `id` the database gave it.

## Conclusion

We have added SQLite to our project and created a **books** table with a task we can run again at any time.
We seeded it from a JSON file, credited under its CC BY-SA licence.
We wrote our first model, using prepared statements and named parameters, so the rest of our code never needs to write SQL.
Our website hasn't changed, but the books are ready for the books page.

Our project now looks like this.

```plaintext {linenos=false, hl_lines="4-7 16-18 27-29 31"}
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
│   │   ├── home.js
│   │   ├── notFound.js
│   │   └── static.js
│   ├── models/
│   │   ├── books.js
│   │   └── db.js
│   ├── tools/
│   │   └── render.js
│   ├── views/
│   │   ├── home.js
│   │   ├── layout.js
│   │   └── notFound.js
│   └── main.js
├── tasks/
│   ├── books.js
│   ├── db-create.js
│   ├── db-seed.js
│   └── server.js
├── .gitignore
├── deno.json
└── deno.lock
```

> **db/app.db** is in the folder, but not in git.

## Challenge: Experiment with the database

- Change `20` to `100` in **tasks/db-seed.js**, then run `deno task db:reset` and `deno task books`. Can you still read the whole list?
- In `getBooks`, temporarily add `WHERE year < 1950` after `FROM books` and run `deno task books` again. Try other conditions, such as `WHERE author = 'J.K. Rowling'`.
- Open **db/books.json** and look for a book with no cover images.

> Make sure **tasks/db-seed.js** and **server/models/books.js** are back to the versions above, then run `deno task db:reset` before moving on.

Finally, something to think about.
Our `books` task is the only thing that can see the books.
What would a controller and a view for the books page need to do?

[previous exercise]: views-and-controllers/controllers/
[goodbooks]: https://github.com/zygmuntz/goodbooks-10k
[cc by-sa]: https://creativecommons.org/licenses/by-sa/4.0/
[sqlite]: https://www.sqlite.org/
[db sqlite]: https://jsr.io/@db/sqlite
[jsr imports]: http-servers/static-files/#jsr-imports
[gitignore]: https://git-scm.com/docs/gitignore
[create table]: https://www.sqlite.org/lang_createtable.html
[ffi]: https://docs.deno.com/runtime/fundamentals/ffi/
[permissions]: http-servers/deno-serve/#permissions
[destructuring]: views-and-controllers/controllers/#title
[prepared statement]: https://en.wikipedia.org/wiki/Prepared_statement
[sql injection]: https://en.wikipedia.org/wiki/SQL_injection
[returning]: https://www.sqlite.org/lang_returning.html
[mvc views]: views-and-controllers/views/#mvc
[mvc controllers]: views-and-controllers/controllers/#mvc
[import attributes]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import/with
[for of]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...of
[select]: https://www.sqlite.org/lang_select.html
[idempotent]: https://developer.mozilla.org/en-US/docs/Glossary/Idempotent

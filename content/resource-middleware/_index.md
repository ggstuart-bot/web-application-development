---
type: unit
title: Resource middleware
weight: 110
---

In this unit we will put books on shelves, and take them off again, from the browser.
Along the way, the same few lines start to appear in controller after controller, and we will move them into middleware.

<!--more-->

First we will add a button for each shelf on a book's page, and a remove button on each book on a shelf's page.
Each new button starts with the not found page, then a stub controller that only says what it would have done, then the real thing.
Then our router will learn to run a chain of handlers for a route, and two new middleware, `withShelf` and `withBook`, will load the shelf and the book before the controller runs.
The unit ends with a button that deletes a shelf.

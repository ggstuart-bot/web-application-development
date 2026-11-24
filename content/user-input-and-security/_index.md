---
type: unit
title: User input and security
weight: 107
---

In this unit we will let our users type something into our application for the first time.
Anything a user types is untrusted, so security is the major theme.

<!--more-->

We will start by paging through the books, with a page number that the server reads from the query string in the URL.
Then we will add a search form that sends its query to the server in the same way.
Finally, we will see why everything we write into a page must be escaped, and protect every view with an `html` helper.

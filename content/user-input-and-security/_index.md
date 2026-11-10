---
type: unit
title: User input and security
weight: 107
---

In this unit we will let our users type something into our application for the first time.
Anything a user types is untrusted, so security is the major theme.

<!--more-->

We will add a search form that sends its query to the server in the URL, and read it with `searchParams`.
Then we will see why everything we write into a page must be escaped, and protect every view with an `html` helper.
The unit ends with paging through the books, another use of the query string.

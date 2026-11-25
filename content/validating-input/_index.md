---
type: unit
title: Validating input
weight: 111
---

In this unit we will check what our users type, before we let it into our database.
Anything a user sends us is untrusted, so the server must decide what to accept.

<!--more-->

First we will stop the new shelf form from creating a shelf with an empty, short or very long title, and show the user why their title was rejected.
Along the way we will write *validators*, small functions that each check one thing.
Then we will add a sign up page, and check the whole form against a *schema*, so that every field is checked and every problem is shown at once.

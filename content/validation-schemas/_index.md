---
type: unit
title: Validation schemas
weight: 112
---

In this unit we will put every form in our app onto a validation schema, checked by one piece of middleware.

<!--more-->

We will move the new shelf form onto a schema, then take the checking out of our controllers and into a `withValidation` middleware.
Then we will add a sign in page with a schema of its own, and reuse the new shelf form to rename a shelf.

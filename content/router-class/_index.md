---
type: unit
title: A router class
weight: 106
---

In this unit we will pay off some technical debt by refactoring our routing into a router class.
We will also consolidate the context pattern and write our first middleware.

<!--more-->

When we are done, we will register routes with the router, every controller will work with the same request context and we will have middleware to log every request and handle errors in one place.

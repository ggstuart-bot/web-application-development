---
type: unit
title: Privacy
weight: 116
---

In this unit we will let users decide who can see their profile and their shelves.
Profiles come first, then shelves.

<!--more-->

First we will give every user a public or private profile, and hide a private profile, and its avatar, from everyone but its owner.
Along the way, our permission checks get names of their own, in **tools/permissions.js**, and our `may` guards are made from them.
Then we will do the same for shelves: only public shelves are listed, a private shelf opens only for its owner, and you can share a shelf, or take it back.
By the end of the unit, the permissions pattern from [owners and guards][pattern] is complete.

[pattern]: owners-and-guards/owners/#pattern

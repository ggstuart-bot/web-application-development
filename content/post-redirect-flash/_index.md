---
type: unit
title: POST, redirect and flash
weight: 109
---

In this unit we will add the first form that changes our data.
Until now, every request has only read from the database.

<!--more-->

First we will add a form that makes a new shelf, and see why a form that changes data should be answered with a redirect.
Our router will learn to tell a `GET` from a `POST`, so each controller can do one job.
Then we will use a cookie to carry a short message across the redirect, and show it on the next page as a *flash message*.

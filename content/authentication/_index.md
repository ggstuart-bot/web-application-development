---
type: unit
title: Authentication
weight: 113
---

In this unit we will turn the sign up and sign in stubs into a real account system.
Users will be stored with hashed passwords, and signing in will be remembered with a session.

<!--more-->

First we will add a users table, hash passwords, and let the sign up form create an account.
A username that's already taken will be rejected with a message, not a server error.
Then we will check the password when signing in, see why a cookie that holds the user id is not safe, and replace it with a session.
By the end of the unit you can sign out, and a new user is signed in as soon as they sign up.

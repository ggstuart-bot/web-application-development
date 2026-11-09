---
type: exercise
title: Deno setup
weight: 2
author: Dr Graeme Stuart
---

The *deno* runtime lets us run JavaScript without a browser.
In this exercise we will install deno, write a short script and run it.

<!--more-->

## Learning objectives

By the end of this exercise you should be able to:

- Install Deno and check that it works
- Create a JavaScript file and run it with `deno`
- Use `console.log`, `const` and a `for...of` loop

## Installing deno

We will install deno from the integrated terminal in **Visual Studio Code**.

> The following instructions are suitable for the lab machines running windows.
> General instructions for installing deno can be found at [the deno website][deno].

Open VSCode and press **Ctrl + @** to open the integrated terminal.

> We will often use the terminal to run our code, so keep it open.

If the terminal shows "cmd.exe", click the dropdown and select **PowerShell** to switch to PowerShell.
Copy and paste the following command into the PowerShell terminal and press **Enter**:

```powershell {linenos=false}
irm https://deno.land/install.ps1 | iex
```

Wait for the installation to complete (this may take a few minutes).

### Verify the installation

We have seen the install fail on the lab machines in the past, so it's worth checking that it worked.

Type the following command into the integrated PowerShell terminal and press **Enter**:

```powershell {linenos=false}
deno --version
```

If you see output showing the Deno version (e.g. `deno 2.9.7`), you're ready to go.

> If not, close and restart VSCode and try again.

## Create and run your first script

Create a new folder for this project and open it in VSCode.
Create a file called **main.js** with a simple "hello world" example.

**main.js**
```JavaScript
console.log("Hello World");
```

Make sure you save the file (**Ctrl + S**).

Open the PowerShell terminal in VSCode (**Ctrl + @**) and run the script.

```powershell {linenos=false}
deno main.js
```

> We are running the **deno** programme and passing it the path to the JavaScript code we want to execute.

You should see the output printed in the terminal.

```plaintext {linenos=false}
Hello World
```

> If you don't see anything then ask your lab tutor to help.

Our project looks like this.

```plaintext {linenos=false, hl_lines="2"}
my-app/
└── main.js
```

## A list of items

Update **main.js** with the following code.

**main.js**
```JavaScript{hl_lines="1-3"}
const items = ["apples", "bananas", "cherries"];
for (const item of items) {
    console.log("logging", item);
}
```

[`const`][const] declares a name that we will not reassign.
`items` is an [array][array], an ordered list of values.
[`for...of`][for of] walks through each value in turn.

Run the programme again.

```powershell {linenos=false}
deno main.js
```

```plaintext {linenos=false}
logging apples
logging bananas
logging cherries
```

## Conclusion

We have installed deno and run a simple JavaScript programme.
We used `console.log` to write to the terminal, `const` to name a value, and `for...of` to walk through an array.

## Challenge: Try a while loop

- Rewrite the loop with a [`while`][while] loop and an index.
- Add a fourth item to the array and run the programme again.
- Change `"logging"` to something else and watch the output change.

> Put **main.js** back to the version above before moving on, or start a fresh folder for the next exercise.

[deno]: https://deno.com
[const]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/const
[array]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array
[for of]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/for...of
[while]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/while

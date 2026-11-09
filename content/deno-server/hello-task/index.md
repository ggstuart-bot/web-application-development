---
type: exercise
title: Hello world task
weight: 1
author: Dr Graeme Stuart
---

The *deno* runtime allows us to execute JavaScript code. 
In this exercise we will write a hello world programme and run it as a deno task.

<!--more-->

> Before you start, make sure deno is installed and working.
See the [Deno setup][deno setup] exercise for details.

## Learning objectives

- Configuring a deno project with a **deno.json** file
- Creating and running [tasks]
- Organising our entry points in a **tasks** folder

## Setup

This will be the first of a long series of exercises which build towards a complex and fully featured web application.

### Initialise a git repository

Before you start, you should create a new empty folder, open it in VSCode and initialise a git repository.

> See [Getting Started with Git and Github]({{< ref "/extras/git-in-vscode" >}}) for details. 

## Create a hello world script

Our project will have several *entry points*, i.e. modules that we execute directly.
We will keep all of them together in a **tasks** folder.

Create a **tasks** folder and a new file inside it called **hello.js** with a simple "hello world" example.

**tasks/hello.js**
```javascript
console.log("Hello World");
```

Make sure you save the file (**Ctrl + S**).

Open the PowerShell terminal in VSCode (**Ctrl + @**) and run the script with the following command.

```powershell{linenos=false}
deno tasks/hello.js
```

> We are running the **deno** programme and passing it the path to the JavaScript code we want to execute. 

You should see the output printed in the terminal.

```plaintext{linenos=false}
Hello World
```

> If you don't see anything then ask your lab tutor to help. 

## Create a task

Deno can be configured by creating a [**deno.json**][deno.json] file in the root of the project.
One of the important uses of this file is to create `tasks` which are essentially shortcuts for complex commands.

Create a **deno.json** file in the root of the project and add a `hello` task.

**deno.json**
```json
{
    "tasks": {
        "hello": "deno tasks/hello.js"
    }
}
```

> The `"tasks"` property is an object.
> Each key is the name of a task and each value is the command that will be executed.

Now you should be able to execute the task from the terminal like this.

```bash{linenos=false}
deno task hello
```

All this does is lookup the command in **deno.json** and executes it.
Deno prints the command it is running, followed by the output.

```plaintext{linenos=false}
Task hello deno tasks/hello.js
Hello World
```

> Try running `deno task` on its own.
> Deno will list all the available tasks.

### Our convention

In this module, every task in **deno.json** will point to a module in the **tasks** folder.

> This means the **tasks** folder tells us all the ways our project can be run.
> The rest of our code will be organised into other folders and imported by the task modules.

Our project now looks like this.

```plaintext{linenos=false}
deno.json
tasks/
    hello.js
```

### Permissions

Deno is *secure by default*.
A script cannot access the network, the file system or the environment unless we give it [permission][permissions].

Our hello world script only writes to the terminal, which needs no permissions at all, so our task needs no permission flags.

> Later tasks will need permissions such as `--allow-net`.
> We will often use the `-A` flag (short for `--allow-all`) to keep our tasks simple.
> This grants **every** permission, so the script can do anything you can do, just make sure you know what code you are executing.
> In each case we will note which specific permissions are actually needed.

## Write a more complex programme

Update **tasks/hello.js** with the following code.

```JavaScript
const items = ['apples', 'bananas', 'cherries'];
for(const item of items) {
   console.log("logging", item);
}
```

Our code iterates over an array and logs each element.

### Run the task

Execute the task from the terminal as before.

```bash{linenos=false}
deno task hello
```

You should see the output is printed to the terminal.

```plaintext{linenos=false}
Task hello deno tasks/hello.js
logging apples
logging bananas
logging cherries
```

> Before moving on, change **tasks/hello.js** back to the simple hello world example.

## Conclusion

We have created a deno project with a **deno.json** file and used a task to run a very simple JavaScript programme.
Every task points to a module in the **tasks** folder.

> Now try adding a second task which runs a different script in the **tasks** folder.
> Perhaps experiment with [while loops].

[tasks]: https://docs.deno.com/runtime/reference/cli/task/
[deno.json]: https://docs.deno.com/runtime/fundamentals/configuration/
[permissions]: https://docs.deno.com/runtime/fundamentals/security/
[while loops]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/while

[deno setup]: javascript-fundamentals/deno-setup/

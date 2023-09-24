+++
title = "Building Azure Functions With F# Script and Visual Studio Code - Step 1 - Setup Your Environment"
abstract = "Part 1 in a series of posts describing how to create a simple Azure Function using F# Script, VS Code, and v1 of the Azure Functions Core Tools."
date = 2018-04-12
tags = ['Azure', 'FSharp', 'Azure Functions', 'Visual Studio Code']
draft = false
github = "https://github.com/mikesigs/building-azure-functions-with-fsharp-and-vscode"

[header]
image = "headers/high-speed-motion.jpg"
preview = true
+++

This is Step 1 in a [series of posts](../) where I will walk you through the steps required to create a simple Azure Function using F# Script, VS Code, and v1 of the Azure Functions Core Tools.
I'll cover everything from what you need to install, all the way through creating the function, and deploying it to your Azure account.

1. **[Setup Your Environment](../1-setup/)** :arrow_backward:
2. [Create the Function App](../2-create-function-app/)
3. [Run the Function Locally](../3-running-locally/)
4. [Deploy the Function App to Azure](../4-deploy-to-azure/)

## Setup Your Environment

There are several ways to develop Azure Functions. You can choose to do everything in the Azure Portal, you can use the full blown Visual Studio, or you can use Visual Studio Code and the CLI. We'll be going the latter route.

Let's get your machine properly setup! You will need the following:

### 1) Install Visual Studio Code

This seems like a no-brainer. But if you don't already have it, install [Visual Studio Code](https://code.visualstudio.com/).

### 2) Install F\#

If you have Visual Studio installed, perhaps you've already included support for F#. If that's the case, you can skip this step. If not, then head over to [fsharp.org](http://fsharp.org/) and download the appropriate version for your system.

### 3) Install the Ionide Plugins for Visual Studio Code

If you've done any F# development in VS Code, then you almost certainly have these installed. If not, you'll soon come to learn that **Ionide is awesome!**

Use the Visual Studio Code Extensions Marketplace to install them. Just click the Extensions button in the sidebar and search for _"Ionide"_.

#### [Ionide-fsharp](https://github.com/ionide/ionide-vscode-fsharp)

For F# intellisense, document formatting, syntax and error highlighting, and more. It also provides project scaffolding using a tool called [Forge](http://fsharp-editing.github.io/Forge/).

#### [Ionide-Paket]([Ionide Paket](https://github.com/ionide/ionide-vscode-paket))

[Paket](http://fsprojects.github.io/Paket/) integration, for better package management

#### [Ionide-FAKE](https://github.com/ionide/ionide-vscode-fake)

[FAKE](http://fsharp.github.io/FAKE/) integration. FAKE is an amazing build tool for .NET projects.

### 4) Install the Azure Functions Core Tools

The [Azure Functions Core Tools](https://github.com/Azure/azure-functions-core-tools) let you run your functions locally. It is the exact same runtime used in Azure to host your functions. Pretty cool!

We're going to be using v1.0 of the Core Tools in this series. The beta 2.0 version doesn't support F# script yet. It does support pre-compiled F# however. But that is the subject of a future blog post.

One significant caveat to using v1.0 is it only runs on Windows. Sorry about that Mac/Linux users :disappointed:

To install the Core Tools you have two options, [npm](https://www.npmjs.com/get-npm) or [Chocolatey](https://chocolatey.org/).

#### Option 1: NPM

npm is distributed with Node.js, so if you don't have Node installed that should be step one. Alternatively, you could use Chocolatey (see below).

{{< notice note >}}
I highly recommend you don't install Node directly. You should use [nvm-windows](https://github.com/coreybutler/nvm-windows),
which allows multiple versions of Node to be installed simultaneously. Check the [GitHub](https://github.com/coreybutler/nvm-windows) page for installation instructions.

It's not essential to use nvm-windows for this tutorial, but in my experience, it is often nice to be able to be able to switch between different node versions as the need arises.
{{< /notice >}}

Once you have Node installed, and thus npm, you can install the Core Tools with the following command:

```shell
npm i -g azure-functions-core-tools
```

#### Option 2: Chocolatey

Chocolatey is a package manager for Windows. It allows you to install _nearly_ anything from the command line. If you've never checked it out before then now would definitely be a good time.
Head over to [the Chocolatey website](https://chocolatey.org/install) for installation instructions.

{{< notice note >}}
To give you some idea of how Chocolatey could improve your computering experience. When I have to do a fresh install on a new machine (which I've had to do 4 times this year. Long story...)
I can install 90% of the things I need with a single command. I won't list everything I have installed, but as an example:

`choco install 7zip azure-cli cmder diffmerge git gitkraken hub hugo linqpad lockhunter ngrok notepadplusplus ...`

Chocolatey then downloads and silently installs each of those programs.
{{< /notice >}}

Once you have Chocolatey installed, you can get the Core Tools with the following command:

```shell
choco install azure-functions-core-tools
```

### 5) Install the Azure Functions Extension

Another extremely useful Visual Studio Code extension is the [Azure Functions](https://github.com/Microsoft/vscode-azurefunctions) extension. Search for and install it in the Extensions sidebar.

This extension will make working with Azure Functions in Visual Studio Code a breeze. It leverages the Azure Functions Core Tools to power a lot of its functionality, along with providing some additional customization to your VS Code workspace.
Unfortunately, [we cannot create an F# project with it yet](https://github.com/Microsoft/vscode-azurefunctions/issues/315). But it will still recognize an existing F# Azure Functions project the first time you open it.

## What's Next

That's pretty much it. You're all setup!

Now that you've got all the tools you need. Let's move on to [Step 2](../2-create-function-app/) and Create The Function App!

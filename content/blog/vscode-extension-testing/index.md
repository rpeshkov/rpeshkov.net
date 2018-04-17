+++
title = "VSCode extension testing"
date = 2018-04-17T22:15:49+02:00
draft = true
tags = ["vscode"]
+++

When comes to text editing, Visual Studio Code is my favourite editor. It's blazingly fast (except startup time) and has a lot of useful features that makes text and code editing easy as a pie. If you want, you can extend its functionality even more with a lot of extensions available in marketplace. Also it's relatively easy to write extension by yourself since VSCode has very good and well-documented extensibility API with a lot of samples.

However, when it comes to writing tests and measuring tests code coverage, there are not so many documents available. Some information about testing I found only as comments in various GitHub issues.

In this article I would like to fill the gap related to testing your extension and provide a bunch of examples of how to make your tests better and your extension more robust to the future changes.

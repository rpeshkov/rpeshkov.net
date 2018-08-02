+++
title = "VSCode extension dependency injection"
date = 2018-08-02T20:00:05+02:00
draft = true
tags = ["vscode"]
+++

In this post I'll show how to use dependency injection in your extension via InversifyJS library. Here's about from [official site][inversifyjs]:

> InversifyJS is a lightweight (4KB) inversion of control (IoC) container for TypeScript and JavaScript apps. A IoC container uses a class constructor to identify and inject its dependencies.

Sounds good. Let's begin. :)

First we need to create our extension. You can read about extension creation in my post about [VSCode extension code coverage]({{< ref "/blog/vscode-extension-coverage#creating-extension" >}}). I've chosen `vscode-di` name for extension so everywhere later I'll be using that name. After extension is created, open it in VSCode.

[![VSCode with extension][vscode]][vscode]

After you have extension created it's required to install InversifyJS itself and also additional package called [reflect-metadata][reflect-metadata]. Do it via this console command:

{{< highlight sh >}}
npm install inversify reflect-metadata --save
{{< / highlight >}}

Here what I got:

{{< highlight sh >}}
+ reflect-metadata@0.1.12
+ inversify@4.13.0
{{< / highlight >}}

One note here: packages installed as dependencies and not devDependencies. That's very important thing. If you install those packages as dev dependencies, it will still work fine while you're developing, but it will crash if your extension will be installed from marketplace.

Next step that's required is to enable some compilation options in your `tsconfig.json` file. `experimentalDecorators` and `emitDecoratorMetadata` options must be enabled and `types` option must contain `reflect-metadata` in it. Open `tsconfig.json` file and add there required changes. Here how my `tsconfig.json` file looks like after changes:

{{< highlight json "" >}}
{
    "compilerOptions": {
        "module": "commonjs",
        "target": "es6",
        "outDir": "out",
        "lib": ["es6"],
        "sourceMap": true,
        "rootDir": "src",
        "experimentalDecorators": true,
        "emitDecoratorMetadata": true,
        "types": ["reflect-metadata"],
        "strict": true,
        "noUnusedLocals": true
    },
    "exclude": [
        "node_modules",
        ".vscode-test"
    ]
}
{{< / highlight >}}

[inversifyjs]: http://inversify.io/
[reflect-metadata]: https://www.npmjs.com/package/reflect-metadata
[vscode]: ./1-vscode.png

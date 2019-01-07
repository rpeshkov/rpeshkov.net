---
title: "VSCode extension dependency injection"
date: 2018-08-02T20:00:05+02:00
draft: false
tags: ["vscode"]
---

In this post I'll show how to use dependency injection in your extension via InversifyJS library. Here's about from [official site][inversifyjs]:

> InversifyJS is a lightweight (4KB) inversion of control (IoC) container for TypeScript and JavaScript apps. A IoC container uses a class constructor to identify and inject its dependencies.

Sounds good. Let's begin. :)

First we need to create our extension. You can read about extension creation in my post about [VSCode extension code coverage]({{< ref "/blog/vscode-extension-coverage#creating-extension" >}}). I've chosen `vscode-di` name for extension so everywhere later I'll be using that name. After extension is created, open it in VSCode.

[![VSCode with extension][vscode]][vscode]

After you have extension created it's required to install InversifyJS itself and also additional package called [reflect-metadata][reflect-metadata]. Do it via this console command:

```sh
npm install inversify reflect-metadata --save
```

Here what I got:

```sh
+ reflect-metadata@0.1.12
+ inversify@4.13.0
```

One note here: packages installed as dependencies and not devDependencies. That's very important thing. If you install those packages as dev dependencies, it will still work fine while you're developing, but it will crash if your extension will be installed from marketplace.

Next step that's required is to enable some compilation options in your `tsconfig.json` file. `experimentalDecorators` and `emitDecoratorMetadata` options must be enabled. Open `tsconfig.json` file and add there required changes. Here how my `tsconfig.json` file looks like after changes:

```json
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
        "strict": true,
        "noUnusedLocals": true
    },
    "exclude": [
        "node_modules",
        ".vscode-test"
    ]
}
```

That's it with initialization, now let's proceed with coding.

Let's start with defining our interfaces. For this tutorial we'll do very basic stuff just to show how DI works in Inversify.

```typescript
export interface Command {
    id: string;
    execute(...args: any[]): any;
}
```

This `Command` interface will be used for describing commands that we will register in VSCode.

```typescript
export interface Printer {
    print(message: string): void;
}
```

This `Printer` interface defines abstract point of message output. Later we will inject this `Printer` into our commands.

For this article I put `Command` interface into `commands` folder and `Printer` interface into `utils` folder.

Also, for correct injection we need symbols. Symbol defines an identifier that will be used later for registering and resolving dependencies. I placed symbols definition in the root of `src` folder and this file contains this:

```typescript
const TYPES = {
    Command: Symbol("Command"),
    Printer: Symbol("Printer")
};

export default TYPES;
```

After all those changes, your `src` folder should look this way:

```sh
.
├── commands
│   ├── command.ts
├── extension.ts
├── test
│   ├── extension.test.ts
│   └── index.ts
├── types.ts
└── utils
    └── printer.ts
```

Next step will be to define implementations for our interfaces.

We'll have 2 commands and 1 printer. Commands will be `AddCommand` and `RemoveCommand`, and printer will be very simple - the one that displays message in console through `console.log` call.

Create new file `add-command.ts` inside `commands` folder with the following content:

```typescript
import { injectable, inject } from 'inversify';

import TYPES from '../types';

import { Command } from './command';
import { Printer } from '../utils/printer';

@injectable()
export class AddCommand implements Command {
    constructor(
        @inject(TYPES.Printer) private printer: Printer
    ) {}

    get id() {
        return 'extension.add';
    }

    execute(...args: any[]) {
        this.printer.print('AddCommand');
    }
}
```

There are two main things to notice here:

+ `@injectable` decorator on the class;
+ `@inject` decorator for the parameters of constructor.

`@injectable` decorator tells that this class will be injected. It's mandatory decorator for registering class in the DI container.

`@inject` decorator for parameter tells DI container that it should resolve the type provided and pass it here.

Second command that we'll implement looks very similar to `AddCommand` and it's called `RemoveCommand`. Create new file `remove-command.ts` inside `commands` folder with the following contents:

```typescript
import { injectable, inject } from 'inversify';

import TYPES from '../types';

import { Command } from './command';
import { Printer } from '../utils/printer';

@injectable()
export class RemoveCommand implements Command {
    constructor(
        @inject(TYPES.Printer) private printer: Printer
    ) {}

    get id() {
        return 'extension.remove';
    }

    execute(...args: any[]) {
        this.printer.print('RemoveCommand');
    }
}
```

Difference between 2 commands are basically name of the class, id of command and what this command prints via `Printer`.

Now, let's implement our printer. Create new file `console-printer.ts` inside `utils` folder with the following content:

```typescript
import { injectable } from 'inversify';
import { Printer } from './printer';

@injectable()
export class ConsolePrinter implements Printer {
    print(message: string): void {
        console.log(message);
    }
}
```

As you may already noticed, it's implementation is very simple.

Before we will start setting up our container, let's create one more thing that will help up with commands registering in VSCode. I called it `CommandsManager` and placed near all the commands we defined - inside `commands` folder. Here how it looks like:

```typescript
import * as vscode from 'vscode';
import { multiInject, injectable } from 'inversify';
import TYPES from '../types';
import { Command } from './command';

@injectable()
export class CommandsManager {
    constructor(
        @multiInject(TYPES.Command) private commands: Command[]
    ) {}

    registerCommands(context: vscode.ExtensionContext) {
        for (const c of this.commands) {
            const cmd = vscode.commands.registerCommand(c.id, c.execute);
            context.subscriptions.push(cmd);
        }
    }
}
```

While this class is quite small, there is one thing that differs from our command implementations. You should notice `@multiInject` decorator at the constructor parameter and parameter type is array. `@multiInject` decorator tells DI container to inject all the entities with specified symbol (`TYPES.Command` in our case). That basically means that all the implementations of `Command` interface will be passed here as an array.

Phew, that's it with implementations, now let's finally configure our DI container and try to work with it.

If you look through official documentation for Inversify, you'll see that it recommends putting container into `inversify.config.ts` file. Let's stick with this recommendation and create the same file in `src` folder.

```typescript
import 'reflect-metadata';

import { Container } from 'inversify';
import TYPES from './types';
import { Printer } from './utils/printer';
import { ConsolePrinter } from './utils/console-printer';
import { AddCommand } from './commands/add-command';
import { Command } from './commands/command';
import { RemoveCommand } from './commands/remove-command';
import { CommandsManager } from './commands/commands-manager';

const container = new Container();
container.bind<Printer>(TYPES.Printer).to(ConsolePrinter);
container.bind<Command>(TYPES.Command).to(AddCommand);
container.bind<Command>(TYPES.Command).to(RemoveCommand);
container.bind<CommandsManager>(TYPES.CommandManager).to(CommandsManager);

export default container;
```

If you previously worked with any DI container, this will look very familiar to you. Still, one important thing here is the first line. Without it, nothing would work, because this library should be imported globally once.

So, now we have our entities, symbols and set up our container. Let's finally do something useful and register our commands so that they will be working.

Open file `extension.ts` and inside `activate` method write the following:

```typescript
const cmdManager = container.get<CommandsManager>(TYPES.CommandManager);
cmdManager.registerCommands(context);
```

Also, don't forget to tell in `package.json` file that your extension contributes commands. Here how mine `contributes` section looks like:

```json
"contributes": {
    "commands": [
        {
            "command": "extension.add",
            "title": "DI: Add"
        },
        {
            "command": "extension.remove",
            "title": "DI: Remove"
        }
    ]
}
```

That's basically it. You can now launch your extension and try to invoke command from command palette.

I tried to launch `add` command and it failed... :(

```sh
Running the contributed command:'extension.add' failed.
```

You might be wondering "why did that happen???". Well, if you will try to launch your extension under debug and stop at breakpoint inside any command's execute method, you'll notice that `this` is undefined.

[![Extension run inside debugger][extension-debug]][extension-debug]

Explanation for this is quite simple: `this` context is not set to your object when `execute` method is launched. You can fix this by providing correct `this` context when registering your command inside VSCode. Open your `CommandManager` and find line with `registerCommand` invocation. This function accepts third parameter `thisArg?: any`. Since it's optional and we didn't provide it, `this` when `execute` method is called by VSCode is undefined.

Change this line into this:

```typescript
const cmd = vscode.commands.registerCommand(c.id, c.execute, c);
```

and that will setup correct `this` context during method execution.

You can try to launch extension once again and you'll notice that everything works fine and your commands displaying messages in debug console of VSCode.

[![Debug console output][debug-console]][debug-console]

## Conclusion

Hopefully, you now have a better understanding of how to integrate DI framework into your extension codebase and start using it. During this article we've just scratched the surface of what Inversify can do. If you want to know more about Inversify's features, refer to the [documentation][inversify-docs]. If you have any questions about this article or something isn't working after you followed all steps, you can drop me a message in Twitter or view the source code of final extension in GitHub.

[inversifyjs]: http://inversify.io/
[reflect-metadata]: https://www.npmjs.com/package/reflect-metadata
[vscode]: ./1-vscode.png
[extension-debug]: ./2-extension-debug.png
[debug-console]: ./3-debug-console-output.png
[inversify-docs]: https://github.com/inversify/InversifyJS/tree/master/wiki

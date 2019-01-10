---
title: "VSCode extension code coverage"
date: 2018-05-06T22:38:35+02:00
draft: false
tags: ["vscode"]
---

In this article I'll show you how to add code coverage info for you Visual Studio Code extension code. We'll start from the very beginning and in the end you'll have working extension with code coverage metrics setup.

## Creating extension

Creating extension is fairly simple. We'll start from the default code that's generated by Yeoman's generator for VSCode.

Open your terminal and type `yo code` in it. You can leave all the values to their defaults. Here what I used for generating extension:

[![Yeoman VSCode extension generator][yeoman]][yeoman]

After last question about initializing git repository, generator will install scaffold extension code and install all the required npm packages. Now you can navigate into the extension's folder and open it in VSCode:

```bash
cd vscode-testcov
code .
```

[![VSCode with opened extension][vscode-ext]][vscode-ext]

Just in order the check that everything is fine, try to launch tests for your new extension. If everything is fine, you should see similar picture in your debug console:

[![VSCode after running tests][vscode-test-run]][vscode-test-run]

## Istanbul for code coverage

For measuring the code coverage we'll be using [Istanbul](https://github.com/gotwarlost/istanbul). Description for Istanbul says:

> Yet another JS code coverage tool that computes statement, line, function and branch coverage with module loader hooks to transparently add coverage when running tests. Supports all JS coverage use cases including unit tests, server side functional tests and browser tests. Built for scale.

First we need to add Instabul and other utility packages to our project. Open terminal window, navigate to your project and execute following command:

```bash
npm install istanbul remap-istanbul glob @types/glob decache --save
```

Here is the output with packages and their versions that were installed:

```bash
+ remap-istanbul@0.11.1
+ istanbul@0.4.5
+ decache@4.4.0
+ glob@7.1.2
+ @types/glob@5.0.35
```

## Reimplementing test runner

For launching tests of VSCode extension, VSCode itself provides test runner that does a lot of boilerplate and launches testing framework (Mocha by default). In order to have code coverage in your extension, we need to reimplement this test runner a bit, injecting additional instructions there.

Now, open `index.ts` file in `src/test` folder of extension. You should see there:

```typescript
import * as testRunner from 'vscode/lib/testrunner';

// You can directly control Mocha options by uncommenting the following lines
// See https://github.com/mochajs/mocha/wiki/Using-mocha-programmatically#set-options for more info
testRunner.configure({
    ui: 'tdd', 		// the TDD UI is being used in extension.test.ts (suite, test, etc.)
    useColors: true // colored output from test results
});

module.exports = testRunner;
```

This is default configuration that basically just sets settings to TDD style and enables colored output.

Replace contents of this file with the following:

```typescript
'use strict';

declare var global: any;

/* tslint:disable no-require-imports */

import * as fs from 'fs';
import * as glob from 'glob';
import * as paths from 'path';

const istanbul = require('istanbul');
const Mocha = require('mocha');
const remapIstanbul = require('remap-istanbul');

// Linux: prevent a weird NPE when mocha on Linux requires the window size from the TTY
// Since we are not running in a tty environment, we just implementt he method statically
const tty = require('tty');
if (!tty.getWindowSize) {
    tty.getWindowSize = (): number[] => {
        return [80, 75];
    };
}

let mocha = new Mocha({
    ui: 'tdd',
    useColors: true,
});

function configure(mochaOpts: any): void {
    mocha = new Mocha(mochaOpts);
}
exports.configure = configure;

function _mkDirIfExists(dir: string): void {
    if (!fs.existsSync(dir)) {
        fs.mkdirSync(dir);
    }
}

function _readCoverOptions(testsRoot: string): ITestRunnerOptions | undefined {
    const coverConfigPath = paths.join(testsRoot, '..', '..', 'coverconfig.json');
    if (fs.existsSync(coverConfigPath)) {
        const configContent = fs.readFileSync(coverConfigPath, 'utf-8');
        return JSON.parse(configContent);
    }
    return undefined;
}

function run(testsRoot: string, clb: any): any {
    // Read configuration for the coverage file
    const coverOptions = _readCoverOptions(testsRoot);
    if (coverOptions && coverOptions.enabled) {
        // Setup coverage pre-test, including post-test hook to report
        const coverageRunner = new CoverageRunner(coverOptions, testsRoot);
        coverageRunner.setupCoverage();
    }

    // Glob test files
    glob('**/**.test.js', { cwd: testsRoot }, (error, files): any => {
        if (error) {
            return clb(error);
        }
        try {
            // Fill into Mocha
            files.forEach((f): Mocha => mocha.addFile(paths.join(testsRoot, f)));
            // Run the tests
            let failureCount = 0;

            mocha.run()
                .on('fail', () => failureCount++)
                .on('end', () => clb(undefined, failureCount)
            );
        } catch (error) {
            return clb(error);
        }
    });
}
exports.run = run;

interface ITestRunnerOptions {
    enabled?: boolean;
    relativeCoverageDir: string;
    relativeSourcePath: string;
    ignorePatterns: string[];
    includePid?: boolean;
    reports?: string[];
    verbose?: boolean;
}

class CoverageRunner {

    private coverageVar: string = '$$cov_' + new Date().getTime() + '$$';
    private transformer: any = undefined;
    private matchFn: any = undefined;
    private instrumenter: any = undefined;

    constructor(private options: ITestRunnerOptions, private testsRoot: string) {
        if (!options.relativeSourcePath) {
            return;
        }
    }

    public setupCoverage(): void {
        // Set up Code Coverage, hooking require so that instrumented code is returned
        const self = this;
        self.instrumenter = new istanbul.Instrumenter({ coverageVariable: self.coverageVar });
        const sourceRoot = paths.join(self.testsRoot, self.options.relativeSourcePath);

        // Glob source files
        const srcFiles = glob.sync('**/**.js', {
            cwd: sourceRoot,
            ignore: self.options.ignorePatterns,
        });

        // Create a match function - taken from the run-with-cover.js in istanbul.
        const decache = require('decache');
        const fileMap: any = {};
        srcFiles.forEach((file) => {
            const fullPath = paths.join(sourceRoot, file);
            fileMap[fullPath] = true;

            // On Windows, extension is loaded pre-test hooks and this mean we lose
            // our chance to hook the Require call. In order to instrument the code
            // we have to decache the JS file so on next load it gets instrumented.
            // This doesn't impact tests, but is a concern if we had some integration
            // tests that relied on VSCode accessing our module since there could be
            // some shared global state that we lose.
            decache(fullPath);
        });

        self.matchFn = (file: string): boolean => fileMap[file];
        self.matchFn.files = Object.keys(fileMap);

        // Hook up to the Require function so that when this is called, if any of our source files
        // are required, the instrumented version is pulled in instead. These instrumented versions
        // write to a global coverage variable with hit counts whenever they are accessed
        self.transformer = self.instrumenter.instrumentSync.bind(self.instrumenter);
        const hookOpts = { verbose: false, extensions: ['.js'] };
        istanbul.hook.hookRequire(self.matchFn, self.transformer, hookOpts);

        // initialize the global variable to stop mocha from complaining about leaks
        global[self.coverageVar] = {};

        // Hook the process exit event to handle reporting
        // Only report coverage if the process is exiting successfully
        process.on('exit', (code: number) => {
            self.reportCoverage();
            process.exitCode = code;
        });
    }

    /**
     * Writes a coverage report.
     * Note that as this is called in the process exit callback, all calls must be synchronous.
     *
     * @returns {void}
     *
     * @memberOf CoverageRunner
     */
    public reportCoverage(): void {
        const self = this;
        istanbul.hook.unhookRequire();
        let cov: any;
        if (typeof global[self.coverageVar] === 'undefined' || Object.keys(global[self.coverageVar]).length === 0) {
            console.error('No coverage information was collected, exit without writing coverage information');
            return;
        } else {
            cov = global[self.coverageVar];
        }

        // TODO consider putting this under a conditional flag
        // Files that are not touched by code ran by the test runner is manually instrumented, to
        // illustrate the missing coverage.
        self.matchFn.files.forEach((file: any) => {
            if (cov[file]) {
                return;
            }
            self.transformer(fs.readFileSync(file, 'utf-8'), file);

            // When instrumenting the code, istanbul will give each FunctionDeclaration a value of 1 in coverState.s,
            // presumably to compensate for function hoisting. We need to reset this, as the function was not hoisted,
            // as it was never loaded.
            Object.keys(self.instrumenter.coverState.s).forEach((key) => {
                self.instrumenter.coverState.s[key] = 0;
            });

            cov[file] = self.instrumenter.coverState;
        });

        // TODO Allow config of reporting directory with
        const reportingDir = paths.join(self.testsRoot, self.options.relativeCoverageDir);
        const includePid = self.options.includePid;
        const pidExt = includePid ? ('-' + process.pid) : '';
        const coverageFile = paths.resolve(reportingDir, 'coverage' + pidExt + '.json');

        // yes, do this again since some test runners could clean the dir initially created
        _mkDirIfExists(reportingDir);

        fs.writeFileSync(coverageFile, JSON.stringify(cov), 'utf8');

        const remappedCollector = remapIstanbul.remap(cov, {
            warn: (warning: any) => {
                // We expect some warnings as any JS file without a typescript mapping will cause this.
                // By default, we'll skip printing these to the console as it clutters it up
                if (self.options.verbose) {
                    console.warn(warning);
                }
            }
        });

        const reporter = new istanbul.Reporter(undefined, reportingDir);
        const reportTypes = (self.options.reports instanceof Array) ? self.options.reports : ['lcov'];
        reporter.addAll(reportTypes);
        reporter.write(remappedCollector, true, () => {
            console.log(`reports written to ${reportingDir}`);
        });
    }
}
```

Originally this code is taken from [here](https://github.com/codecov/example-typescript-vscode-extension/blob/master/test/index.ts), but I've made some changes to be compatible with strict mode.

Also, you need to provide configuration for the Istanbul. Create file `coverconfig.json` in the root of your project with the following content:

```json
{
    "enabled": true,
    "relativeSourcePath": "../src",
    "relativeCoverageDir": "../../coverage",
    "ignorePatterns": [
        "**/node_modules/**"
    ],
    "includePid": false,
    "reports": [
        "html",
        "lcov",
        "text-summary"
    ],
    "verbose": false
}
```

Now, you're ready to go. Try to launch tests once again and see the results. If you were following the instructions, you should see the message `No coverage information was collected, exit without writing coverage information` after your tests execution output. What?! Why?! Well, no coverage information was collected because default test doesn't execute any functions from the extension. You can read this message more like `There was nothing to collect`.

Change your `extension.test.ts` file to execute at least something that's related with your extension. Change the body of `Something 1` test to this:

```typescript
vscode.commands.executeCommand("extension.sayHello");
```

and fix your imports so code would be compilable and then run your tests once again. After all that fixes you would see the report:

```bash
=============================== Coverage summary ===============================
Statements   : 6.54% ( 7/107 )
Branches     : 0% ( 0/26 )
Functions    : 9.09% ( 1/11 )
Lines        : 6.73% ( 7/104 )
================================================================================
reports written to /Users/rpeshkov/Developer/vscode-extensions/vscode-testcov/coverage
```

Yay! All the information related to code coverage was saved to `coverage` folder in the root of your extension. You now can open `index.html` file there to see HTML-based report.

[![HTML Code coverage report][html-rep]][html-rep]

It looks really nice, but you may noticed one small problem here that tests code was covered as well. That doesn't seem right, so let's fix it by some structural changes.

## Reorganizing project

The problem with tests code coverage arise from a simple fact that coverage is measured for `src` folder and tests code is there as well. To fix this, move `test` folder out of `src` folder so they will be on the same level.

[![src and test folders][src-test-folders]][src-test-folders]

With this change you also need to change a couple of parameters in your `tsconfig.json` file and `package.json` file.

In your `tsconfig.json` file change `rootDir` parameter to `.`:

```json
"rootDir": ".",
```

In your `package.json` file change `main` parameter to `./out/src/extension`.

```json
"main": "./out/src/extension",
```

Now relaunch your tests and you'll see that coverage report doesn't have coverage for your tests code.

## Conclusion

Strange that VSCode extension generator doesn't have this functionality out of the box, but as you can see, it's actually not so hard to implement it by yourself. If you had any troubles during implementation of the steps above, you may get final version from the GitHub repository https://github.com/rpeshkov/vscode-testcov and compare it with what you got or you can ask me in Twitter and I'll try to help you.

[yeoman]: https://s3.eu-central-1.amazonaws.com/rpeshkov.net/vscode-extension-coverage/1-yeoman-code-generator.png
[vscode-ext]: https://s3.eu-central-1.amazonaws.com/rpeshkov.net/vscode-extension-coverage/2-vscode-with-opened-extension.png
[vscode-test-run]: https://s3.eu-central-1.amazonaws.com/rpeshkov.net/vscode-extension-coverage/3-vscode-run-tests.png
[html-rep]: https://s3.eu-central-1.amazonaws.com/rpeshkov.net/vscode-extension-coverage/4-html-report.png
[src-test-folders]: https://s3.eu-central-1.amazonaws.com/rpeshkov.net/vscode-extension-coverage/5-src-test-folders.png
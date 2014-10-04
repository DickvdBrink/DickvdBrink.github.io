---
layout: post
title:  "Using the TypeScript language service in NodeJS"
date:   2014-10-04 14:08:00
categories: TypeScript
---

**Update**: This is actually an old post, TypeScript moved to [GitHub](https://github.com/microsoft/typescript).

This blog post is about using the TypeScript languageservice in Node.js. If you don't know what TypeScript is you might want to look at the website first: [TypeScript Homepage](http://www.typescriptlang.org/).
In short, it is JavaScript with optional typeinfo and the language provides extra features like classes, modules and interfaces. After compiling the TypeScript files, plain JavaScript comes out which run everywhere (where JavaScript is supported).

Why would you want to use the languageservice, what are the possibilities? To answer this question it is easier to build TypeScript from source and look at the ILanguageService interface but some nice things you can do with it are: get autocompletion info, syntaxtree information, references. So basically you can add TypeScript support to an IDE (or build your own) with autocompletion, find all references support by using this languageservice.

<!--more-->

**Note**: This API isn't stable yet and might in the future!

## Building TypeScript

To follow the steps below you need to install some tools:

 * Node.js
 * Git

First we start by building the TypeScript environment from source. Execute the folowing commands from your command prompt or terminal:

 * git clone [https://git01.codeplex.com/typescript](https://git01.codeplex.com/typescript)
 * npm install -g jake
 * git checkout v1.0.2

This wil clone the TypeScript repository and will install jake. Jake is the buildsystem TypeScript uses. The git checkout command switches to the tag v1.0.2 to make sure the steps below work correctly. It might still work on the develop branch though.

With "jake local" you can build TypeScript. After everything is finished you can find the output in the "built\local" folder.
In the folder you find several JavaScript and TypeScript definition files (.d.ts). We are interested in the following files: "typescriptServices.d.ts" and "typescriptServices.js";
![TypeScript output](/assets/language_service/build_output.png)


Because we want to use the TypeScript language service in Node.js we need to make some small changes to the above two files.

In the "typescriptServices.d.ts" file you can find the following declaration:

{% highlight ts %}
declare class FormattingOptions {
    public useTabs: boolean;
    ...
    static defaultOptions: FormattingOptions;
}
{% endhighlight %}

Remove it and put it in a new file called formattingOptions.d.ts, this isn't necessary on the develop branch anymore though.
Add `module.exports = TypeScript;` to the typescriptServices.js file and `export = TypeScript;` to the typescriptServices.d.ts file. In both cases, add it to the bottom of the file.

Now we create a new file called languageService.ts and put the following code at the top of the file:

{% highlight javascript %}
    import TypeScript = require("./typescriptServices");
{% endhighlight %}

It is possible to compile it with the following command when executed from the built\local folder: 

`node tsc.js --module commonjs typescriptServices.d.ts formattingOptions.d.ts languageService.ts`

If you create a Visual Studio project and include the files, you get full intellisense when you use the TypeScript object. Just make sure you set the module format to "CommonJS". I will use that from now on but you can use any IDE you like or just continue using notepad and compiling from the commandline.


## Implementing a LanguageServiceHost

Now we can use the TypeScript object it is time to implement a LanguageServiceHost. To implement we need to implement the following interface:

{% highlight ts %}
interface ILanguageServiceHost extends ILogger, IReferenceResolverHost {
    getCompilationSettings(): CompilationSettings;
    getScriptFileNames(): string[];
    getScriptVersion(fileName: string): number;
    getScriptIsOpen(fileName: string): boolean;
    getScriptByteOrderMark(fileName: string): ByteOrderMark;
    getScriptSnapshot(fileName: string): IScriptSnapshot;
    getDiagnosticsObject(): ILanguageServicesDiagnostics;
    getLocalizedDiagnosticMessages(): any;
}
{% endhighlight %}

This interface implements two other interfaces which have the following declaration:
{% highlight ts %}
interface ILogger {
    information(): boolean;
    debug(): boolean;
    warning(): boolean;
    error(): boolean;
    fatal(): boolean;
    log(s: string): void;
}
interface IReferenceResolverHost {
    getScriptSnapshot(fileName: string): IScriptSnapshot;
    resolveRelativePath(path: string, directory: string): string;
    fileExists(path: string): boolean;
    directoryExists(path: string): boolean;
    getParentDirectory(path: string): string;
}
{% endhighlight %}

An implementation might look something like the code below but the below implementation isn't finished yet:

{% highlight ts %}
class LanguageServiceHost implements TypeScript.Services.ILanguageServiceHost {

    private files : TypeScript.StringHashTable<string> = new TypeScript.StringHashTable<string>();

    constructor(private settings: TypeScript.CompilationSettings) {
    }

    public addFile(fileName: string) {
        var buffer = fs.readFileSync(fileName);
        this.files.add(fileName, buffer.toString());
    }

    public addScript(fileName: string, contents: string) {
        this.files.add(fileName, contents);
    }

    //#region Services.ILanguageServiceHost

    public getCompilationSettings(): TypeScript.CompilationSettings {
        return this.settings;
    }

    public getScriptFileNames(): string[] {
        return this.files.getAllKeys();
    }

    public getScriptVersion(fileName: string): number { return 1; }
    public getScriptIsOpen(fileName: string): boolean { return false; }

    public getScriptByteOrderMark(fileName: string): TypeScript.ByteOrderMark {
        return TypeScript.ByteOrderMark.None;
    }

    public getScriptSnapshot(fileName: string): TypeScript.IScriptSnapshot {
        var content = this.files.lookup(fileName);
        if (content === null) {
            console.error("getScriptSnapshot: file not found: " + fileName);
        }
        return TypeScript.ScriptSnapshot.fromString(content);
    }

    public getDiagnosticsObject(): TypeScript.Services.ILanguageServicesDiagnostics { return null; }
    public getLocalizedDiagnosticMessages(): any { return null; }

    //#endregion
    //#region TypeScript.ILogger

    public information(): boolean { return false; }
    public debug(): boolean { return false; }
    public warning(): boolean { return false; }
    public error(): boolean { return false; }
    public fatal(): boolean { return false; }
    public log(s: string): void { console.log("log:" + s); }

    //#endregion
    //#region TypeScript.IReferenceResolverHost

    public resolveRelativePath(path: string, directory: string): string { return null; }
    public fileExists(path: string): boolean { return false; }
    public directoryExists(path: string): boolean { return false; }
    public getParentDirectory(path: string): string { return null; }

    //#endregion
}
{% endhighlight %}

Now that we have a host we can create something which implements the ILanguageService. Because our host implementation isn't complete, not everything might work correctly but lets continue with a simple file. This is sufficient for our demo.

Let's create a new file called test.ts and we add the following to it:

{% highlight ts %}
class test {
    constructor(public param1: string) {
    }
}
var t = new test("param1");
t.param1 = "string";
{% endhighlight %}

When we want to get all the references for the class test we call the function getReferencesAtPosition. In this case we get two entries back(the class declaration and the one where we use the class).

{% highlight ts %}
console.dir(ls.getReferencesAtPosition("test.ts", 6));
console.dir(ls.getCompletionsAtPosition("test.ts", 99, true));
{% endhighlight %}

The values 6 is just after the word "class" and 99 is just after the "." (t.param1).

The getCompletionsAtPosition will return an array with only one item, if you add some more public members they will show up in this list too. Note, if you ask for autocompletion it will return all possible values, if you have two members (called foo and bar) and already did type the "f", you need to filter the list your self. Because it returns all results, you can create a really rich autocompletion mechanism where you can choose between "starts with", "contains" or some "fuzzy" autocompletion. For a rich autocompletion experience also add lib.d.ts as a reference!

Don't forget to take a look at the ILanguageService interface defined in the typescriptServices.d.ts file for other functions you can now use to extract information from the TypeScript compiler.

In a next post I will talk about using the languageservice from C#, a lot will be similar but there are some differences and off course the bridge from TypeScript to C#. Checkout the example project (attachment), note you need the Node.js tools for Visual Studio which you can find here: [https://nodejstools.codeplex.com](https://nodejstools.codeplex.com/).

I hope you guys liked this post about using the languageservice, feel free to comment when you encounter an issue or to provide feedback. Thanks for reading!
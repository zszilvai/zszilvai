---
title: 'Optimizing lodash size using Typescript compiler plugin'
description: Refactoring thousands of lines using less than a hundred one.
date: 2022-04-28T00:00:00Z
---
## Introduction
We all met with codebases of frontend applications where keeping bundle size minimal wasn't a priority during the initial phase of the development. And when it comes to the optimization of these sites handling some of the issues seems to be too painful even to estimate, how long will it take.

I'm sure you have seen this line before:

```ts
import * as _ from 'lodash'
```
Of course, since the library is not tree-shakeable this results in the entire lodash library being bundled together with the application code when you try to ship it. And nowadays, in the Lighthouse era when all the transferred bytes matter for your search positioning, having this amount of unnecessary code is avoidable.

Of course, refactoring all `_.something(...)` to a direct import and function call, like `import * as somethingFn from 'lodash/something' ... somethingFn(...)` is the solution, let's take a look at how widespread the problem is:

![1000+ Occurrences](/assets/images/posts/lodash_occurrences.png)

1000+ occurrence is high, and the task is pretty much robotic therefore I was thinking about all the wasted resources I have to spend on these robotic tasks and all the possible errors coming from the boring refactoring process of the same issue again and again...

But since the refactoring process is really straightforward, I was thinking about other possibilities - isn't there a compiler plugin that could do the same for me during the build?

## Compiler plugins to the rescue!

I remembered that there was a Babel plugin that did exactly the same, and I found [babel-plugin-lodash](https://github.com/lodash/babel-plugin-lodash) and [lodash-webpack-plugin](https://github.com/lodash/lodash-webpack-plugin) pretty fast, but I realized that I don't want to introduce Babel to the build process of the Angular application. So I decided to check out the extensibility of the Typescript compiler while being sure that if a plugin like this existed for Babel, it should exist for Typescript, they should have a similar plugin system after all. Right?

![Dwight false](/assets/images/posts/dwight.gif)

I was really disappointed, because not only the plugin does not exist but the TSC does not have an extendible plugin system - without [cloning the entire compiler](https://github.com/microsoft/TypeScript/issues/14419) at least. I think is highly ill-advised, keeping up a fork of the typescript repo for this purpose just looks overkill. But hey, here I figured out that things like [ttypescript](https://github.com/cevek/ttypescript) or [ts-patch](https://github.com/nonara/ts-patch) exist, and using them you can integrate a compiler plugin - a source transformer to be exact - pretty simple.

Ttypescript is a custom typescript compiler - it was made the way described in the GitHub issue, it is meant to be used as a custom typescript compiler - it has its own CLI command, if you would like to use it with ts-node, or Webpack, you have to explicitly specify to use a different compiler that the default TSC. On the other hand, ts-patch actually patches the installed typescript compiler and exposes the internal plugin system, eliminating the need to specify a custom compiler explicitly. At first glance, ttypescript would require changing the build process - something I was not sure I wanted to do at this point. Also, I explored the original code of tsc, which is patched by ts-patch and I found it less frequently modified, which made me think it's more resilient against a patch or minor version updates. So, I decided to go with ts-patch.

After running a test with ts-patch and finding it working like a charm, the next step was to evaluate the substitution process.

## Designing the plugin behavior

Consider a dummy example: 
```ts
import * as _ from 'lodash'; 
_.map(array, (item)=> item+1)
``` 
We have to import the function directly, so at first glance, it'd look like 
```ts
import * as map from 'lodash/map';  
map(array, (item)=> item+1)
```
By looking at this code, you can immediately determine the problem: if you've ever worked with RxJs, Leaflet, or anything that leads you to use a function or a variable named map, this can cause several issues from ambigous reference to name shadowing, preventing a compilation. So, I decided to prefix the function names with something like `__LODASH__` - just to be prepared for the unlikely case of someone naming something __LODASH__ in a file, I ended up with checking the existence of the string __LODASH__ in the code and append a random number to the end, if needed, like `__LODASH__38728_`. I decided to name the function consistently, transforming the names of them to snake case. This way an import will look like `import * as __LODASH__MAP__ from 'lodash'`.

To take it one step further, I decided to migrate everything to lodash-es compile-time: `import { map as __LODASH__MAP__ } from 'lodash-es'`. 

Replacing all occurrences of `_.map` to __LODASH__MAP__ and doing it subsequently with every other lodash function would require a tremendous amount of code substitution, but I realized we have a variable named _ (or whatever name the lodash import is), which we are now free to use, and this way we don't have to touch the occurrences of lodash at all!

This way

```ts
import * as _ from 'lodash';

_.deburr(_.toLower('déjà vu'));
```

becomes

```ts
import { deburr as __LODASH__DEBURR__, toLower as __LODASH__TO_LOWER__ } from 'lodash-es';

const _ = {
	deburr: __LODASH__DEBURR__,
	toLower: __LODASH__TO_LOWER__ 
}
_.deburr(_.toLower('déjà vu'));
```

## Implementation

After establishing the process in theorem, and getting familiar with the nature of compiler plugins (luckily, I already had experience working the TS-AST), the implementation plan was easy. 

I had to write a visitor that during visit checks if a lodash import declaration exists and the import declaration is a namespace import  (I assumed that there is only one declaration like this)

```ts
function isLodashNameNode(node: ts.Node): node is ts.ImportDeclaration {
 if (!ts.isImportDeclaration(node)) {
  return false
 }
 const moduleName = node.moduleSpecifier.getText()
 if (!/(['"])lodash(['"])/.test(moduleName)) {
  return false
 }
 const importClause: ts.ImportClause | undefined = node.importClause
 return !!importClause?.namedBindings && ts.isNamespaceImport(importClause.namedBindings)
}

function findLodashNameVisit(node: ts.Node) {
 if(isLodashNameNode(node)) {
  lodashName = (node.importClause!.namedBindings as ts.NamespaceImport).name.getText()
  lodashDeclaration = node
 }
 return ts.visitEachChild(node, findLodashNameVisit, context);
}
```
If it founds one, we save its name and location.

If we have this declaration, another visitor finds any call expression that accesses to a lodash property 
```ts
function isLodashCall(node: ts.Node, lodashName: string): node is ts.CallExpression {
 if (!ts.isCallExpression(node)) {
  return false
 }
 const callExpression = node as ts.CallExpression
 if (!ts.isPropertyAccessExpression(callExpression.expression)) {
  return false
 }
 const propertyAccessExpression = callExpression.expression as ts.PropertyAccessExpression
 return propertyAccessExpression.expression.getText() === lodashName
}

function findLodashCallsVisit(node: ts.Node) {
 if(isLodashCall(node, lodashName)) {
  const lodashFunctionName = (node.expression as ts.PropertyAccessExpression).name.getText()
  if (!usedLodashFunctions.includes(lodashFunctionName)) {
   usedLodashFunctions.push(lodashFunctionName)
  }
 }
 return ts.visitEachChild(node, findLodashCallsVisit, context);
}
```
Then we replace the import declaration with the `lodash-es` import, followed by the constant declaration and update the source file

```ts
const statements = [...rootNode.statements];
const lodashIndex = _.indexOf(statements, lodashDeclaration);
statements.splice(lodashIndex, 1, ...[
 context.factory.createImportDeclaration(...),
 context.factory.createVariableStatement(...)
]);
return context.factory.updateSourceFile(rootNode,statements)
```
And of course, to implement this with style, I used the `import * as _ from 'lodash'` declaration in the beginning.
But hey, it never ends up client-side, does it?

And the reduction in code size was considerable:

<<TODO images>>



The complete source code can be found in github:<<TODO source>>

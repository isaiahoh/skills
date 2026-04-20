# Rename Variable

```
let a = height * width;
```

```
let area = height * width;
```

## Motivation

Gives a variable a clearer name that better communicates its purpose, with naming care proportional to the variable's scope.

## Mechanics

- If the variable is used widely, consider Encapsulate Variable.

- Find all references to the variable, and change every one.

  If there are references from another code base, the variable is a published variable, and you cannot do this refactoring.

  If the variable does not change, you can copy it to one with the new name, then change gradually, testing after each change.

- Test.

## Example

The simplest case is a variable local to a single function -- just find each reference and change it.

For variables with wider scope, there may be many references across the code base:

```js
let tpHd = "untitled";
```

Some references access the variable:

```js
result += `<h1>${tpHd}</h1>`;
```

Others update it:

```js
tpHd = obj['articleTitle'];
```

Apply Encapsulate Variable:

```js
result += `<h1>${title()}</h1>`;

setTitle(obj['articleTitle']);

function title()       {return tpHd;}
function setTitle(arg) {tpHd = arg;}
```

Now rename the variable:

```js
let _title = "untitled";

function title()       {return _title;}
function setTitle(arg) {_title = arg;}
```

If the variable is used widely enough to need encapsulation for a rename, it's worth keeping it encapsulated behind functions for the future.

### Renaming a Constant

For a constant (or something that acts like a constant to clients), you can avoid encapsulation and rename gradually by copying:

```js
const cpyNm = "Acme Gooseberries";
```

Make a copy:

```js
const companyName = "Acme Gooseberries";
const cpyNm = companyName;
```

Gradually change references from the old name to the new name. When done, remove the copy. Declaring the new name and copying to the old name makes it easier to remove the old name and revert if a test fails.

This works for constants as well as for variables that are read-only to clients (such as an exported variable in JavaScript).

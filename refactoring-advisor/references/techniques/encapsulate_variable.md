# Encapsulate Variable

```
let defaultOwner = {firstName: "Martin", lastName: "Fowler"};
```

```
let defaultOwnerData = {firstName: "Martin", lastName: "Fowler"};
export function defaultOwner()       {return defaultOwnerData;}
export function setDefaultOwner(arg) {defaultOwnerData = arg;}
```

## Motivation

Wraps widely accessed data behind getter and setter functions so that all access is routed through a controlled interface. Encapsulation is much less important for immutable data, since it can be freely copied without risk of inconsistency.

## Mechanics

- Create encapsulating functions to access and update the variable.

- Run static checks.

- For each reference to the variable, replace with a call to the appropriate encapsulating function. Test after each replacement.

- Restrict the visibility of the variable.

  If not possible, consider renaming the variable and testing to detect remaining references.

- Test.

- If the value of the variable is a record, consider Encapsulate Record.

## Example

Some useful data held in a global variable:

```js
let defaultOwner = {firstName: "Martin", lastName: "Fowler"};
```

Referenced with code like:

```js
spaceship.owner = defaultOwner;
```

Updated like:

```js
defaultOwner = {firstName: "Rebecca", lastName: "Parsons"};
```

Start by defining functions to read and write the data:

```js
function getDefaultOwner()    {return defaultOwner;}
function setDefaultOwner(arg) {defaultOwner = arg;}
```

Replace references with calls to the getting function:

```js
spaceship.owner = getDefaultOwner();
```

Replace assignments with calls to the setting function:

```js
setDefaultOwner({firstName: "Rebecca", lastName: "Parsons"});
```

Test after each replacement.

Once done, restrict the visibility of the variable. In JavaScript, move the variable and accessor methods to their own file and only export the accessors:

defaultOwner.js...

```js
let defaultOwner = {firstName: "Martin", lastName: "Fowler"};
export function getDefaultOwner()    {return defaultOwner;}
export function setDefaultOwner(arg) {defaultOwner = arg;}
```

If you cannot restrict access, renaming the variable to something like `__privateOnly_defaultOwner` may help.

Remove the `get` prefix on the getter (but keep the `set` prefix on the setter):

defaultOwner.js...

```js
let defaultOwnerData = {firstName: "Martin", lastName: "Fowler"};
export function defaultOwner()       {return defaultOwnerData;}
export function setDefaultOwner(arg) {defaultOwnerData = arg;}
```

### Encapsulating the Value

The basic refactoring encapsulates the reference but doesn't control changes to the structure's contents:

```js
const owner1 = defaultOwner();
assert.equal("Fowler", owner1.lastName, "when set");
const owner2 = defaultOwner();
owner2.lastName = "Parsons";
assert.equal("Parsons", owner1.lastName, "after change owner2"); // is this ok?
```

To control changes to contents, the simplest option is to return a copy of the data from the getter:

defaultOwner.js...

```js
let defaultOwnerData = {firstName: "Martin", lastName: "Fowler"};
export function defaultOwner()       {return Object.assign({}, defaultOwnerData);}
export function setDefaultOwner(arg) {defaultOwnerData = arg;}
```

Be careful with copies: some code may expect to change shared data. An alternative to prevent changes is Encapsulate Record:

```js
let defaultOwnerData = {firstName: "Martin", lastName: "Fowler"};
export function defaultOwner()       {return new Person(defaultOwnerData);}
export function setDefaultOwner(arg) {defaultOwnerData = arg;}

class Person {
  constructor(data) {
    this._lastName = data.lastName;
    this._firstName = data.firstName
  }
  get lastName() {return this._lastName;}
  get firstName() {return this._firstName;}
  // and so on for other properties
```

Any attempt to reassign properties will cause an error. Different languages have different techniques for this.

It may also be worthwhile to make a copy in the setter to prevent accidents from changes to the source data. Copying and wrapping only work one level deep -- going deeper requires more levels of copies or object wrapping.

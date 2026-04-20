# Extract Class

```
class Person {
  get officeAreaCode() {return this._officeAreaCode;}
  get officeNumber()   {return this._officeNumber;}
```

```
class Person {
  get officeAreaCode() {return this._telephoneNumber.areaCode;}
  get officeNumber()   {return this._telephoneNumber.number;}
}
class TelephoneNumber {
  get areaCode() {return this._areaCode;}
  get number()   {return this._number;}
}
```

## Motivation

Splits a class that has taken on too many responsibilities by moving a cohesive subset of fields and methods into a new class.

## Mechanics

- Decide how to split the responsibilities of the class.

- Create a new child class to express the split-off responsibilities.

  If the responsibilities of the original parent class no longer match its name, rename the parent.

- Create an instance of the child class when constructing the parent and add a link from parent to child.

- Use Move Field on each field you wish to move. Test after each move.

- Use Move Function to move methods to the new child. Start with lower-level methods (those being called rather than calling). Test after each move.

- Review the interfaces of both classes, remove unneeded methods, change names to better fit the new circumstances.

- Decide whether to expose the new child. If so, consider applying Change Reference to Value to the child class.

## Example

I start with a simple person class:

class Person...

```js
get name()    {return this._name;}
set name(arg) {this._name = arg;}
get telephoneNumber() {return `(${this.officeAreaCode}) ${this.officeNumber}`;}
get officeAreaCode()    {return this._officeAreaCode;}
set officeAreaCode(arg) {this._officeAreaCode = arg;}
get officeNumber() {return this._officeNumber;}
set officeNumber(arg) {this._officeNumber = arg;}
```

I can separate the telephone number behavior into its own class. I start by defining an empty telephone number class:

```js
class TelephoneNumber {
}
```

Next, I create an instance of telephone number when constructing the person:

class Person...

```js
constructor() {
  this._telephoneNumber = new TelephoneNumber();
}
```

I then use Move Field on one of the fields.

class TelephoneNumber...

```js
get officeAreaCode()    {return this._officeAreaCode;}
set officeAreaCode(arg) {this._officeAreaCode = arg;}
```

class Person...

```js
get officeAreaCode()    {return this._telephoneNumber.officeAreaCode;}
set officeAreaCode(arg) {this._telephoneNumber.officeAreaCode = arg;}
```

I test, then move the next field.

class TelephoneNumber...

```js
get officeNumber() {return this._officeNumber;}
set officeNumber(arg) {this._officeNumber = arg;}
```

class Person...

```js
get officeNumber() {return this._telephoneNumber.officeNumber;}
set officeNumber(arg) {this._telephoneNumber.officeNumber = arg;}
```

Test again, then move the telephone number method.

class TelephoneNumber...

```js
get telephoneNumber() {return `(${this.officeAreaCode}) ${this.officeNumber}`;}
```

class Person...

```js
get telephoneNumber() {return this._telephoneNumber.telephoneNumber;}
```

Now I tidy up. Having "office" as part of the telephone number code makes no sense, so I rename them.

class TelephoneNumber...

```js
get areaCode()    {return this._areaCode;}
set areaCode(arg) {this._areaCode = arg;}

get number()    {return this._number;}
set number(arg) {this._number = arg;}
```

class Person...

```js
get officeAreaCode()    {return this._telephoneNumber.areaCode;}
set officeAreaCode(arg) {this._telephoneNumber.areaCode = arg;}
get officeNumber()    {return this._telephoneNumber.number;}
set officeNumber(arg) {this._telephoneNumber.number = arg;}
```

The telephone number method on the telephone number class also doesn't make much sense, so I apply Rename Function.

class TelephoneNumber...

```js
toString() {return `(${this.areaCode}) ${this.number}`;}
```

class Person...

```js
get telephoneNumber() {return this._telephoneNumber.toString();}
```

Telephone numbers are generally useful, so I'd expose the new object to clients by replacing "office" methods with accessors for the telephone number. The telephone number will work better as a Value Object, so I would apply Change Reference to Value first.

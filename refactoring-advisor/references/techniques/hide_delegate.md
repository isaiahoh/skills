# Hide Delegate

```
manager = aPerson.department.manager;
```

```
manager = aPerson.manager;

class Person {
  get manager() {return this.department.manager;}
```

## Motivation

Places a delegating method on the server object so clients no longer need to reach through it to call the delegate directly, reducing coupling to the delegate's interface.

## Mechanics

- For each method on the delegate, create a simple delegating method on the server.

- Adjust the client to call the server. Test after each change.

- If no client needs to access the delegate anymore, remove the server's accessor for the delegate.

- Test.

## Example

I start with a person and a department.

class Person...

```js
constructor(name) {
  this._name = name;
}
get name() {return this._name;}
get department()    {return this._department;}
set department(arg) {this._department = arg;}
```

class Department...

```js
get chargeCode()    {return this._chargeCode;}
set chargeCode(arg) {this._chargeCode = arg;}
get manager()    {return this._manager;}
set manager(arg) {this._manager = arg;}
```

Some client code wants to know the manager of a person. To do this, it needs to get the department first.

client code...

```js
manager = aPerson.department.manager;
```

This reveals to the client how the department class works and that the department is responsible for tracking the manager. I can reduce this coupling by creating a simple delegating method on person:

class Person...

```js
get manager() {return this._department.manager;}
```

I now change all clients of person to use this new method:

client code...

```js
manager = aPerson.manager;
```

Once I've made the change for all methods of department and for all clients of person, I can remove the `department` accessor on person.

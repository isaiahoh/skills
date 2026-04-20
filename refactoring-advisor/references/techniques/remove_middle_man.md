# Remove Middle Man

```
manager = aPerson.manager;

class Person {
  get manager() {return this.department.manager;}
```

```
manager = aPerson.department.manager;
```

## Motivation

Removes forwarding methods from a server class that has become a pure middle man, letting clients access the delegate directly instead.

## Mechanics

- Create a getter for the delegate.

- For each client use of a delegating method, replace the call to the delegating method by chaining through the accessor. Test after each replacement.

  If all calls to a delegating method are replaced, you can delete the delegating method.

  With automated refactorings, you can use Encapsulate Variable on the delegate field and then Inline Function on all the methods that use it.

## Example

I begin with a person class that uses a linked department object to determine a manager.

client code...

```js
manager = aPerson.manager;
```

class Person...

```js
get manager() {return this._department.manager;}
```

class Department...

```js
get manager()    {return this._manager;}
```

This is simple to use and encapsulates the department. But if lots of methods are doing this, there are too many simple delegations on person. First, I make an accessor for the delegate:

class Person...

```js
get department()    {return this._department;}
```

Now I modify each client to use the department directly.

client code...

```js
manager = aPerson.department.manager;
```

Once I've done this with all the clients, I can remove the `manager` method from `Person`. I can repeat this for any other simple delegations on `Person`.

There is a useful variation using automated refactorings. First, use Encapsulate Variable on `department`. This changes the manager getter to use the public department getter:

class Person...

```js
get manager() {return this.department.manager;}
```

The change is subtle in JavaScript--by removing the underscore from `department` I'm using the new getter rather than accessing the field directly.

Then apply Inline Function on the manager method to replace all the callers at once.

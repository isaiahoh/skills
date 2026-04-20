# Remove Setting Method

## Motivation

Removes a setter for a field that should not change after object creation, making the field immutable and set only in the constructor to signal that post-construction updates are not allowed.

## Mechanics

- If the value that's being set isn't provided to the constructor, use Change Function Declaration to add it. Add a call to the setting method within the constructor.

  If you wish to remove several setting methods, add all their values to the constructor at once. This simplifies the later steps.

- Remove each call of a setting method outside of the constructor, using the new constructor value instead. Test after each one.

  If you can't replace the call to the setter by creating a new object (because you are updating a shared reference object), abandon the refactoring.

- Use Inline Function on the setting method. Make the field immutable if possible.
- Test.

## Example

I have a simple person class.

class Person...

```
get name()    {return this._name;}
set name(arg) {this._name = arg;}
get id()    {return this._id;}
set id(arg) {this._id = arg;}
```

At the moment, I create a new object with code like this:

```
const martin = new Person();
martin.name = "martin";
martin.id = "1234";
```

The name of a person may change after it's created, but the ID does not. To make this clear, I want to remove the setting method for ID.

I still need to set the ID initially, so I'll use Change Function Declaration to add it to the constructor.

class Person...

```
constructor(id) {
  this.id = id;
}
```

I then adjust the creation script to set the ID via the constructor.

```
const martin = new Person("1234");
martin.name = "martin";
martin.id = "1234";
```

I do this in each place I create a person, testing after each change.

When they are all done, I can apply Inline Function to the setting method.

class Person...

```
constructor(id) {
  this._id = id;
}
get name()    {return this._name;}
set name(arg) {this._name = arg;}
get id()    {return this._id;}
```

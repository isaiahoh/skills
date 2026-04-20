# Pull Up Constructor Body

```
class Party {...}

class Employee extends Party {
  constructor(name, id, monthlyCost) {
    super();
    this._id = id;
    this._name = name;
    this._monthlyCost = monthlyCost;
  }
}
```

```
class Party {
  constructor(name){
    this._name = name;
  }
}

class Employee extends Party {
  constructor(name, id, monthlyCost) {
    super(name);
    this._id = id;
    this._monthlyCost = monthlyCost;
  }
}
```

## Motivation

Moves common constructor statements from subclasses into the superclass constructor. If this gets messy, reach for Replace Constructor with Factory Function instead.

## Mechanics

- Define a superclass constructor, if one doesn't already exist. Ensure it's called by subclass constructors.

- Use Slide Statements to move any common statements to just after the super call.

- Remove the common code from each subclass and put it in the superclass. Add to the super call any constructor parameters referenced in the common code.

- Test.
- If there is any common code that cannot move to the start of the constructor, use Extract Function followed by Pull Up Method.

## Example

I start with the following code:

```
class Party {}


class Employee extends Party {
  constructor(name, id, monthlyCost) {
    super();
    this._id = id;
    this._name = name;
    this._monthlyCost = monthlyCost;
  }
  // rest of class...


class Department extends Party {
  constructor(name, staff){
    super();
    this._name = name;
    this._staff = staff;
  }
  // rest of class...
```

The common code here is the assignment of the name. I use Slide Statements to move the assignment in `Employee` next to the call to `super()`:

```
class Employee extends Party {
  constructor(name, id, monthlyCost) {
    super();
    this._name = name;
    this._id = id;
    this._monthlyCost = monthlyCost;
  }
  // rest of class...
```

With that tested, I move the common code to the superclass. Since that code contains a reference to a constructor argument, I pass that in as a parameter.

class Party...

```
constructor(name){
  this._name = name;
}
```

class Employee...

```
constructor(name, id, monthlyCost) {
  super(name);
  this._id = id;
  this._monthlyCost = monthlyCost;
}
```

class Department...

```
constructor(name, staff){
  super(name);
  this._staff = staff;
}
```

Run the tests, and I'm done.

Most of the time, constructor behavior will work like this: do the common elements first (with a `super` call), then do extra work that the subclass needs. Occasionally, however, there is some common behavior later.

Consider this example:

class Employee...

```
constructor (name) {...}

get isPrivileged() {...}

assignCar() {...}
```

class Manager extends Employee...

```
constructor(name, grade) {
  super(name);
  this._grade = grade;
  if (this.isPrivileged) this.assignCar(); // every subclass does this
}

get isPrivileged() {
  return this._grade > 4;
}
```

The call to `isPrivileged` can't be made until after the `grade` field is assigned, and that can only be done in the subclass.

In this case, I do Extract Function on the common code:

class Manager...

```
constructor(name, grade) {
  super(name);
  this._grade = grade;
  this.finishConstruction();
}

finishConstruction() {
  if (this.isPrivileged) this.assignCar();
}
```

Then, I use Pull Up Method to move it to the superclass.

class Employee...

```
finishConstruction() {
  if (this.isPrivileged) this.assignCar();
}
```

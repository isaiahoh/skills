# Replace Type Code with Subclasses

```
function createEmployee(name, type) {
  return new Employee(name, type);
}
```

```
function createEmployee(name, type) {
  switch (type) {
    case "engineer": return new Engineer(name);
    case "salesman": return new Salesman(name);
    case "manager":  return new Manager (name);
  }
```

## Motivation

Replaces a type code field with subclasses, enabling polymorphism and scoping of type-specific fields/methods. Use direct subclassing when simpler; use indirect subclassing (via Replace Primitive with Object on the type code first) when inheritance is needed for another axis or the type is mutable.

## Mechanics

- Self-encapsulate the type code field.

- Pick one type code value. Create a subclass for that type code. Override the type code getter to return the literal type code value.

- Create selector logic to map from the type code parameter to the new subclass.

  With direct inheritance, use Replace Constructor with Factory Function and put the selector logic in the factory. With indirect inheritance, the selector logic may stay in the constructor.

- Test.
- Repeat creating the subclass and adding to the selector logic for each type code value. Test after each change.

- Remove the type code field.

- Test.
- Use Push Down Method and Replace Conditional with Polymorphism on any methods that use the type code accessors. Once all are replaced, you can remove the type code accessors.

## Example

I'll start with this employee example:

class Employee...

```
constructor(name, type){
  this.validateType(type);
  this._name = name;
  this._type = type;
}
validateType(arg) {
  if (!["engineer", "manager", "salesman"].includes(arg))
    throw new Error(`Employee cannot be of type ${arg}`);
}
toString() {return `${this._name} (${this._type})`;}
```

My first step is to use Encapsulate Variable to self-encapsulate the type code.

class Employee...

```
get type() {return this._type;}
toString() {return `${this._name} (${this.type})`;}
```

Note that `toString` uses the new getter by removing the underscore.

I pick one type code, the engineer, to start with. I use direct inheritance, subclassing the employee class itself. The employee subclass just overrides the type code getter with the appropriate literal value.

```
class Engineer extends Employee {
   get type() {return "engineer";}
}
```

Although JavaScript constructors can return other objects, things will get messy if I try to put selector logic in there, since that logic gets intertwined with field initialization. So I use Replace Constructor with Factory Function to create a new space for it.

```
function createEmployee(name, type) {
  return new Employee(name, type);
}
```

To use the new subclass, I add selector logic into the factory.

```
function createEmployee(name, type) {
  switch (type) {
    case "engineer": return new Engineer(name, type);
  }
  return new Employee(name, type);
}
```

I test to ensure that worked, then alter the return value of the engineer's override and test again to ensure the test fails -- verifying the subclass is being used. I correct the return value and continue with the other cases, testing after each change.

```
class Salesman extends Employee {
   get type() {return "salesman";}
}

class Manager extends Employee {
   get type() {return "manager";}
}

function createEmployee(name, type) {
  switch (type) {
    case "engineer": return new Engineer(name, type);
    case "salesman": return new Salesman(name, type);
    case "manager":  return new Manager (name, type);
  }
  return new Employee(name, type);
}
```

Once I'm done with them all, I can remove the type code field and the superclass getting method (the ones in the subclasses remain).

class Employee...

```
constructor(name, type){
  this.validateType(type);
  this._name = name;
  this._type = type;
}

get type() {return this._type;}
toString() {return `${this._name} (${this.type})`;}
```

After testing, I can remove the validation logic since the switch is effectively doing the same thing.

class Employee...

```
constructor(name, type){
  this.validateType(type);
  this._name = name;
}
```

```
function createEmployee(name, type) {
  switch (type) {
    case "engineer": return new Engineer(name, type);
    case "salesman": return new Salesman(name, type);
    case "manager":  return new Manager (name, type);
    default: throw new Error(`Employee cannot be of type ${type}`);
  }
  return new Employee(name, type);
}
```

The type argument to the constructor is now useless, so it falls victim to Change Function Declaration.

class Employee...

```
constructor(name, type){
  this._name = name;
}
```

```
function createEmployee(name, type) {
  switch (type) {
    case "engineer": return new Engineer(name, type);
    case "salesman": return new Salesman(name, type);
    case "manager":  return new Manager (name, type);
    default: throw new Error(`Employee cannot be of type ${type}`);
  }
}
```

The type code accessors on the subclasses (`get type`) can also be removed over time. Use Replace Conditional with Polymorphism and Push Down Method to eliminate consumers, then apply Remove Dead Code to the getters.

## Example: Using Indirect Inheritance

If there are already existing subclasses (e.g., part-time and full-time employees) or if the type needs to be mutable, use indirect inheritance instead.

class Employee...

```
constructor(name, type){
  this.validateType(type);
  this._name = name;
  this._type = type;
}
validateType(arg) {
  if (!["engineer", "manager", "salesman"].includes(arg))
    throw new Error(`Employee cannot be of type ${arg}`);
}
get type()    {return this._type;}
set type(arg) {this._type = arg;}

get capitalizedType() {
  return this._type.charAt(0).toUpperCase() + this._type.substr(1).toLowerCase();
}
toString() {
  return `${this._name} (${this.capitalizedType})`;
}
```

My first step is to use Replace Primitive with Object on the type code.

```
class EmployeeType {
  constructor(aString) {
    this._value = aString;
  }
  toString() {return this._value;}
}
```

class Employee...

```
constructor(name, type){
  this.validateType(type);
  this._name = name;
  this.type = type;
}
validateType(arg) {
  if (!["engineer", "manager", "salesman"].includes(arg))
    throw new Error(`Employee cannot be of type ${arg}`);
}
get typeString()    {return this._type.toString();}
get type()    {return this._type;}
set type(arg) {this._type = new EmployeeType(arg);}

get capitalizedType() {
  return this.typeString.charAt(0).toUpperCase()
    + this.typeString.substr(1).toLowerCase();
}

toString() {
  return `${this._name} (${this.capitalizedType})`;
}
```

I then apply the usual mechanics of Replace Type Code with Subclasses to the employee type.

class Employee...

```
set type(arg) {this._type = Employee.createEmployeeType(arg);}

static createEmployeeType(aString) {
  switch(aString) {
    case "engineer": return new Engineer();
    case "manager":  return new Manager ();
    case "salesman": return new Salesman();
    default: throw new Error(`Employee cannot be of type ${aString}`);
  }
}
```

```
class EmployeeType {
}
class Engineer extends EmployeeType {
  toString() {return "engineer";}
}
class Manager extends EmployeeType {
  toString() {return "manager";}
}
class Salesman extends EmployeeType {
  toString() {return "salesman";}
}
```

I prefer to leave `EmployeeType` in place as it makes the relationship between the subclasses explicit and provides a handy spot for moving other behavior, such as the capitalization logic.

class Employee...

```
toString() {
  return `${this._name} (${this.type.capitalizedName})`;
}
```

class EmployeeType...

```
get capitalizedName() {
  return this.toString().charAt(0).toUpperCase()
    + this.toString().substr(1).toLowerCase();
}
```

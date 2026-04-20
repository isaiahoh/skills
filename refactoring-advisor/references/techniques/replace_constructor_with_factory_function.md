# Replace Constructor with Factory Function

## Motivation

Replaces a constructor call with a factory function that can return subclasses, use a descriptive name, and be passed around like any other function.

## Mechanics

- Create a factory function, its body being a call to the constructor.
- Replace each call to the constructor with a call to the factory function.
- Test after each change.
- Limit the constructor's visibility as much as possible.

## Example

Consider an employee class:

class Employee...

```
constructor (name, typeCode) {
  this._name = name;
  this._typeCode = typeCode;
}
get name() {return this._name;}
get type() {
  return Employee.legalTypeCodes[this._typeCode];
}
static get legalTypeCodes() {
  return {"E": "Engineer", "M": "Manager", "S": "Salesman"};
}
```

This is used from:

caller...

```
candidate = new Employee(document.name, document.empType);
```

and

caller...

```
const leadEngineer = new Employee(document.leadEngineer, 'E');
```

My first step is to create the factory function. Its body is a simple delegation to the constructor.

top level...

```
function createEmployee(name, typeCode) {
  return new Employee(name, typeCode);
}
```

I then find the callers of the constructor and change them, one at a time, to use the factory function instead.

The first one is obvious:

caller...

```
candidate = createEmployee(document.name, document.empType);
```

With the second case, I could use the new factory function like this:

caller...

```
const leadEngineer = createEmployee(document.leadEngineer, 'E');
```

But I don't like using the type code here -- it's generally a bad smell to pass a code as a literal string. So I prefer to create a new factory function that embeds the kind of employee I want into its name.

caller...

```
const leadEngineer = createEngineer(document.leadEngineer);
```

top level...

```
function createEngineer(name) {
  return new Employee(name, 'E');
}
```

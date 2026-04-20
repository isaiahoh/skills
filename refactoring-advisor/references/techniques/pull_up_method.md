# Pull Up Method

```
class Employee {...}

class Salesman extends Employee {
  get name() {...}
}

class Engineer extends Employee {
  get name() {...}
}
```

```
class Employee {
  get name() {...}
}

class Salesman extends Employee {...}
class Engineer extends Employee {...}
```

## Motivation

Moves identical (or near-identical) methods from subclasses into the superclass to eliminate duplication. If methods differ, refactor them until they match first; if they reference subclass-only features, Pull Up Field on those elements first.

## Mechanics

- Inspect methods to ensure they are identical.

  If they do the same thing but are not identical, refactor them until they have identical bodies.

- Check that all method calls and field references inside the method body refer to features that can be called from the superclass.

- If the methods have different signatures, use Change Function Declaration to get them to the one you want to use on the superclass.

- Create a new method in the superclass. Copy the body of one of the methods over to it.

- Run static checks.
- Delete one subclass method.

- Test.
- Keep deleting subclass methods until they are all gone.

## Example

I have two subclass methods that do the same thing.

class Employee extends Party...

```
get annualCost() {
  return this.monthlyCost * 12;
}
```

class Department extends Party...

```
get totalAnnualCost() {
  return this.monthlyCost * 12;
}
```

Both refer to `monthlyCost`, which isn't defined on the superclass but is present in both subclasses. In a dynamic language that's OK; in a static language, I'd need to define an abstract method on `Party`.

The methods have different names, so I Change Function Declaration to make them the same.

class Department...

```
get annualCost() {
  return this.monthlyCost * 12;
}
```

I copy the method from one subclass and paste it into the superclass.

class Party...

```
get annualCost() {
  return this.monthlyCost * 12;
}
```

I first remove `annualCost` from `Employee`, test, then remove it from `Department`.

That completes the refactoring, but `annualCost` calls `monthlyCost`, which doesn't appear in `Party`. There is value in signaling that subclasses of `Party` should provide an implementation for `monthlyCost`, particularly if more subclasses get added later. A good way to provide this signal is a trap method:

class Party...

```
get monthlyCost() {
  throw new SubclassResponsibilityError();
}
```

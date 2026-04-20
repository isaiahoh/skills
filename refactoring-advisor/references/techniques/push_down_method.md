# Push Down Method

```
class Employee {
  get quota {...}
}

class Engineer extends Employee {...}
class Salesman extends Employee {...}
```

```
class Employee {...}
class Engineer extends Employee {...}
class Salesman extends Employee {
  get quota {...}
}
```

## Motivation

If a method is only relevant to one subclass (or a small proportion of subclasses), removing it from the superclass and putting it only on the subclass(es) makes that clearer. This can only be done if the caller knows it's working with a particular subclass -- otherwise, use Replace Conditional with Polymorphism with some placebo behavior on the superclass.

## Mechanics

- Copy the method into every subclass that needs it.

- Remove the method from the superclass.

- Test.
- Remove the method from each subclass that doesn't need it.

- Test.

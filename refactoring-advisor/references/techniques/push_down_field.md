# Push Down Field

```
class Employee {        // Java
  private String quota;
}

class Engineer extends Employee {...}
class Salesman extends Employee {...}
```

```
class Employee {...}
class Engineer extends Employee {...}

class Salesman extends Employee {
  protected String quota;
}
```

## Motivation

If a field is only used by one subclass (or a small proportion of subclasses), move it to those subclasses.

## Mechanics

- Declare field in all subclasses that need it.

- Remove the field from the superclass.

- Test.
- Remove the field from all subclasses that don't need it.

- Test.

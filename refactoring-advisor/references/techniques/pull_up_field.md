# Pull Up Field

```
class Employee {...} // Java

class Salesman extends Employee {
  private String name;
}

class Engineer extends Employee {
  private String name;
}
```

```
class Employee {
  protected String name;
}

class Salesman extends Employee {...}
class Engineer extends Employee {...}
```

## Motivation

Moves duplicate fields from subclasses into the superclass, removing data duplication and enabling behavior that uses the field to be pulled up too.

## Mechanics

- Inspect all users of the candidate field to ensure they are used in the same way.

- If the fields have different names, use Rename Field to give them the same name.

- Create a new field in the superclass.

  The new field will need to be accessible to subclasses (`protected` in common languages).

- Delete the subclass fields.

- Test.

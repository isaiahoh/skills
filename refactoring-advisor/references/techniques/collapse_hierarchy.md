# Collapse Hierarchy

```
class Employee {...}
class Salesman extends Employee {...}
```

```
class Employee {...}
```

## Motivation

As a hierarchy evolves, a class and its parent sometimes become no longer different enough to be worth keeping separate. Merge them together.

## Mechanics

- Choose which one to remove.

  Choose based on which name makes most sense in the future. If neither name is best, pick one arbitrarily.

- Use Pull Up Field, Push Down Field, Pull Up Method, and Push Down Method to move all the elements into a single class.

- Adjust any references to the victim to change them to the class that will stay.

- Remove the empty class.

- Test.

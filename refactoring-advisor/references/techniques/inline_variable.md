# Inline Variable

```
let basePrice = anOrder.basePrice;
return (basePrice > 1000);
```

```
return anOrder.basePrice > 1000;
```

## Motivation

Removes a variable whose name communicates no more than the expression itself, replacing all references with the original expression.

## Mechanics

- Check that the right-hand side of the assignment is free of side effects.

- If the variable isn't already declared immutable, do so and test.

  This checks that it's only assigned to once.

- Find the first reference to the variable and replace it with the right-hand side of the assignment.

- Test.

- Repeat replacing references to the variable until you've replaced all of them.

- Remove the declaration and assignment of the variable.

- Test.

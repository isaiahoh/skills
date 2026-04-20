# Replace Magic Literal

```
function potentialEnergy(mass, height) {
  return mass * 9.81 * height;
}
```

```
const STANDARD_GRAVITY = 9.81;
function potentialEnergy(mass, height) {
  return mass * STANDARD_GRAVITY * height;
}
```

## Motivation

Replace a literal value whose meaning is not self-evident with a named constant (or function) that communicates its purpose.

## Mechanics

- Declare a constant and set it to the magic literal.

- Search for all appearances of the literal.

- For each, see if its use matches the meaning of the new constant. If so, replace it with the new constant and test.

- Once done, a good way to check is to change the value of the constant and see if the tests fully reflect this change. This isn't always possible, but handy if it is.

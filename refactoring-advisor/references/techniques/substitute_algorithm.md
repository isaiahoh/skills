# Substitute Algorithm

```
function foundPerson(people) {
  for(let i = 0; i < people.length; i++) {
    if (people[i] === "Don") {
      return "Don";
    }
    if (people[i] === "John") {
      return "John";
    }
    if (people[i] === "Kent") {
      return "Kent";
    }
  }
  return "";
}
```

```
function foundPerson(people) {
  const candidates = ["Don", "John", "Kent"];
  return people.find(p => candidates.includes(p)) || '';
}
```

## Motivation

Replaces the body of a function with a clearer or simpler algorithm that produces the same results. Decompose the method as much as possible before substituting--replacing a large, complex algorithm in one step is error-prone.

## Mechanics

- Arrange the code to be replaced so that it fills a complete function.

- Prepare tests using this function only, to capture its behavior.

- Prepare your alternative algorithm.

- Run static checks.
- Run tests to compare the output of the old algorithm to the new one. If they are the same, you're done. Otherwise, use the old algorithm for comparison in testing and debugging.

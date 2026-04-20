# Replace Inline Code with Function Call

```
let appliesToMass = false;
for(const s of states) {
  if (s === "MA") appliesToMass = true;
}
```

```
appliesToMass = states.includes("MA");
```

## Motivation

Replace inline code with a call to an existing function that does the same thing, eliminating duplication and replacing mechanics with a meaningful name. Do not replace if the similarity is coincidental -- if changing the function body should not affect the inline behavior, keep them separate.

## Mechanics

- Replace the inline code with a call to the existing function.
- Test.

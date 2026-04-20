# Remove Dead Code

```
if(false) {
  doSomethingThatUsedToMatter();
}
```

## Motivation

Delete code that is no longer executed so readers don't waste time trying to understand logic that has no effect on program behavior.

## Mechanics

- If the dead code can be referenced from outside, e.g., when it's a full function, do a search to check for callers.
- Remove the dead code.
- Test.

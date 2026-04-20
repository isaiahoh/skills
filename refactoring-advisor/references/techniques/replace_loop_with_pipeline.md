# Replace Loop with Pipeline

```
const names = [];
for (const i of input) {
  if (i.job === "programmer")
    names.push(i.name);
}
```

```
const names = input
  .filter(i => i.job === "programmer")
  .map(i => i.name);
```

## Motivation

Replaces a loop that processes a collection with a pipeline of operations (map, filter, slice, reduce, etc.), making the processing steps read top-to-bottom as a series of transformations.

## Mechanics

- Create a new variable for the loop's collection. This may be a simple copy of an existing variable.

- Starting at the top, take each bit of behavior in the loop and replace it with a collection pipeline operation in the derivation of the loop collection variable. Test after each change.

- Once all behavior is removed from the loop, remove it. If it assigns to an accumulator, assign the pipeline result to the accumulator.

## Example

A function picks out offices in India from CSV data and returns their cities and telephone numbers:

```js
function acquireData(input) {
  const lines = input.split("\n");
  let firstLine = true;
  const result = [];
  for (const line of lines) {
    if (firstLine) {
      firstLine = false;
      continue;
    }
    if (line.trim() === "") continue;
    const record = line.split(",");
    if (record[1].trim() === "India") {
      result.push({city: record[0].trim(), phone: record[2].trim()});
    }
  }
  return result;
}
```

First, create a separate variable for the loop to work over:

```js
const loopItems = lines;
for (const line of loopItems) {
```

The first part of the loop skips the CSV header. Replace it with a `slice`:

```js
const loopItems = lines
  .slice(1);
```

This also lets me delete `firstLine` -- deleting control variables is always satisfying.

Next, remove blank lines with `filter`:

```js
const loopItems = lines
  .slice(1)
  .filter(line => line.trim() !== "")
  ;
```

Use `map` to split lines into field arrays:

```js
const loopItems = lines
  .slice(1)
  .filter(line => line.trim() !== "")
  .map(line => line.split(","))
  ;
```

`filter` again for India records:

```js
const loopItems = lines
  .slice(1)
  .filter(line => line.trim() !== "")
  .map(line => line.split(","))
  .filter(record => record[1].trim() === "India")
  ;
```

`map` to the output record form:

```js
const loopItems = lines
  .slice(1)
  .filter(line => line.trim() !== "")
  .map(line => line.split(","))
  .filter(record => record[1].trim() === "India")
  .map(record => ({city: record[0].trim(), phone: record[2].trim()}))
  ;
```

Now all the loop does is assign values to the accumulator, so remove it and assign the pipeline result directly. After cleanup (inline result, rename lambdas, table-style layout):

```js
function acquireData(input) {
  const lines = input.split("\n");
  return lines
    .slice  (1)
    .filter (line   => line.trim() !== "")
    .map    (line   => line.split(","))
    .filter (fields => fields[1].trim() === "India")
    .map    (fields => ({city: fields[0].trim(), phone: fields[2].trim()}))
    ;
}
```

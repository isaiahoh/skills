# Replace Control Flag with Break

```
for (const p of people) {
  if (! found) {
    if ( p === "Don") {
      sendAlert();
      found = true;
    }
```

```
for (const p of people) {
  if ( p === "Don") {
    sendAlert();
    break;
  }
```

## Motivation

Replace a control-flag variable that governs loop or function flow with `break`, `continue`, or `return` statements for clearer control flow.

## Mechanics

- Consider using Extract Function on the code that uses the control flag.
- For each update of the control flag, replace the update with an appropriate control statement. Test after each change. These are usually `return`, `break`, or `continue`.
- Once all the updates are done, remove the control flag.

## Example

Here's some code that checks a list of people looking for hard-coded suspicious characters.

```js
// some unimportant code
let found = false;
for (const p of people) {
  if (! found) {
    if ( p === "Don") {
      sendAlert();
      found = true;
    }
    if ( p === "John") {
      sendAlert();
      found = true;
    }
  }
}
// more code
```

The control flag is the variable `found` which is used to alter the control flow. My usual first move is to use Extract Function to put it into its own function so I can see it in isolation.

```js
// some unimportant code
checkForMiscreants(people);
// more code


function checkForMiscreants(people) {
  let found = false;
  for (const p of people) {
    if (! found) {
      if ( p === "Don") {
        sendAlert();
        found = true;
      }
      if ( p === "John") {
        sendAlert();
        found = true;
      }
    }
  }
}
```

When the control flag becomes true, I'm done with the loop. Since there is nothing else for the function to do, I might as well return. I do small steps, inserting a single `return` and then testing.

```js
function checkForMiscreants(people) {
  let found = false;
  for (const p of people) {
    if (! found) {
      if ( p === "Don") {
        sendAlert();
        return;
      }
      if ( p === "John") {
        sendAlert();
        found = true;
      }
    }
  }
}
```

I continue until I've removed all the places where the control flag is updated.

```js
function checkForMiscreants(people) {
  let found = false;
  for (const p of people) {
    if (! found) {
      if ( p === "Don") {
        sendAlert();
        return;
      }
      if ( p === "John") {
        sendAlert();
        return;
      }
    }
  }
}
```

Once I've removed all the updates, I can remove all other references to the control flag.

```js
function checkForMiscreants(people) {
  for (const p of people) {
    if ( p === "Don") {
      sendAlert();
      return;
    }
    if ( p === "John") {
      sendAlert();
      return;
    }
  }
}
```

This refactoring is done. But once I've done that, I'd carry on to end up with this code:

```js
function checkForMiscreants(people) {
  if (people.some(p => ["Don", "John"].includes(p))) sendAlert();
}
```

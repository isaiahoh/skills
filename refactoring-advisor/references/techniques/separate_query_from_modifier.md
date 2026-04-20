# Separate Query from Modifier

## Motivation

Splits a function that both returns a value and has side effects into a separate query (pure) and modifier, enforcing command-query separation. Caching a query result in a field is acceptable since it doesn't produce observable side effects.

## Mechanics

- Copy the function, name it as a query.

  Look at what is returned. If the query populates a variable, the variable name should suggest a good name.

- Remove any side effects from the new query function.

- Run static checks.

- Find each call of the original method. If that call uses the return value, replace the original call with a call to the query and insert a call to the original method below it. Test after each change.

- Remove return values from original.

- Test.

- Often after doing this there will be duplication between the query and the original method that can be tidied up.

## Example

Here is a function that scans a list of names for a miscreant. If it finds one, it returns the name of the bad guy and sets off the alarms. It only does this for the first miscreant it finds.

```javascript
function alertForMiscreant (people) {
  for (const p of people) {
    if (p === "Don") {
      setOffAlarms();
      return "Don";
    }
    if (p === "John") {
      setOffAlarms();
      return "John";
    }
  }
  return "";
}
```

I begin by copying the function, naming it after the query aspect of the function.

```javascript
function findMiscreant (people) {
  for (const p of people) {
    if (p === "Don") {
      setOffAlarms();
      return "Don";
    }
    if (p === "John") {
      setOffAlarms();
      return "John";
    }
  }
  return "";
}
```

I remove the side effects from this new query.

```javascript
function findMiscreant (people) {
  for (const p of people) {
    if (p === "Don") {
      return "Don";
    }
    if (p === "John") {
      return "John";
    }
  }
  return "";
}
```

I now go to each caller and replace it with a call to the query, followed by a call to the modifier. So

```javascript
const found = alertForMiscreant(people);
```

changes to

```javascript
const found = findMiscreant(people);
alertForMiscreant(people);
```

I now remove the return values from the modifier.

```javascript
function alertForMiscreant (people) {
  for (const p of people) {
    if (p === "Don") {
      setOffAlarms();
      return;
    }
    if (p === "John") {
      setOffAlarms();
      return;
    }
  }
  return;
}
```

Now I have a lot of duplication between the original modifier and the new query, so I can use Substitute Algorithm so that the modifier uses the query.

```javascript
function alertForMiscreant (people) {
  if (findMiscreant(people) !== "") setOffAlarms();
}
```

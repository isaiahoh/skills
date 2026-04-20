# Move Statements into Function

```
result.push(`<p>title: ${person.photo.title}</p>`);
result.concat(photoData(person.photo));

function photoData(aPhoto) {
  return [
    `<p>location: ${aPhoto.location}</p>`,
    `<p>date: ${aPhoto.date.toDateString()}</p>`,
  ];
}
```

```
result.concat(photoData(person.photo));

function photoData(aPhoto) {
  return [
    `<p>title: ${aPhoto.title}</p>`,
    `<p>location: ${aPhoto.location}</p>`,
    `<p>date: ${aPhoto.date.toDateString()}</p>`,
  ];
}
```

## Motivation

Absorb statements that repeat at every call site into the called function itself, so future changes to that shared behavior happen in one place.

## Mechanics

- If the repetitive code isn't adjacent to the call of the target function, use Slide Statements to get it adjacent.

- If the target function is only called by the source function, just cut the code from the source, paste it into the target, test, and ignore the rest of these mechanics.

  This is the simple case. The remaining steps handle the multi-caller situation safely.

- If you have more callers, use Extract Function on one of the call sites to extract both the call to the target function and the statements you wish to move into it. Give it a name that's transient, but easy to grep.

  A throwaway name like `zznew` makes it obvious the function is temporary and trivial to search for.

- Convert every other call to use the new function. Test after each conversion.

  One caller at a time keeps each step small and reversible.

- When all the original calls use the new function, use Inline Function to inline the original function completely into the new function, removing the original function.

  At this point the new function contains both the moved statements and the original body.

- Use Rename Function to change the name of the new function to the same name as the original function -- or to a better name, if there is one.

## Example

Starting code that emits HTML for data about a photo:

```js
function renderPerson(outStream, person) {
  const result = [];
  result.push(`<p>${person.name}</p>`);
  result.push(renderPhoto(person.photo));
  result.push(`<p>title: ${person.photo.title}</p>`);
  result.push(emitPhotoData(person.photo));
  return result.join("\n");
}


function photoDiv(p) {
  return [
    "<div>",
    `<p>title: ${p.title}</p>`,
    emitPhotoData(p),
    "</div>",
  ].join("\n");
}

function emitPhotoData(aPhoto) {
  const result = [];
  result.push(`<p>location: ${aPhoto.location}</p>`);
  result.push(`<p>date: ${aPhoto.date.toDateString()}</p>`);
  return result.join("\n");
}
```

There are two calls to `emitPhotoData`, each preceded by a line that prints the title. That duplicated title line is semantically part of what `emitPhotoData` does, so it belongs inside it. With only one caller you could just cut and paste, but with multiple callers a safer sequence avoids mistakes.

First, use Extract Function on one of the callers. Extract the title line together with the `emitPhotoData` call into a temporary function named `zznew` -- easy to grep for later.

function photoDiv -- after extracting `zznew`:

```js
function photoDiv(p) {
  return [
    "<div>",
    zznew(p),
    "</div>",
  ].join("\n");
}


function zznew(p) {
  return [
    `<p>title: ${p.title}</p>`,
    emitPhotoData(p),
  ].join("\n");
}
```

Now look at the other callers of `emitPhotoData` and, one by one, replace the call and the preceding title statement with a call to `zznew`. Each replacement is tested before moving to the next.

function renderPerson -- after switching to `zznew`:

```js
function renderPerson(outStream, person) {
  const result = [];
  result.push(`<p>${person.name}</p>`);
  result.push(renderPhoto(person.photo));
  result.push(zznew(person.photo));
  return result.join("\n");
}
```

Now that all the original callers use `zznew`, use Inline Function to inline `emitPhotoData` into `zznew`, eliminating the original function entirely. The body of `zznew` now contains everything.

function zznew -- after inlining `emitPhotoData`:

```js
function zznew(p) {
  return [
    `<p>title: ${p.title}</p>`,
    `<p>location: ${p.location}</p>`,
    `<p>date: ${p.date.toDateString()}</p>`,
  ].join("\n");
}
```

Finish with Rename Function to restore the original name, and tidy up parameter names to match convention.

Final result:

```js
function renderPerson(outStream, person) {
  const result = [];
  result.push(`<p>${person.name}</p>`);
  result.push(renderPhoto(person.photo));
  result.push(emitPhotoData(person.photo));
  return result.join("\n");
}


function photoDiv(aPhoto) {
  return [
    "<div>",
    emitPhotoData(aPhoto),
    "</div>",
  ].join("\n");
}


function emitPhotoData(aPhoto) {
  return [
    `<p>title: ${aPhoto.title}</p>`,
    `<p>location: ${aPhoto.location}</p>`,
    `<p>date: ${aPhoto.date.toDateString()}</p>`,
  ].join("\n");
}
```

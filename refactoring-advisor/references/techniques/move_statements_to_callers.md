# Move Statements to Callers

```
emitPhotoData(outStream, person.photo);

function emitPhotoData(outStream, photo) {
  outStream.write(`<p>title: ${photo.title}</p>\n`);
  outStream.write(`<p>location: ${photo.location}</p>\n`);
}
```

{...}

```
emitPhotoData(outStream, person.photo);
outStream.write(`<p>location: ${person.photo.location}</p>\n`);

function emitPhotoData(outStream, photo) {
  outStream.write(`<p>title: ${photo.title}</p>\n`);
}
```

## Motivation

Move statements out of a function and into its callers when behavior that was once shared now needs to vary at individual call sites. For large-scale boundary changes, prefer Inline Function first and then re-extract rather than moving statements piecemeal.

## Mechanics

- In simple circumstances -- only one or two callers and a straightforward function -- just cut the statement from the called function and paste it into each caller. Test and you're done.

- Otherwise, apply Extract Function to all the statements that you _don't_ wish to move; give the extracted function a temporary but easily searchable name (e.g., `zztmp`).

  If the function is a method overridden by subclasses, do the extraction on all of them so that the remaining method is identical in every class. Then remove the subclass methods.

- Use Inline Function on the original function. This pushes the statement-to-move into every caller alongside a call to the extracted temp function.

- Apply Change Function Declaration on the extracted function to rename it to the original name -- or to a better name, if you can think of one.

## Example

Here is a simple case: a function with two callers.

```javascript
function renderPerson(outStream, person) {
  outStream.write(`<p>${person.name}</p>\n`);
  renderPhoto(outStream, person.photo);
  emitPhotoData(outStream, person.photo);
}

function listRecentPhotos(outStream, photos) {
  photos
    .filter(p => p.date > recentDateCutoff())
    .forEach(p => {
      outStream.write("<div>\n");
      emitPhotoData(outStream, p);
      outStream.write("</div>\n");
    });
}

function emitPhotoData(outStream, photo) {
  outStream.write(`<p>title: ${photo.title}</p>\n`);
  outStream.write(`<p>date: ${photo.date.toDateString()}</p>\n`);
  outStream.write(`<p>location: ${photo.location}</p>\n`);
}
```

We need to modify the software so that `listRecentPhotos` renders the location information differently while `renderPerson` stays the same. To make this change easier, we will use Move Statements to Callers on the final line of `emitPhotoData`.

For a case this simple we could just cut the last line and paste it into the callers. But to demonstrate the safer procedure for trickier situations, we will go through the full mechanics.

**Step 1 -- Extract Function on the statements that will stay.** We apply Extract Function to everything in `emitPhotoData` _except_ the line we want to move. The temporary name `zztmp` is easy to grep for later.

```javascript
function emitPhotoData(outStream, photo) {
  zztmp(outStream, photo);
  outStream.write(`<p>location: ${photo.location}</p>\n`);
}

function zztmp(outStream, photo) {
  outStream.write(`<p>title: ${photo.title}</p>\n`);
  outStream.write(`<p>date: ${photo.date.toDateString()}</p>\n`);
}
```

We test here to make sure the extraction preserves behavior across the new function-call boundary.

**Step 2 -- Inline Function on `emitPhotoData`, one caller at a time.** We inline into `renderPerson` first. This replaces the call to `emitPhotoData` with its body -- a call to `zztmp` plus the location line -- directly in the caller.

function renderPerson -- after inlining:

```javascript
function renderPerson(outStream, person) {
  outStream.write(`<p>${person.name}</p>\n`);
  renderPhoto(outStream, person.photo);
  zztmp(outStream, person.photo);
  outStream.write(`<p>location: ${person.photo.location}</p>\n`);
}

function listRecentPhotos(outStream, photos) {
  photos
    .filter(p => p.date > recentDateCutoff())
    .forEach(p => {
      outStream.write("<div>\n");
      emitPhotoData(outStream, p);
      outStream.write("</div>\n");
    });
}

function emitPhotoData(outStream, photo) {
  zztmp(outStream, photo);
  outStream.write(`<p>location: ${photo.location}</p>\n`);
}

function zztmp(outStream, photo) {
  outStream.write(`<p>title: ${photo.title}</p>\n`);
  outStream.write(`<p>date: ${photo.date.toDateString()}</p>\n`);
}
```

Test again to make sure this caller works properly, then do the same for `listRecentPhotos`.

function listRecentPhotos -- after inlining:

```javascript
function renderPerson(outStream, person) {
  outStream.write(`<p>${person.name}</p>\n`);
  renderPhoto(outStream, person.photo);
  zztmp(outStream, person.photo);
  outStream.write(`<p>location: ${person.photo.location}</p>\n`);
}

function listRecentPhotos(outStream, photos) {
  photos
    .filter(p => p.date > recentDateCutoff())
    .forEach(p => {
      outStream.write("<div>\n");
      zztmp(outStream, p);
      outStream.write(`<p>location: ${p.location}</p>\n`);
      outStream.write("</div>\n");
    });
}

function zztmp(outStream, photo) {
  outStream.write(`<p>title: ${photo.title}</p>\n`);
  outStream.write(`<p>date: ${photo.date.toDateString()}</p>\n`);
}
```

All callers have been inlined, so `emitPhotoData` is now dead code and can be deleted. The location-writing statement is now in each caller where it can vary independently.

**Step 3 -- Rename `zztmp` back to the original name.** We apply Change Function Declaration to give the extracted function a proper name again.

```javascript
function renderPerson(outStream, person) {
  outStream.write(`<p>${person.name}</p>\n`);
  renderPhoto(outStream, person.photo);
  emitPhotoData(outStream, person.photo);
  outStream.write(`<p>location: ${person.photo.location}</p>\n`);
}

function listRecentPhotos(outStream, photos) {
  photos
    .filter(p => p.date > recentDateCutoff())
    .forEach(p => {
      outStream.write("<div>\n");
      emitPhotoData(outStream, p);
      outStream.write(`<p>location: ${p.location}</p>\n`);
      outStream.write("</div>\n");
    });
}

function emitPhotoData(outStream, photo) {
  outStream.write(`<p>title: ${photo.title}</p>\n`);
  outStream.write(`<p>date: ${photo.date.toDateString()}</p>\n`);
}
```

The location line is now in each caller. We can freely change how `listRecentPhotos` handles location without affecting `renderPerson`, which was the goal.

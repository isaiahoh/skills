# Encapsulate Collection

```
class Person {
  get courses() {return this._courses;}
  set courses(aList) {this._courses = aList;}
```

```
class Person {
  get courses() {return this._courses.slice();}
  addCourse(aCourse)    { ... }
  removeCourse(aCourse) { ... }
```

## Motivation

Replaces direct access to a collection field with add/remove methods on the owning class and a getter that returns a copy or read-only proxy, so the class can control mutations to its collection.

## Mechanics

- Apply Encapsulate Variable if the reference to the collection isn't already encapsulated.

- Add functions to add and remove elements from the collection.

  If there is a setter for the collection, use Remove Setting Method if possible. If not, make it take a copy of the provided collection.

- Run static checks.
- Find all references to the collection. If anyone calls modifiers on the collection, change them to use the new add/remove functions. Test after each change.

- Modify the getter for the collection to return a protected view on it, using a read-only proxy or a copy.

- Test.

## Example

I start with a person class that has a field for a list of courses.

class Person...

```js
constructor (name) {
  this._name = name;
  this._courses = [];
}
get name() {return this._name;}
get courses() {return this._courses;}
set courses(aList) {this._courses = aList;}
```

class Course...

```js
constructor(name, isAdvanced) {
  this._name = name;
  this._isAdvanced = isAdvanced;
}
get name()       {return this._name;}
get isAdvanced() {return this._isAdvanced;}
```

Clients use the course collection to gather information on courses.

```js
numAdvancedCourses = aPerson.courses
  .filter(c => c.isAdvanced)
  .length
;
```

Anyone updating courses as a single value has proper control through the setter:

client code...

```js
const basicCourseNames = readBasicCourseNames(filename);
aPerson.courses = basicCourseNames.map(name => new Course(name, false));
```

But clients might update the course list directly, which violates encapsulation because the person class has no ability to intervene:

client code...

```js
for(const name of readBasicCourseNames(filename)) {
  aPerson.courses.push(new Course(name, false));
}
```

I begin by adding methods to the person class that allow a client to add and remove individual courses.

class Person...

```js
addCourse(aCourse) {
  this._courses.push(aCourse);
}
removeCourse(aCourse, fnIfAbsent = () => {throw new RangeError();}) {
  const index = this._courses.indexOf(aCourse);
  if (index === -1) fnIfAbsent();
  else this._courses.splice(index, 1);
}
```

With a removal, I default to raising an error if the element isn't present, but give callers an opportunity to do something else if they wish.

I then change any code that calls modifiers directly on the collection to use the new methods.

client code...

```js
for(const name of readBasicCourseNames(filename)) {
  aPerson.addCourse(new Course(name, false));
}
```

With individual add and remove methods, there is usually no need for `setCourses`, so use Remove Setting Method on it. If the API still needs a setter, ensure it stores a copy of the collection.

class Person...

```js
set courses(aList) {this._courses = aList.slice();}
```

Finally, ensure nobody modifies the list without using the modifier methods by returning a copy from the getter.

class Person...

```js
get courses() {return this._courses.slice();}
```

Be moderately paranoid about collections--copy them unnecessarily rather than debug errors due to unexpected modifications. Note that sorting an array in JavaScript modifies the original.

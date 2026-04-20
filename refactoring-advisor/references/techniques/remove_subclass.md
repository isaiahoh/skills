# Remove Subclass

```
class Person {
  get genderCode() {return "X";}
}
class Male extends Person {
  get genderCode() {return "M";}
}
class Female extends Person {
  get genderCode() {return "F";}
}
```

```
class Person {
  get genderCode() {return this._genderCode;}
}
```

## Motivation

Subclasses lose their value as the variations they support are moved elsewhere or removed. A subclass that does too little incurs an understanding cost that is no longer worthwhile. When that happens, replace it with a field on its superclass.

## Mechanics

- Use Replace Constructor with Factory Function on the subclass constructor.

  If the clients of the constructors use a data field to decide which subclass to create, put that decision logic into a superclass factory method.

- If any code tests against the subclass's types, use Extract Function on the type test and Move Function to move it to the superclass. Test after each change.

- Create a field to represent the subclass type.

- Change the methods that refer to the subclass to use the new type field.

- Delete the subclass.

- Test.
- Often, this refactoring is used on a group of subclasses at once -- in which case carry out the steps to encapsulate them (add factory function, move type tests) first, then individually fold them into the superclass.

## Example

I'll start with this stump of subclasses:

class Person...

```
constructor(name) {
  this._name = name;
}
get name()    {return this._name;}
get genderCode() {return "X";}
// snip
```

```
class Male extends Person {
  get genderCode() {return "M";}
}

class Female extends Person {
  get genderCode() {return "F";}
}
```

If that's all a subclass does, it's not really worth having. But before I remove these subclasses, it's worth checking for any subclass-dependent behavior in the clients that should be moved in there.

client...

```
const numberOfMales = people.filter(p => p instanceof Male).length;
```

To minimize the impact on client code, I first encapsulate the current representation. I use Replace Constructor with Factory Function. In this case, objects like this are often loaded from a source that uses the gender codes directly.

```
function loadFromInput(data) {
  const result = [];
  data.forEach(aRecord => {
    let p;
    switch (aRecord.gender) {
      case 'M': p = new Male(aRecord.name); break;
      case 'F': p = new Female(aRecord.name); break;
      default: p = new Person(aRecord.name);
    }
    result.push(p);
  });
  return result;
}
```

I use Extract Function on the selection logic for which class to create, making that the factory function.

```
function createPerson(aRecord) {
  let p;
  switch (aRecord.gender) {
    case 'M': p = new Male(aRecord.name); break;
    case 'F': p = new Female(aRecord.name); break;
    default: p = new Person(aRecord.name);
  }
  return p;
}

function loadFromInput(data) {
  const result = [];
  data.forEach(aRecord => {
    result.push(createPerson(aRecord));
  });
  return result;
}
```

I'll clean up those two functions. I use Inline Variable on `createPerson`:

```
function createPerson(aRecord) {
  switch (aRecord.gender) {
    case 'M': return new Male  (aRecord.name);
    case 'F': return new Female(aRecord.name);
    default:  return new Person(aRecord.name);
  }
}
```

and Replace Loop with Pipeline on `loadFromInput`:

```
function loadFromInput(data) {
  return data.map(aRecord => createPerson(aRecord));
}
```

The factory encapsulates the creation of the subclasses, but there is also the use of `instanceof`. I use Extract Function on the type check.

client...

```
const numberOfMales = people.filter(p => isMale(p)).length;
```

```
function isMale(aPerson) {return aPerson instanceof Male;}
```

Then I use Move Function to move it into `Person`.

class Person...

```
get isMale() {return this instanceof Male;}
```

client...

```
const numberOfMales = people.filter(p => p.isMale).length;
```

All knowledge of the subclasses is now safely encased within the superclass and the factory function.

I now add a field to represent the difference between the subclasses; since I'm using a code loaded from elsewhere, I just use that.

class Person...

```
constructor(name, genderCode) {
  this._name = name;
  this._genderCode = genderCode || "X";
}

get genderCode() {return this._genderCode;}
```

I take the male case and fold its logic into the superclass. This involves modifying the factory to return a `Person` and modifying any `instanceof` tests to use the gender code field.

```
function createPerson(aRecord) {
  switch (aRecord.gender) {
    case 'M': return new Person(aRecord.name, "M");
    case 'F': return new Female(aRecord.name);
    default:  return new Person(aRecord.name);
  }
}
```

class Person...

```
get isMale() {return "M" === this._genderCode;}
```

I test, remove the male subclass, test again, and repeat for the female subclass.

```
function createPerson(aRecord) {
  switch (aRecord.gender) {
    case 'M': return new Person(aRecord.name, "M");
    case 'F': return new Person(aRecord.name, "F");
    default:  return new Person(aRecord.name);
  }
}
```

The lack of symmetry with the gender code is annoying -- a future reader will wonder about it. So I prefer to make it symmetrical.

```
function createPerson(aRecord) {
  switch (aRecord.gender) {
    case 'M': return new Person(aRecord.name, "M");
    case 'F': return new Person(aRecord.name, "F");
    default:  return new Person(aRecord.name, "X");
  }
}
```

class Person...

```
constructor(name, genderCode) {
  this._name = name;
  this._genderCode = genderCode || "X";
}
```

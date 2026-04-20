# Rename Field

```
class Organization {
  get name() {...}
}
```

```
class Organization {
  get title() {...}
}
```

## Motivation

Rename a field (or its getter/setter) so its name clearly communicates its purpose within the data structure.

## Mechanics

- If the record has limited scope, rename all accesses to the field and test; no need to do the rest of the mechanics.

- If the record isn't already encapsulated, apply Encapsulate Record.

- Rename the private field inside the object, adjust internal methods to fit.

- Test.

- If the constructor uses the name, apply Change Function Declaration to rename it.

- Apply Rename Function to the accessors.

## Example: Renaming a Field

I'll start with a constant.

```
const organization = {name: "Acme Gooseberries", country: "GB"};
```

I want to change "name" to "title". The object is widely used in the code base, and there are updates to the title in the code. So my first move is to apply Encapsulate Record.

class Organization...

```
class Organization {
  constructor(data) {
    this._name = data.name;
    this._country = data.country;
  }
  get name()    {return this._name;}
  set name(aString) {this._name = aString;}
  get country()    {return this._country;}
  set country(aCountryCode) {this._country = aCountryCode;}
}
```

```
const organization = new Organization({name: "Acme Gooseberries", country: "GB"});
```

Now that I've encapsulated the record structure into the class, there are four places I need to look at for renaming: the getting function, the setting function, the constructor, and the internal data structure. I can now change these independently instead of all at once, taking smaller steps.

Since I've copied the input data structure into the internal data structure, I need to separate them so I can work on them independently. I can do this by defining a separate field and adjusting the constructor and accessors to use it.

class Organization...

```
class Organization {
  constructor(data) {
    this._title = data.name;
    this._country = data.country;
  }
  get name()    {return this._title;}
  set name(aString) {this._title = aString;}
  get country()    {return this._country;}
  set country(aCountryCode) {this._country = aCountryCode;}
}
```

Next, I add support for using "title" in the constructor.

class Organization...

```
class Organization {
  constructor(data) {
    this._title = (data.title !== undefined) ? data.title : data.name;
    this._country = data.country;
  }
  get name()    {return this._title;}
  set name(aString) {this._title = aString;}
  get country()    {return this._country;}
  set country(aCountryCode) {this._country = aCountryCode;}
}
```

Now, callers of my constructor can use either name or title (with title taking precedence). I can now go through all constructor callers and change them one-by-one to use the new name.

```
const organization = new Organization({title: "Acme Gooseberries", country: "GB"});
```

Once I've done all of them, I can remove the support for the name.

class Organization...

```
class Organization {
  constructor(data) {
    this._title = data.title;
    this._country = data.country;
  }
  get name()    {return this._title;}
  set name(aString) {this._title = aString;}
  get country()    {return this._country;}
  set country(aCountryCode) {this._country = aCountryCode;}
}
```

Now that the constructor and data use the new name, I can change the accessors, which is as simple as applying Rename Function to each one.

class Organization...

```
class Organization {
  constructor(data) {
    this._title = data.title;
    this._country = data.country;
  }
  get title()    {return this._title;}
  set title(aString) {this._title = aString;}
  get country()    {return this._country;}
  set country(aCountryCode) {this._country = aCountryCode;}
}
```

I've shown this process in its most heavyweight form needed for a widely used data structure. If it's being used only locally, as in a single function, I can probably just rename the various properties in one go without doing encapsulation. If my tests break, that's a sign I need to use the more gradual procedure.

Some languages allow me to make a data structure immutable. In this case, rather than encapsulating it, I can copy the value to the new name, gradually change the users, then remove the old name. Duplicating data is a recipe for disaster with mutable data structures; removing such disasters is why immutable data is so popular.

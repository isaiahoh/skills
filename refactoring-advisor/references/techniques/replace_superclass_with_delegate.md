# Replace Superclass with Delegate

```
class List {...}
class Stack extends List {...}
```

```
class Stack {
  constructor() {
    this._storage = new List();
  }
}
class List {...}
```

## Motivation

Replaces inheritance with delegation when the superclass interface doesn't fully apply to the subclass, storing the former superclass as a field and forwarding only the operations that are valid.

## Mechanics

- Create a field in the subclass that refers to the superclass object. Initialize this delegate reference to a new instance.

- For each element of the superclass, create a forwarding function in the subclass that forwards to the delegate reference. Test after forwarding each consistent group.

  Most of the time you can test after each function that's forwarded, but, for example, get/set pairs can only be tested once both have been moved.

- When all superclass elements have been overridden with forwarders, remove the inheritance link.

## Example

A library keeps details of scrolls in a catalog. Each scroll has an ID number and records its title and list of tags.

class CatalogItem...

```
constructor(id, title, tags) {
  this._id = id;
  this._title = title;
  this._tags = tags;
}

get id() {return this._id;}
get title() {return this._title;}
hasTag(arg) {return this._tags.includes(arg);}
```

Scrolls need regular cleaning. The code extends the catalog item with cleaning data.

class Scroll extends CatalogItem...

```
constructor(id,  title, tags, dateLastCleaned) {
  super(id, title, tags);
  this._lastCleaned = dateLastCleaned;
}

needsCleaning(targetDate) {
  const threshold =  this.hasTag("revered") ? 700 : 1500;
  return this.daysSinceLastCleaning(targetDate) > threshold ;
}
daysSinceLastCleaning(targetDate) {
  return this._lastCleaned.until(targetDate, ChronoUnit.DAYS);
}
```

This is a common modeling error. There is a difference between the physical scroll and the catalog item -- the scroll for the greyscale disease treatment may have several copies but be just one catalog item.

I begin by creating a property in `Scroll` that refers to the catalog item, initializing it with a new instance.

class Scroll extends CatalogItem...

```
constructor(id,  title, tags, dateLastCleaned) {
  super(id, title, tags);
  this._catalogItem = new CatalogItem(id, title, tags);
  this._lastCleaned = dateLastCleaned;
}
```

I create forwarding methods for each element of the superclass that I use on the subclass.

class Scroll...

```
get id() {return this._catalogItem.id;}
get title() {return this._catalogItem.title;}
hasTag(aString) {return this._catalogItem.hasTag(aString);}
```

I remove the inheritance link to the catalog item.

```
class Scroll extends CatalogItem{
  constructor(id,  title, tags, dateLastCleaned) {
    super(id, title, tags);
    this._catalogItem = new CatalogItem(id, title, tags);
    this._lastCleaned = dateLastCleaned;
  }
```

Breaking the inheritance link finishes the basic refactoring, but there is more to do in this case.

The refactoring shifts the catalog item to a component of scroll; each scroll contains a unique instance. A better model is to link the greyscale catalog item to the six scrolls that are copies -- essentially Change Value to Reference.

There's a problem to fix first. In the original inheritance structure, the scroll used the catalog item's ID field for its own ID. If I treat the catalog item as a reference, it needs that ID for itself. So I create a separate ID field on scroll.

class Scroll...

```
constructor(id,  title, tags, dateLastCleaned) {
  this._id = id;
  this._catalogItem = new CatalogItem(null, title, tags);
  this._lastCleaned = dateLastCleaned;
}

get id() {return this._id;}
```

Currently the scrolls are loaded as part of a load routine.

load routine...

```
const scrolls = aDocument
      .map(record => new Scroll(record.id,
                                record.catalogData.title,
                                record.catalogData.tags,
                                LocalDate.parse(record.lastCleaned)));
```

The first step in Change Value to Reference is finding or creating a repository. I find one that supplies catalog items indexed by an ID. The catalog item's ID is present in the input data and was being ignored when using inheritance. I use Change Function Declaration to add both the catalog and the catalog item's ID to the constructor parameters.

load routine...

```
const scrolls = aDocument
      .map(record => new Scroll(record.id,
                                record.catalogData.title,
                                record.catalogData.tags,
                                LocalDate.parse(record.lastCleaned),
                                record.catalogData.id,
                                catalog));
```

class Scroll...

```
constructor(id,  title, tags, dateLastCleaned, catalogID, catalog) {
  this._id = id;
  this._catalogItem = new CatalogItem(null, title, tags);
  this._lastCleaned = dateLastCleaned;
}
```

I now modify the constructor to use the catalog ID to look up the catalog item instead of creating a new one.

class Scroll...

```
constructor(id,  title, tags, dateLastCleaned, catalogID, catalog) {
  this._id = id;
  this._catalogItem = catalog.get(catalogID);
  this._lastCleaned = dateLastCleaned;
}
```

I no longer need the title and tags passed into the constructor, so I use Change Function Declaration to remove them.

load routine...

```
const scrolls = aDocument
      .map(record => new Scroll(record.id,
                                LocalDate.parse(record.lastCleaned),
                                record.catalogData.id,
                                catalog));
```

class Scroll...

```
constructor(id, dateLastCleaned, catalogID, catalog) {
  this._id = id;
  this._catalogItem = catalog.get(catalogID);
  this._lastCleaned = dateLastCleaned;
}
```

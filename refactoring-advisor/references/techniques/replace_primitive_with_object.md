# Replace Primitive with Object

```
orders.filter(o => "high" === o.priority
                || "rush" === o.priority);
```

```
orders.filter(o => o.priority.higherThan(new Priority("normal")))
```

## Motivation

Wraps a primitive value (string, number) in a small class, giving it a home for domain-specific behavior like validation, formatting, and comparison.

## Mechanics

- Apply Encapsulate Variable if it isn't already.

- Create a simple value class for the data value. It should take the existing value in its constructor and provide a getter for that value.

- Run static checks.
- Change the setter to create a new instance of the value class and store that in the field, changing the type of the field if present.

- Change the getter to return the result of invoking the getter of the new class.

- Test.
- Consider using Rename Function on the original accessors to better reflect what they do.

- Consider clarifying the role of the new object as a value or reference object by applying Change Reference to Value or Change Value to Reference.

## Example

I begin with an order class that reads priority as a simple string.

class Order...

```js
constructor(data) {
  this.priority = data.priority;
  // more initialization
```

Some client code uses it like this:

client...

```js
highPriorityCount = orders.filter(o => "high" === o.priority
                                    || "rush" === o.priority)
                          .length;
```

First, I use Encapsulate Variable on the priority field.

class Order...

```js
get priority()        {return this._priority;}
set priority(aString) {this._priority = aString;}
```

The constructor line that initializes the priority will now use the setter. This self-encapsulates the field so I can preserve its current use while I manipulate the data itself.

I create a simple value class for the priority. I prefer using a conversion function (`toString`) rather than a getter (`value`)--for clients, asking for the string representation should feel more like a conversion than getting a property.

```js
class Priority {
  constructor(value) {this._value = value;}
  toString() {return this._value;}
}
```

I then modify the accessors to use the new class.

class Order...

```js
get priority()        {return this._priority.toString();}
set priority(aString) {this._priority = new Priority(aString);}
```

The current getter is now misleading--it returns a string, not a priority. I use Rename Function.

class Order...

```js
get priorityString()  {return this._priority.toString();}
set priority(aString) {this._priority = new Priority(aString);}
```

client...

```js
highPriorityCount = orders.filter(o => "high" === o.priorityString
                                    || "rush" === o.priorityString)
                          .length;
```

I retain the name of the setter since the argument name communicates what it expects.

Now I provide a getter on order that exposes the new priority object directly.

class Order...

```js
get priority()        {return this._priority;}
get priorityString()  {return this._priority.toString();}
set priority(aString) {this._priority = new Priority(aString);}
```

client...

```js
highPriorityCount = orders.filter(o => "high" === o.priority.toString()
                                    || "rush" === o.priority.toString())
                          .length;
```

To allow clients to set priority with a priority instance, I adjust the constructor:

class Priority...

```js
constructor(value) {
  if (value instanceof Priority) return value;
  this._value = value;
}
```

Now the priority class becomes a place for new behavior. Here's validation and comparison logic:

class Priority...

```js
constructor(value) {
  if (value instanceof Priority) return value;
  if (Priority.legalValues().includes(value))
    this._value = value;
  else
    throw new Error(`<${value}> is invalid for Priority`);
}
toString() {return this._value;}
get _index() {return Priority.legalValues().findIndex(s => s === this._value);}
static legalValues() {return ['low', 'normal', 'high', 'rush'];}

equals(other) {return this._index === other._index;}
higherThan(other) {return this._index > other._index;}
lowerThan(other) {return this._index < other._index;}
```

I decide that priority should be a value object, so I provide an `equals` method and ensure it is immutable.

Now client code becomes more meaningful:

client...

```js
highPriorityCount = orders.filter(o => o.priority.higherThan(new Priority("normal")))
                          .length;
```

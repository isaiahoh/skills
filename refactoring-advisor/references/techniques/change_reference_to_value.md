# Change Reference to Value

```
class Product {
  applyDiscount(arg) {this._price.amount -= arg;}
```

```
class Product {
  applyDiscount(arg) {
    this._price = new Money(this._price.amount - arg, this._price.currency);
  }
```

## Motivation

Replace a mutable inner object (reference) with an immutable Value Object so it can be safely shared and reasoned about. Do not apply this if multiple owners need to see mutations to a shared object -- keep it as a reference in that case.

## Mechanics

- Check that the candidate class is immutable or can become immutable.

- For each setter, apply Remove Setting Method.

- Provide a value-based equality method that uses the fields of the value object.

  Most language environments provide an overridable equality function for this purpose. Usually you must override a hashcode generator method as well.

## Example

Imagine we have a person object that holds onto a crude telephone number.

class Person...

```
constructor() {
  this._telephoneNumber = new TelephoneNumber();
}

get officeAreaCode()    {return this._telephoneNumber.areaCode;}
set officeAreaCode(arg) {this._telephoneNumber.areaCode = arg;}
get officeNumber()    {return this._telephoneNumber.number;}
set officeNumber(arg) {this._telephoneNumber.number = arg;}
```

class TelephoneNumber...

```
get areaCode()    {return this._areaCode;}
set areaCode(arg) {this._areaCode = arg;}

get number()    {return this._number;}
set number(arg) {this._number = arg;}
```

This situation is the result of an Extract Class where the old parent still holds update methods for the new object. This is a good time to apply Change Reference to Value since there is only one reference to the new class.

The first thing I need to do is to make the telephone number immutable. I do this by applying Remove Setting Method to the fields. The first step of Remove Setting Method is to use Change Function Declaration to add the two fields to the constructor and enhance the constructor to call the setters.

class TelephoneNumber...

```
constructor(areaCode, number) {
  this._areaCode = areaCode;
  this._number = number;
}
```

Now I look at the callers of the setters. For each one, I need to change it to a reassignment. I start with the area code.

class Person...

```
get officeAreaCode()    {return this._telephoneNumber.areaCode;}
set officeAreaCode(arg) {
  this._telephoneNumber = new TelephoneNumber(arg, this.officeNumber);
}
get officeNumber()    {return this._telephoneNumber.number;}
set officeNumber(arg) {this._telephoneNumber.number = arg;}
```

I then repeat that step with the remaining field.

class Person...

```
get officeAreaCode()    {return this._telephoneNumber.areaCode;}
set officeAreaCode(arg) {
  this._telephoneNumber = new TelephoneNumber(arg, this.officeNumber);
}
get officeNumber()    {return this._telephoneNumber.number;}
set officeNumber(arg) {
  this._telephoneNumber = new TelephoneNumber(this.officeAreaCode, arg);
}
```

Now the telephone number is immutable, it is ready to become a true value. The citizenship test for a value object is that it uses value-based equality. This is an area where JavaScript falls down, as there is nothing in the language and core libraries that understands replacing a reference-based equality with a value-based one. The best I can do is to create my own `equals` method.

class TelephoneNumber...

```
equals(other) {
  if (!(other instanceof TelephoneNumber)) return false;
  return this.areaCode === other.areaCode &&
    this.number === other.number;
}
```

It's also important to test it with something like

```
it('telephone equals', function() {
  assert(        new TelephoneNumber("312", "555-0142")
         .equals(new TelephoneNumber("312", "555-0142")));
});
```

The vital thing I do in the test is create two independent objects and test that they match as equal.

In most object-oriented languages, there is a built-in equality test that is supposed to be overridden for value-based equality. In Ruby, I can override the `==` operator; in Java, I override the `Object.equals()` method. And whenever I override an equality method, I usually need to override a hashcode generating method too (e.g., `Object.hashCode()` in Java) to ensure collections that use hashing work properly with my new value.

If the telephone number is used by more than one client, the procedure is still the same. As I apply Remove Setting Method, I'll be modifying several clients instead of just one. Tests for non-equal telephone numbers, as well as comparisons to non-telephone-numbers and null values, are also worthwhile.

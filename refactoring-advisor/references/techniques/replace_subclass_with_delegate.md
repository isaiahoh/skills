# Replace Subclass with Delegate

```
class Order {
  get daysToShip() {
    return this._warehouse.daysToShip;
  }
}

class PriorityOrder extends Order {
  get daysToShip() {
    return this._priorityPlan.daysToShip;
  }
}
```

```
class Order {
  get daysToShip() {
    return (this._priorityDelegate)
      ? this._priorityDelegate.daysToShip
      : this._warehouse.daysToShip;
  }
}

class PriorityOrderDelegate {
  get daysToShip() {
    return this._priorityPlan.daysToShip
  }
}
```

## Motivation

Replaces subclass-based variation with delegation, allowing multiple axes of variation and looser coupling. When replacing a full hierarchy, the delegates often benefit from their own inheritance via Extract Superclass.

## Mechanics

- If there are many callers for the constructors, apply Replace Constructor with Factory Function.

- Create an empty class for the delegate. Its constructor should take any subclass-specific data as well as, usually, a back-reference to the superclass.

- Add a field to the superclass to hold the delegate.

- Modify the creation of the subclass so that it initializes the delegate field with an instance of the delegate.

  This can be done in the factory function, or in the constructor if the constructor can reliably tell whether to create the correct delegate.

- Choose a subclass method to move to the delegate class.

- Use Move Function to move it to the delegate class. Don't remove the source's delegating code.

  If the method needs elements that should move to the delegate, move them. If it needs elements that should stay in the superclass, add a field to the delegate that refers to the superclass.

- If the source method has callers outside the class, move the source's delegating code from the subclass to the superclass, guarding it with a check for the presence of the delegate. If not, apply Remove Dead Code.

  If there's more than one subclass, and you start duplicating code within them, use Extract Superclass. In this case, any delegating methods on the source superclass no longer need a guard if the default behavior is moved to the delegate superclass.

- Test.
- Repeat until all the methods of the subclass are moved.

- Find all callers of the subclass's constructor and change them to use the superclass constructor.

- Test.
- Use Remove Dead Code on the subclass.

## Example

I have a class that makes a booking for a show.

class Booking...

```
constructor(show, date) {
  this._show = show;
  this._date = date;
}
```

There is a subclass for premium booking that takes into account various extras.

class PremiumBooking extends Booking...

```
constructor(show, date, extras) {
  super(show, date);
  this._extras = extras;
}
```

The premium booking overrides and extends superclass behavior in several ways.

First, a simple override. Regular bookings offer a talkback after the show, but only on nonpeak days.

class Booking...

```
get hasTalkback() {
  return this._show.hasOwnProperty('talkback') && !this.isPeakDay;
}
```

Premium bookings override this to offer talkbacks on all days.

class PremiumBooking...

```
get hasTalkback() {
  return this._show.hasOwnProperty('talkback');
}
```

Determining the price is a similar override, with a twist that the premium method calls the superclass method.

class Booking...

```
get basePrice() {
  let result = this._show.price;
  if (this.isPeakDay) result += Math.round(result * 0.15);
  return result;
}
```

class PremiumBooking...

```
get basePrice() {
  return Math.round(super.basePrice + this._extras.premiumFee);
}
```

The last example is where the premium booking offers a behavior that isn't present on the superclass.

class PremiumBooking...

```
get hasDinner() {
  return this._extras.hasOwnProperty('dinner') && !this.isPeakDay;
}
```

I have clients call the constructors of the two classes:

booking client

```
aBooking = new Booking(show,date);
```

premium client

```
aBooking = new PremiumBooking(show, date, extras);
```

I encapsulate the constructor calls with Replace Constructor with Factory Function.

top level...

```
function createBooking(show, date) {
  return new Booking(show, date);
}
function createPremiumBooking(show, date, extras) {
  return new PremiumBooking (show, date, extras);
}
```

booking client

```
aBooking = createBooking(show, date);
```

premium client

```
aBooking = createPremiumBooking(show, date, extras);
```

I now make the new delegate class. Its constructor parameters are those parameters that are only used in the subclass, together with a back-reference to the booking object (needed because several subclass methods require access to data stored in the superclass).

class PremiumBookingDelegate...

```
constructor(hostBooking, extras) {
  this._host = hostBooking;
  this._extras = extras;
}
```

I connect the new delegate to the booking object by modifying the factory function for premium bookings.

top level...

```
function createPremiumBooking(show, date, extras) {
  const result = new PremiumBooking (show, date, extras);
  result._bePremium(extras);
  return result;
}
```

class Booking...

```
_bePremium(extras) {
  this._premiumDelegate = new PremiumBookingDelegate(this, extras);
}
```

I use a leading underscore on `_bePremium` to indicate it shouldn't be part of the public interface for `Booking`. If the point of this refactoring is to allow a booking to mutate to premium, it can be a public method.

With the structures set up, I start moving the behavior. First, the simple override of `hasTalkback`. I use Move Function to move the subclass method to the delegate, routing any access to superclass data through `_host`.

class PremiumBookingDelegate...

```
get hasTalkback() {
  return this._host._show.hasOwnProperty('talkback');
}
```

class PremiumBooking...

```
get hasTalkback() {
  return this._premiumDelegate.hasTalkback;
}
```

I test, then delete the subclass method. I run the tests expecting some to fail.

Now I finish the move by adding dispatch logic to the superclass method to use the delegate if present.

class Booking...

```
get hasTalkback() {
  return (this._premiumDelegate)
    ? this._premiumDelegate.hasTalkback
    : this._show.hasOwnProperty('talkback') && !this.isPeakDay;
}
```

The next case is the base price. The wrinkle is the call to `super` -- when I move the subclass code to the delegate, calling `this._host._basePrice` would cause endless recursion.

I have a couple of options. One is to apply Extract Function on the base calculation to separate the dispatch logic from price calculation.

class Booking...

```
get basePrice() {
  return (this._premiumDelegate)
    ? this._premiumDelegate.basePrice
    : this._privateBasePrice;
}

get _privateBasePrice() {
  let result = this._show.price;
  if (this.isPeakDay) result += Math.round(result * 0.15);
  return result;
}
```

class PremiumBookingDelegate...

```
get basePrice() {
  return Math.round(this._host._privateBasePrice + this._extras.premiumFee);
}
```

Alternatively, I can recast the delegate's method as an extension of the base method.

class Booking...

```
get basePrice() {
  let result = this._show.price;
  if (this.isPeakDay) result += Math.round(result * 0.15);
  return (this._premiumDelegate)
    ? this._premiumDelegate.extendBasePrice(result)
    : result;
}
```

class PremiumBookingDelegate...

```
extendBasePrice(base) {
  return Math.round(base + this._extras.premiumFee);
}
```

Both work; the latter is a bit smaller.

The last case is a method that only exists on the subclass. I move it from the subclass to the delegate:

class PremiumBookingDelegate...

```
get hasDinner() {
  return this._extras.hasOwnProperty('dinner') && !this._host.isPeakDay;
}
```

Then I add dispatch logic to `Booking`:

class Booking...

```
get hasDinner() {
  return (this._premiumDelegate)
    ? this._premiumDelegate.hasDinner
    : undefined;
}
```

In JavaScript, accessing a property on an object where it isn't defined returns `undefined`, so I do that here.

Once I've moved all the behavior out of the subclass, I change the factory method to return the superclass and delete the subclass.

top level...

```
function createPremiumBooking(show, date, extras) {
  const result = new PremiumBooking (show, date, extras);
  result._bePremium(extras);
  return result;
}

class PremiumBooking extends Booking ...
```

## Example: Replacing a Hierarchy

The same technique works for an entire hierarchy.

```
function createBird(data) {
  switch (data.type) {
    case 'EuropeanSwallow':
      return new EuropeanSwallow(data);
    case 'AfricanSwallow':
      return new AfricanSwallow(data);
    case 'NorweigianBlueParrot':
      return new NorwegianBlueParrot(data);
    default:
      return new Bird(data);
  }
}


class Bird {
  constructor(data) {
    this._name = data.name;
    this._plumage = data.plumage;
  }
  get name()    {return this._name;}

  get plumage() {
    return this._plumage || "average";
  }
  get airSpeedVelocity() {return null;}
}

class EuropeanSwallow extends Bird {
  get airSpeedVelocity() {return 35;}
}

class AfricanSwallow extends Bird {
  constructor(data) {
    super (data);
    this._numberOfCoconuts = data.numberOfCoconuts;
  }
  get airSpeedVelocity() {
    return 40 - 2 * this._numberOfCoconuts;
  }
}

class NorwegianBlueParrot extends Bird {
  constructor(data) {
    super (data);
    this._voltage = data.voltage;
    this._isNailed = data.isNailed;
  }

  get plumage() {
    if (this._voltage > 100) return "scorched";
    else return this._plumage || "beautiful";
  }

  get airSpeedVelocity() {
    return (this._isNailed) ? 0 : 10 + this._voltage / 10;
  }
}
```

The system will shortly need to differentiate between wild and captive birds. That difference could be modeled as subclasses of `Bird`, but inheritance can only be used once -- so I'll have to remove the species subclasses first.

I tackle them one at a time, starting with `EuropeanSwallow`. I create an empty delegate class.

```
class EuropeanSwallowDelegate {
}
```

I decide to handle delegate initialization in the constructor, using a selector function for the correct delegate.

class Bird...

```
constructor(data) {
  this._name = data.name;
  this._plumage = data.plumage;
  this._speciesDelegate = this.selectSpeciesDelegate(data);
}

selectSpeciesDelegate(data) {
  switch(data.type) {
    case 'EuropeanSwallow':
      return new EuropeanSwallowDelegate();
    default: return null;
  }
}
```

I apply Move Function to the European swallow's air speed velocity.

class EuropeanSwallowDelegate...

```
get airSpeedVelocity() {return 35;}
```

class EuropeanSwallow...

```
get airSpeedVelocity() {return this._speciesDelegate.airSpeedVelocity;}
```

I change `airSpeedVelocity` on the superclass to call the delegate, if present.

class Bird...

```
get airSpeedVelocity() {
  return this._speciesDelegate ? this._speciesDelegate.airSpeedVelocity : null;
}
```

I remove the `EuropeanSwallow` subclass.

Next I tackle the African swallow. I create a class; this time, the constructor needs the data document.

class AfricanSwallowDelegate...

```
constructor(data) {
  this._numberOfCoconuts = data.numberOfCoconuts;
}
```

class Bird...

```
selectSpeciesDelegate(data) {
  switch(data.type) {
    case 'EuropeanSwallow':
      return new EuropeanSwallowDelegate();
    case 'AfricanSwallow':
      return new AfricanSwallowDelegate(data);
    default: return null;
  }
}
```

I use Move Function on `airSpeedVelocity`.

class AfricanSwallowDelegate...

```
get airSpeedVelocity() {
  return 40 - 2 * this._numberOfCoconuts;
}
```

I remove the `AfricanSwallow` subclass.

Now for the Norwegian blue. Creating the class and moving the air speed velocity uses the same steps.

class Bird...

```
selectSpeciesDelegate(data) {
  switch(data.type) {
    case 'EuropeanSwallow':
      return new EuropeanSwallowDelegate();
    case 'AfricanSwallow':
      return new AfricanSwallowDelegate(data);
    case 'NorweigianBlueParrot':
      return new NorwegianBlueParrotDelegate(data);
    default: return null;
  }
}
```

class NorwegianBlueParrotDelegate...

```
constructor(data) {
  this._voltage = data.voltage;
  this._isNailed = data.isNailed;
}
get airSpeedVelocity() {
  return (this._isNailed) ? 0 : 10 + this._voltage / 10;
}
```

The Norwegian blue overrides the `plumage` property. The initial Move Function needs a back-reference to the bird in the constructor.

class NorwegianBlueParrotDelegate...

```
get plumage() {
  if (this._voltage > 100) return "scorched";
  else return this._bird._plumage || "beautiful";
}

constructor(data, bird) {
  this._bird = bird;
  this._voltage = data.voltage;
  this._isNailed = data.isNailed;
}
```

class Bird...

```
selectSpeciesDelegate(data) {
  switch(data.type) {
    case 'EuropeanSwallow':
      return new EuropeanSwallowDelegate();
    case 'AfricanSwallow':
      return new AfricanSwallowDelegate(data);
    case 'NorweigianBlueParrot':
      return new NorwegianBlueParrotDelegate(data, this);
    default: return null;
  }
}
```

The tricky step is removing the subclass method for `plumage`. If I do:

class Bird...

```
get plumage() {
  if (this._speciesDelegate)
    return this._speciesDelegate.plumage;
  else
    return this._plumage || "average";
}
```

I'll get errors because there is no plumage property on the other species' delegate classes.

I could implement the default case on each delegate, but this duplicates the default method. The solution is inheritance -- I apply Extract Superclass to the species delegates:

```
class SpeciesDelegate {
  constructor(data, bird) {
    this._bird = bird;
  }
  get plumage() {
    return this._bird._plumage || "average";
  }


class EuropeanSwallowDelegate extends SpeciesDelegate {


class AfricanSwallowDelegate extends SpeciesDelegate {
  constructor(data, bird) {
    super(data,bird);
    this._numberOfCoconuts = data.numberOfCoconuts;
  }


class NorwegianBlueParrotDelegate extends SpeciesDelegate {
  constructor(data, bird) {
    super(data, bird);
    this._voltage = data.voltage;
    this._isNailed = data.isNailed;
  }
```

Now I can move default behavior from `Bird` to `SpeciesDelegate` by ensuring there's always something in the `speciesDelegate` field.

class Bird...

```
selectSpeciesDelegate(data) {
  switch(data.type) {
    case 'EuropeanSwallow':
      return new EuropeanSwallowDelegate(data, this);
    case 'AfricanSwallow':
      return new AfricanSwallowDelegate(data, this);
    case 'NorweigianBlueParrot':
      return new NorwegianBlueParrotDelegate(data, this);
    default: return new SpeciesDelegate(data, this);
  }
}
// rest of bird's code...

get plumage() {return this._speciesDelegate.plumage;}

get airSpeedVelocity() {return this._speciesDelegate.airSpeedVelocity;}
```

class SpeciesDelegate...

```
get airSpeedVelocity() {return null;}
```

This simplifies the delegating methods on `Bird`. I can easily see which behavior is delegated and which stays.

Here's the final state:

```
function createBird(data) {
  return new Bird(data);
}


class Bird {
  constructor(data) {
    this._name = data.name;
    this._plumage = data.plumage;
    this._speciesDelegate = this.selectSpeciesDelegate(data);
  }
  get name()    {return this._name;}
  get plumage() {return this._speciesDelegate.plumage;}
  get airSpeedVelocity() {return this._speciesDelegate.airSpeedVelocity;}

  selectSpeciesDelegate(data) {
    switch(data.type) {
      case 'EuropeanSwallow':
        return new EuropeanSwallowDelegate(data, this);
      case 'AfricanSwallow':
        return new AfricanSwallowDelegate(data, this);
      case 'NorweigianBlueParrot':
        return new NorwegianBlueParrotDelegate(data, this);
      default: return new SpeciesDelegate(data, this);
    }
  }
  // rest of bird's code...
}

class SpeciesDelegate {
  constructor(data, bird) {
    this._bird = bird;
  }
  get plumage() {
    return this._bird._plumage || "average";
  }
  get airSpeedVelocity() {return null;}
}

class EuropeanSwallowDelegate extends SpeciesDelegate {
  get airSpeedVelocity() {return 35;}
}

class AfricanSwallowDelegate extends SpeciesDelegate {
  constructor(data, bird) {
    super(data,bird);
    this._numberOfCoconuts = data.numberOfCoconuts;
  }
  get airSpeedVelocity() {
    return 40 - 2 * this._numberOfCoconuts;
  }
}


class NorwegianBlueParrotDelegate extends SpeciesDelegate {
  constructor(data, bird) {
    super(data, bird);
    this._voltage = data.voltage;
    this._isNailed = data.isNailed;
  }
  get airSpeedVelocity() {
    return (this._isNailed) ? 0 : 10 + this._voltage / 10;
  }
  get plumage() {
    if (this._voltage > 100) return "scorched";
    else return this._bird._plumage || "beautiful";
  }
}
```

The species inheritance is now more tightly scoped, covering just the data and functions that vary due to species. Any code that's the same for all species remains on `Bird` and its future subclasses.

# Inline Class

```
class Person {
  get officeAreaCode() {return this._telephoneNumber.areaCode;}
  get officeNumber()   {return this._telephoneNumber.number;}
}
class TelephoneNumber {
  get areaCode() {return this._areaCode;}
  get number()   {return this._number;}
}
```

```
class Person {
  get officeAreaCode() {return this._officeAreaCode;}
  get officeNumber()   {return this._officeNumber;}
```

## Motivation

Folds a class that no longer justifies its existence into the class that uses it most, absorbing all its fields and methods.

## Mechanics

- In the target class, create functions for all the public functions of the source class. These functions should just delegate to the source class.

- Change all references to source class methods so they use the target class's delegators instead. Test after each change.

- Move all the functions and data from the source class into the target, testing after each move, until the source class is empty.

- Delete the source class.

## Example

Here's a class that holds tracking information for a shipment.

```js
class TrackingInformation {
  get shippingCompany()    {return this._shippingCompany;}
  set shippingCompany(arg) {this._shippingCompany = arg;}
  get trackingNumber()    {return this._trackingNumber;}
  set trackingNumber(arg) {this._trackingNumber = arg;}
  get display()            {
    return `${this.shippingCompany}: ${this.trackingNumber}`;
  }
}
```

It's used as part of a shipment class.

class Shipment...

```js
get trackingInfo() {
  return this._trackingInformation.display;
}
get trackingInformation()    {return this._trackingInformation;}
set trackingInformation(aTrackingInformation) {
  this._trackingInformation = aTrackingInformation;
}
```

I no longer feel this class is pulling its weight, so I want to inline it into `Shipment`.

I start by looking at places that invoke the methods of `TrackingInformation`.

caller...

```js
aShipment.trackingInformation.shippingCompany = request.vendor;
```

I put a delegating method into the shipment and adjust the client to call that.

class Shipment...

```js
set shippingCompany(arg) {this._trackingInformation.shippingCompany = arg;}
```

caller...

```js
aShipment.shippingCompany = request.vendor;
```

I do this for all the elements of tracking information used by clients. Once done, I move all the elements over into the shipment class.

I start by applying Inline Function to the `display` method.

class Shipment...

```js
get trackingInfo() {
  return `${this.shippingCompany}: ${this.trackingNumber}`;
}
```

I move the shipping company field.

```js
get shippingCompany()    {return this._shippingCompany;}
set shippingCompany(arg) {this._shippingCompany = arg;}
```

I don't use the full mechanics for Move Field since in this case I only reference `shippingCompany` from `Shipment` which is the target of the move, so I don't need the steps that put a reference from the source to the target.

I continue until everything is moved over, then delete the tracking information class.

class Shipment...

```js
get trackingInfo() {
  return `${this.shippingCompany}: ${this.trackingNumber}`;
}
get shippingCompany()    {return this._shippingCompany;}
set shippingCompany(arg) {this._shippingCompany = arg;}
get trackingNumber()    {return this._trackingNumber;}
set trackingNumber(arg) {this._trackingNumber = arg;}
```

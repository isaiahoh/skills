# Replace Command with Function

```
class ChargeCalculator {
  constructor (customer, usage){
    this._customer = customer;
    this._usage = usage;
  }
  execute() {
    return this._customer.rate * this._usage;
  }
}
```

```
function charge(customer, usage) {
  return customer.rate * usage;
}
```

## Motivation

Replaces a command object with a plain function when the command's complexity isn't justified -- most of the time you just want to invoke a function and have it do its thing.

## Mechanics

- Apply Extract Function to the creation of the command and the call to the command's execution method.

This creates the new function that will replace the command.

- For each method called by the command's execution method, apply Inline Function.

If the supporting function returns a value, use Extract Variable on the call first and then Inline Function.

- Use Change Function Declaration to put all the parameters of the constructor into the command's execution method instead.

- For each field, alter the references in the command's execution method to use the parameter instead. Test after each change.

- Inline the constructor call and command's execution method call into the caller (which is the replacement function).

- Test.
- Apply Remove Dead Code to the command class.

## Example

I'll begin with this small command object:

```
class ChargeCalculator {
  constructor (customer, usage, provider){
    this._customer = customer;
    this._usage = usage;
    this._provider = provider;
  }
  get baseCharge() {
    return this._customer.baseRate * this._usage;
  }
  get charge() {
    return this.baseCharge + this._provider.connectionCharge;
  }
}
```

It is used by code like this:

caller…

```
monthCharge = new ChargeCalculator(customer, usage, provider).charge;
```

The command class is small and simple enough to be better off as a function.

I begin by using Extract Function to wrap the class creation and invocation.

caller…

```
monthCharge = charge(customer, usage, provider);
```

top level…

```
function charge(customer, usage, provider) {
  return new ChargeCalculator(customer, usage, provider).charge;
}
```

I have to decide how to deal with any supporting functions, in this case `baseCharge`. My usual approach for a function that returns a value is to first Extract Variable on that value.

class ChargeCalculator…

```
get baseCharge() {
  return this._customer.baseRate * this._usage;
}
get charge() {
  const baseCharge = this.baseCharge;
  return baseCharge + this._provider.connectionCharge;
}
```

Then, I use Inline Function on the supporting function.

class ChargeCalculator…

```
get charge() {
  const baseCharge = this._customer.baseRate * this._usage;
  return baseCharge + this._provider.connectionCharge;
}
```

I now have all the processing in a single function, so my next step is to move the data passed to the constructor to the main method. I first use Change Function Declaration to add all the constructor parameters to the `charge` method.

class ChargeCalculator…

```
constructor (customer, usage, provider){
  this._customer = customer;
  this._usage = usage;
  this._provider = provider;
}
```

```
charge(customer, usage, provider) {
  const baseCharge = this._customer.baseRate * this._usage;
  return baseCharge + this._provider.connectionCharge;
}
```

top level…

```
function charge(customer, usage, provider) {
  return new ChargeCalculator(customer, usage, provider)
                      .charge(customer, usage, provider);
}
```

Now I can alter the body of `charge` to use the passed parameters instead. I can do this one at a time.

class ChargeCalculator…

```
constructor (customer, usage, provider){
  this._customer = customer;
  this._usage = usage;
  this._provider = provider;
}
```

```
charge(customer, usage, provider) {
  const baseCharge = customer.baseRate * this._usage;
  return baseCharge + this._provider.connectionCharge;
}
```

I don't have to remove the assignment to `this._customer` in the constructor, as it will just be ignored. But I prefer to do it since that will make a test fail if I miss changing a use of field to the parameter. (And if a test doesn't fail, I should consider adding a new test.)

I repeat this for the other parameters, ending up with

class ChargeCalculator…

```
charge(customer, usage, provider) {
  const baseCharge = customer.baseRate * usage;
  return baseCharge + provider.connectionCharge;
}
```

Once I've done all of these, I can inline into the top-level `charge` function. This is a special kind of Inline Function, as it's inlining both the constructor and method call together.

top level…

```
function charge(customer, usage, provider) {
    const baseCharge = customer.baseRate * usage;
    return baseCharge + provider.connectionCharge;
}
```

The command class is now dead code, so I'll use Remove Dead Code to give it an honorable burial.

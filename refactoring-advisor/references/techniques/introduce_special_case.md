# Introduce Special Case

```
if (aCustomer === "unknown") customerName = "occupant";
```

```
class UnknownCustomer {
    get name() {return "occupant";}
```

## Motivation

Replace widespread checks for a special-case value (e.g., null) with a Special Case object that encapsulates the common default behavior, eliminating duplicated conditional logic across clients.

## Mechanics

- Begin with a container data structure (or class) that contains a property which is the subject of the refactoring. Clients of the container compare the `subject` property to a special-case value. We wish to replace the special-case value of the subject with a special case class or data structure.
- Add a special-case check property to the subject, returning false.
- Create a special-case object with only the special-case check property, returning true.
- Apply Extract Function to the special-case comparison code. Ensure that all clients use the new function instead of directly comparing it.
- Introduce the new special-case subject into the code, either by returning it from a function call or by applying a transform function.
- Change the body of the special-case comparison function so that it uses the special-case check property.
- Test.
- Use Combine Functions into Class or Combine Functions into Transform to move all of the common special-case behavior into the new element. Since the special-case class usually returns fixed values to simple requests, these may be handled by making the special case a literal record.
- Use Inline Function on the special-case comparison function for the places where it's still needed.

## Example

A utility company installs its services in sites.

class Site...

```js
get customer() {return this._customer;}
```

There are various properties of the customer class; I'll consider three of them.

class Customer...

```js
get name()            {...}
get billingPlan()     {...}
set billingPlan(arg)  {...}
get paymentHistory()  {...}
```

Most of the time, a site has a customer, but sometimes there isn't one. When this happens, the data record fills the customer field with the string "unknown". Clients of the site need to be able to handle an unknown customer. Here are some example fragments:

client 1...

```js
const aCustomer = site.customer;
// ... lots of intervening code ...
let customerName;
if (aCustomer === "unknown") customerName = "occupant";
else customerName = aCustomer.name;
```

client 2...

```js
const plan = (aCustomer === "unknown") ?
      registry.billingPlans.basic
      : aCustomer.billingPlan;
```

client 3...

```js
if (aCustomer !== "unknown") aCustomer.billingPlan = newPlan;
```

client 4...

```js
const weeksDelinquent = (aCustomer === "unknown") ?
      0
      : aCustomer.paymentHistory.weeksDelinquentInLastYear;
```

Looking through the code base, I see many clients of the site object that have to deal with an unknown customer. Most of them do the same thing when they get one: They use "occupant" as the name, give them a basic billing plan, and class them as zero-weeks delinquent. This widespread testing for a special case, plus a common response, is what tells me it's time for a Special Case Object.

I begin by adding a method to the customer to indicate it is unknown.

class Customer...

```js
get isUnknown() {return false;}
```

I then add an Unknown Customer class.

```js
class UnknownCustomer {
  get isUnknown() {return true;}
}
```

Note that I don't make `UnknownCustomer` a subclass of `Customer`. In JavaScript, dynamic typing makes it better to not do that here.

Now comes the tricky bit. I have to return this new special-case object whenever I expect `"unknown"` and change each test for an unknown value to use the new `isUnknown` method. I use Extract Function on the special-case comparison code so I can change all clients one at a time.

```js
function isUnknown(arg) {
  if (!((arg instanceof Customer) || (arg === "unknown")))
    throw new Error(`investigate bad value: <${arg}>`);
  return (arg === "unknown");
}
```

I've put a trap in here for an unexpected value. This can help me to spot any mistakes or odd behavior as I'm doing this refactoring.

I can now use this function whenever I'm testing for an unknown customer. I can change these calls one at a time, testing after each change.

client 1...

```js
let customerName;
if (isUnknown(aCustomer)) customerName = "occupant";
else customerName = aCustomer.name;
```

client 2...

```js
const plan = (isUnknown(aCustomer)) ?
      registry.billingPlans.basic
      : aCustomer.billingPlan;
```

client 3...

```js
if (!isUnknown(aCustomer)) aCustomer.billingPlan = newPlan;
```

client 4...

```js
const weeksDelinquent = isUnknown(aCustomer) ?
      0
      : aCustomer.paymentHistory.weeksDelinquentInLastYear;
```

Once I've changed all the callers to use `isUnknown`, I can change the site class to return an unknown customer.

class Site...

```js
get customer() {
  return (this._customer === "unknown") ? new UnknownCustomer() :  this._customer;
}
```

I can check that I'm no longer using the "unknown" string by changing `isUnknown` to use the unknown value.

```js
function isUnknown(arg) {
  if (!(arg instanceof Customer || arg instanceof UnknownCustomer))
    throw new Error(`investigate bad value: <${arg}>`);
  return arg.isUnknown;
}
```

Now I can take each client's special-case check and replace it with a commonly expected value. I have various clients using "occupant" for the name of an unknown customer:

client 1...

```js
let customerName;
if (isUnknown(aCustomer)) customerName = "occupant";
else customerName = aCustomer.name;
```

I add a suitable method to the unknown customer:

class UnknownCustomer...

```js
get name() {return "occupant";}
```

Now I can make all that conditional code go away.

client 1...

```js
const customerName = aCustomer.name;
```

Next is the billing plan property.

client 2...

```js
const plan = (isUnknown(aCustomer)) ?
      registry.billingPlans.basic
      : aCustomer.billingPlan;
```

client 3...

```js
if (!isUnknown(aCustomer)) aCustomer.billingPlan = newPlan;
```

For read behavior, I do the same thing I did with the name. With the write behavior, the current code doesn't call the setter for an unknown customer -- so for the special case, I let the setter be called, but it does nothing.

class UnknownCustomer...

```js
get billingPlan()    {return registry.billingPlans.basic;}
set billingPlan(arg) { /* ignore */ }
```

client reader...

```js
const plan = aCustomer.billingPlan;
```

client writer...

```js
aCustomer.billingPlan = newPlan;
```

Special-case objects are value objects, and thus should always be immutable, even if the objects they are substituting for are not.

The last case is a bit more involved because the special case needs to return another object that has its own properties.

client...

```js
const weeksDelinquent = isUnknown(aCustomer) ?
      0
      : aCustomer.paymentHistory.weeksDelinquentInLastYear;
```

The general rule with a special-case object is that if it needs to return related objects, they are usually special cases themselves. So here I need to create a null payment history.

class UnknownCustomer...

```js
get paymentHistory() {return new NullPaymentHistory();}
```

class NullPaymentHistory...

```js
get weeksDelinquentInLastYear() {return 0;}
```

client...

```js
const weeksDelinquent = aCustomer.paymentHistory.weeksDelinquentInLastYear;
```

There will be exceptions -- clients that want to do something different with the special case. I may have 23 clients that use "occupant" for the name, but there's always one that needs something different.

client...

```js
const name =  ! isUnknown(aCustomer) ? aCustomer.name : "unknown occupant";
```

In that case, I need to retain a special-case check. I will change it to use the method on customer, using Inline Function on `isUnknown`.

client...

```js
const name =  aCustomer.isUnknown ? "unknown occupant" : aCustomer.name;
```

## Example: Using an Object Literal

Creating a class is a fair bit of work for a simple value. If I only read the data structure (no updates), I can use a literal object instead.

Here is the opening case again -- just the same, except this time there is no client that updates the customer:

class Site...

```js
get customer() {return this._customer;}
```

class Customer...

```js
get name()            {...}
get billingPlan()     {...}
set billingPlan(arg)  {...}
get paymentHistory()  {...}
```

client 1...

```js
const aCustomer = site.customer;
// ... lots of intervening code ...
let customerName;
if (aCustomer === "unknown") customerName = "occupant";
else customerName = aCustomer.name;
```

client 2...

```js
const plan = (aCustomer === "unknown") ?
      registry.billingPlans.basic
      : aCustomer.billingPlan;
```

client 3...

```js
const weeksDelinquent = (aCustomer === "unknown") ?
      0
      : aCustomer.paymentHistory.weeksDelinquentInLastYear;
```

I start by adding an `isUnknown` property to the customer and creating a special-case object with that field. The difference is that this time, the special case is a literal.

class Customer...

```js
get isUnknown() {return false;}
```

top level...

```js
function createUnknownCustomer() {
  return {
    isUnknown: true,
  };
}
```

I apply Extract Function to the special case condition test.

```js
function isUnknown(arg) {
  return (arg === "unknown");
}
```

client 1...

```js
let customerName;
if (isUnknown(aCustomer)) customerName = "occupant";
else customerName = aCustomer.name;
```

client 2...

```js
const plan = isUnknown(aCustomer) ?
      registry.billingPlans.basic
      : aCustomer.billingPlan;
```

client 3...

```js
const weeksDelinquent = isUnknown(aCustomer) ?
      0
      : aCustomer.paymentHistory.weeksDelinquentInLastYear;
```

I change the site class and the condition test to work with the special case.

class Site...

```js
get customer() {
  return (this._customer === "unknown") ?  createUnknownCustomer() :  this._customer;
}
```

top level...

```js
function isUnknown(arg) {
  return arg.isUnknown;
}
```

Then I replace each standard response with the appropriate literal value. I start with the name:

```js
function createUnknownCustomer() {
  return {
    isUnknown: true,
    name: "occupant",
  };
}
```

client 1...

```js
const customerName = aCustomer.name;
```

Then, the billing plan:

```js
function createUnknownCustomer() {
  return {
    isUnknown: true,
    name: "occupant",
    billingPlan: registry.billingPlans.basic,
  };
}
```

client 2...

```js
const plan = aCustomer.billingPlan;
```

Similarly, I can create a nested null payment history with the literal:

```js
function createUnknownCustomer() {
  return {
    isUnknown: true,
    name: "occupant",
    billingPlan: registry.billingPlans.basic,
    paymentHistory: {
      weeksDelinquentInLastYear: 0,
    },
  };
}
```

client 3...

```js
const weeksDelinquent = aCustomer.paymentHistory.weeksDelinquentInLastYear;
```

If I use a literal like this, I should make it immutable, which I might do with `freeze`. Usually, I'd rather use a class.

## Example: Using a Transform

The same idea can be applied to a record by using a transform step.

Let's assume our input is a simple record structure that looks something like this:

```js
{
  name: "Acme Boston",
  location: "Malden MA",
  // more site details
  customer: {
    name: "Acme Industries",
    billingPlan: "plan-451",
    paymentHistory: {
      weeksDelinquentInLastYear: 7
      //more
    },
    // more
  }
}
```

In some cases, the customer isn't known, and such cases are marked in the same way:

```js
{
  name: "Warehouse Unit 15",
  location: "Malden MA",
  // more site details
  customer: "unknown",
}
```

I have similar client code that checks for the unknown customer:

client 1...

```js
const site = acquireSiteData();
const aCustomer = site.customer;
// ... lots of intervening code ...
let customerName;
if (aCustomer === "unknown") customerName = "occupant";
else customerName = aCustomer.name;
```

client 2...

```js
const plan = (aCustomer === "unknown") ?
      registry.billingPlans.basic
      : aCustomer.billingPlan;
```

client 3...

```js
const weeksDelinquent = (aCustomer === "unknown") ?
      0
      : aCustomer.paymentHistory.weeksDelinquentInLastYear;
```

My first step is to run the site data structure through a transform that, currently, does nothing but a deep copy.

client 1...

```js
const rawSite = acquireSiteData();
const site = enrichSite(rawSite);
const aCustomer = site.customer;
// ... lots of intervening code ...
let customerName;
if (aCustomer === "unknown") customerName = "occupant";
else customerName = aCustomer.name;
```

```js
function enrichSite(inputSite) {
  return _.cloneDeep(inputSite);
}
```

I apply Extract Function to the test for an unknown customer.

```js
function isUnknown(aCustomer) {
  return aCustomer === "unknown";
}
```

client 1...

```js
const rawSite = acquireSiteData();
const site = enrichSite(rawSite);
const aCustomer = site.customer;
// ... lots of intervening code ...
let customerName;
if (isUnknown(aCustomer)) customerName = "occupant";
else customerName = aCustomer.name;
```

client 2...

```js
const plan = (isUnknown(aCustomer)) ?
      registry.billingPlans.basic
      : aCustomer.billingPlan;
```

client 3...

```js
const weeksDelinquent = (isUnknown(aCustomer)) ?
      0
      : aCustomer.paymentHistory.weeksDelinquentInLastYear;
```

I begin the enrichment by adding an `isUnknown` property to the customer.

```js
function enrichSite(aSite) {
  const result = _.cloneDeep(aSite);
  const unknownCustomer = {
    isUnknown: true,
  };

  if (isUnknown(result.customer)) result.customer = unknownCustomer;
  else result.customer.isUnknown = false;
  return result;
}
```

I can then modify the special-case condition test to include probing for this new property. I keep the original test as well, so that the test will work on both raw and enriched sites.

```js
function isUnknown(aCustomer) {
  if (aCustomer === "unknown") return true;
  else return aCustomer.isUnknown;
}
```

I test to ensure that's all OK, then start applying Combine Functions into Transform on the special case. First, I move the choice of name into the enrichment function.

```js
function enrichSite(aSite) {
  const result = _.cloneDeep(aSite);
  const unknownCustomer = {
    isUnknown: true,
    name: "occupant",
  };

  if (isUnknown(result.customer)) result.customer = unknownCustomer;
  else result.customer.isUnknown = false;
  return result;
}
```

client 1...

```js
const rawSite = acquireSiteData();
const site = enrichSite(rawSite);
const aCustomer = site.customer;
// ... lots of intervening code ...
const customerName = aCustomer.name;
```

I test, then do the billing plan.

```js
function enrichSite(aSite) {
  const result = _.cloneDeep(aSite);
  const unknownCustomer = {
    isUnknown: true,
    name: "occupant",
    billingPlan: registry.billingPlans.basic,
  };

  if (isUnknown(result.customer)) result.customer = unknownCustomer;
  else result.customer.isUnknown = false;
  return result;
}
```

client 2...

```js
const plan = aCustomer.billingPlan;
```

I test again, then do the last client.

```js
function enrichSite(aSite) {
  const result = _.cloneDeep(aSite);
  const unknownCustomer = {
    isUnknown: true,
    name: "occupant",
    billingPlan: registry.billingPlans.basic,
    paymentHistory: {
      weeksDelinquentInLastYear: 0,
    }
  };

  if (isUnknown(result.customer)) result.customer = unknownCustomer;
  else result.customer.isUnknown = false;
  return result;
}
```

client 3...

```js
const weeksDelinquent = aCustomer.paymentHistory.weeksDelinquentInLastYear;
```

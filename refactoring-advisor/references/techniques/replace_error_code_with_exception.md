# Replace Error Code with Exception

```
if (data)
  return new ShippingRules(data);
else
  return -23;
```

```
if (data)
  return new ShippingRules(data);
else
  throw new OrderProcessingError(-23);
```

## Motivation

Replaces error codes with exceptions so callers don't have to check and propagate error values manually. Only use for truly exceptional behavior -- if replacing the throw with program termination wouldn't make sense, the error should be part of normal flow instead.

## Mechanics

- Create an exception handler higher up the call chain to handle the exception.

This handler should initially just rethrow all exceptions.

If there already is a handler that does the right thing, extend it to cover the new part of the call chain.

- Test.
- Choose an appropriate marker to distinguish the exceptions replacing the error code from other exceptions.

Pick an approach that fits the language; many languages make it natural to use a subclass.

- Run static checks.
- Modify the `catch` clause to perform the error action when it's called with the correct kind of exception and rethrow otherwise.

- Test.
- For each place that returns an error code, replace it with throwing the new exception. Test after each change.

- When all the points that return error codes have been replaced, remove any code that passes the error codes up the call stack. Test after each change.

It's often useful to first replace the error code check with a trap, then remove it after testing that the trap isn't triggered.

This will expire any code that checks for error codes, so use Remove Dead Code to clear up the bodies.

## Example

Consider some code to look up shipping rates from a global table.

```
function localShippingRules(country) {
  const data = countryData.shippingRules[country];
  if (data) return new ShippingRules(data);
  else return -23;
}
```

The code here assumes the country should have been validated earlier on, so getting such an error implies something odd has happened. The caller checks for this, and passes on the error it finds.

```
function calculateShippingCosts(anOrder) {
  // irrelevant code
  const shippingRules = localShippingRules(anOrder.country);
  if (shippingRules < 0) return shippingRules;
  // more irrelevant code
```

A higher-level function reacts to the error by pushing the order into an error list.

top level…

```
const status = calculateShippingCosts(orderData);
if (status < 0) errorList.push({order: orderData, errorCode: status});
```

The first thing I need to consider here is whether the error is expected or not. Should `localShippingRules` assume that the shipping rules are properly loaded into `countryData`? Does the country argument come from a place where it ought to match with the keys in the global data, or has it gone through a validation step?

Assuming I answer those questions in the affirmative, I'm ready to apply Replace Error Code with Exception.

I begin by forming an exception handler at the top level. I want to wrap the call to `localShippingRules` in a `try` block, but I don't want to include the handling logic there. I can't do

top level…

```
try {
  const status = calculateShippingCosts(orderData);
} catch (e) {
  // something
}
if (status < 0) errorList.push({order: orderData, errorCode: status});
```

because then, `status` will not be in scope to be checked in the conditional.

So, first I need to split the declaration of `status` from its initialization.

top level…

```
let status;
status = calculateShippingCosts(orderData);
if (status < 0) errorList.push({order: orderData, errorCode: status});
```

I can then wrap the call within a `try`/`catch` block.

top level…

```
let status;
try {
  status = calculateShippingCosts(orderData);
} catch (e){
  throw e;
}
if (status < 0) errorList.push({order: orderData, errorCode: status});
```

I need to rethrow any exception I catch -- if there are exceptions elsewhere in the code, I don't want to inadvertently swallow them.

There may already be an appropriate handler that adds the order to an error list for another part of the calling code. In that case, I'd adjust the `try` block so it includes the call to `calculateShippingCosts`.

To handle only the exceptions I'm introducing through this refactoring, I need a way to distinguish them from other exceptions. I can either make them a different class or give them a particular property value. Many languages allow exceptions to be caught based on their class, in which case using a subclass is a natural fit.

```
class OrderProcessingError extends Error {
  constructor(errorCode) {
    super(`Order processing error ${errorCode}`);
    this.code = errorCode;
  }
  get name() {return "OrderProcessingError";}
}
```

With that class written, I can now add logic to handle it, the same way I did for the error code.

top level…

```
let status;
try {
  status = calculateShippingCosts(orderData);
} catch (e){
  if (e instanceof OrderProcessingError)
    errorList.push({order: orderData, errorCode: e.code});
  else
    throw e;
}
if (status < 0) errorList.push({order: orderData, errorCode: status});
```

I then modify the error detection code to throw the exception instead of returning an error code.

```
function localShippingRules(country) {
  const data = countryData.shippingRules[country];
  if (data) return new ShippingRules(data);
  else throw new OrderProcessingError(-23);
}
```

Once that's written and tested, I can remove the intermediate error code forwarding, but I like to first put a trap in and test.

```
function calculateShippingCosts(anOrder) {
  // irrelevant code
  const shippingRules = localShippingRules(anOrder.country);
  if (shippingRules < 0) throw new Error("error code not dead yet");
  // more irrelevant code
```

Assuming the trap isn't triggered, I can delete the trap line, relying on the exception mechanism to pass the error up the call stack.

```
function calculateShippingCosts(anOrder) {
  // irrelevant code
  const shippingRules = localShippingRules(anOrder.country);
  // more irrelevant code
```

I can then remove the now-unnecessary `status` variable and error-code check too.

top level…

```
try {
  calculateShippingCosts(orderData);
} catch (e){
  if (e instanceof OrderProcessingError)
    errorList.push({order: orderData, errorCode: e.code});
  else
    throw e;
}
```

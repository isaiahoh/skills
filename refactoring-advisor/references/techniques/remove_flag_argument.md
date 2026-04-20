# Remove Flag Argument

## Motivation

Replaces a function that uses a flag argument (a literal value selecting which logic to execute) with explicit functions for each case, making the caller's intent clear. Only applies when callers pass literals to control flow -- if the argument carries data flowing through the program, it is not a flag argument.

## Mechanics

- Create an explicit function for each value of the parameter.

  If the main function has a clear dispatch conditional, use Decompose Conditional to create the explicit functions. Otherwise, create wrapping functions.

- For each caller that uses a literal value for the parameter, replace it with a call to the explicit function.

## Example

Looking through some code, I see calls to calculate a delivery date for a shipment. Some of the calls look like

```javascript
aShipment.deliveryDate = deliveryDate(anOrder, true);
```

and some look like

```javascript
aShipment.deliveryDate = deliveryDate(anOrder, false);
```

Faced with code like this, I immediately begin to wonder about the meaning of the boolean value.

The body of `deliveryDate` looks like this:

```javascript
function deliveryDate(anOrder, isRush) {
  if (isRush) {
    let deliveryTime;
    if (["MA", "CT"]     .includes(anOrder.deliveryState)) deliveryTime = 1;
    else if (["NY", "NH"].includes(anOrder.deliveryState)) deliveryTime = 2;
    else deliveryTime = 3;
    return anOrder.placedOn.plusDays(1 + deliveryTime);
  }
  else {
    let deliveryTime;
    if (["MA", "CT", "NY"].includes(anOrder.deliveryState)) deliveryTime = 2;
    else if (["ME", "NH"] .includes(anOrder.deliveryState)) deliveryTime = 3;
    else deliveryTime = 4;
    return anOrder.placedOn.plusDays(2 + deliveryTime);
  }
}
```

The caller is using a literal boolean value to determine which code should run -- a classic flag argument. It is better to clarify the caller's intent with explicit functions.

In this case, I can do this by using Decompose Conditional, which gives me this:

```javascript
function deliveryDate(anOrder, isRush) {
  if (isRush) return rushDeliveryDate(anOrder);
  else        return regularDeliveryDate(anOrder);
}

function rushDeliveryDate(anOrder) {
    let deliveryTime;
    if (["MA", "CT"]     .includes(anOrder.deliveryState)) deliveryTime = 1;
    else if (["NY", "NH"].includes(anOrder.deliveryState)) deliveryTime = 2;
    else deliveryTime = 3;
    return anOrder.placedOn.plusDays(1 + deliveryTime);
}
function regularDeliveryDate(anOrder) {
    let deliveryTime;
    if (["MA", "CT", "NY"].includes(anOrder.deliveryState)) deliveryTime = 2;
    else if (["ME", "NH"] .includes(anOrder.deliveryState)) deliveryTime = 3;
    else deliveryTime = 4;
    return anOrder.placedOn.plusDays(2 + deliveryTime);
}
```

The two new functions capture the intent of the call better, so I can replace each call of

```javascript
aShipment.deliveryDate = deliveryDate(anOrder, true);
```

with

```javascript
aShipment.deliveryDate = rushDeliveryDate(anOrder);
```

and similarly with the other case.

When I've replaced all the callers, I remove `deliveryDate`.

A flag argument isn't just the presence of a boolean value; it's that the boolean is set with a literal rather than data. If all the callers of `deliveryDate` were like this:

```javascript
const isRush = determineIfRush(anOrder);
aShipment.deliveryDate = deliveryDate(anOrder, isRush);
```

then I'd have no problem with `deliveryDate`'s signature (although I'd still want to apply Decompose Conditional).

It may be that some callers use the argument as a flag by setting it with a literal, while others set the argument with data. In that case, I'd still use Remove Flag Argument, but not change the data callers and not remove `deliveryDate` at the end. That way I support both interfaces for the different uses.

Decomposing the conditional is a good way to carry out this refactoring, but it only works if the dispatch on the parameter is the outer part of the function (or can easily be refactored to make it so). It's also possible that the parameter is used in a much more tangled way, such as this alternative version of `deliveryDate`:

```javascript
function deliveryDate(anOrder, isRush) {
  let result;
  let deliveryTime;
  if (anOrder.deliveryState === "MA" || anOrder.deliveryState === "CT")
    deliveryTime = isRush? 1 : 2;
  else if (anOrder.deliveryState === "NY" || anOrder.deliveryState === "NH") {
    deliveryTime = 2;
    if (anOrder.deliveryState === "NH" && !isRush)
      deliveryTime = 3;
  }
  else if (isRush)
    deliveryTime = 3;
  else if (anOrder.deliveryState === "ME")
    deliveryTime = 3;
  else
    deliveryTime = 4;
  result = anOrder.placedOn.plusDays(2 + deliveryTime);
  if (isRush) result = result.minusDays(1);
  return result;
}
```

In this case, teasing out `isRush` into a top-level dispatch conditional is likely more work than I fancy. So instead, I can layer functions over the `deliveryDate`:

```javascript
function rushDeliveryDate   (anOrder) {return deliveryDate(anOrder, true);}
function regularDeliveryDate(anOrder) {return deliveryDate(anOrder, false);}
```

These wrapping functions are essentially partial applications of `deliveryDate`.

I can then do the same replacement of callers that I did with the decomposed conditional earlier on. If there aren't any callers using the parameter as data, I like to restrict its visibility or rename it to a name that conveys that it shouldn't be used directly (e.g., `deliveryDateHelperOnly`).

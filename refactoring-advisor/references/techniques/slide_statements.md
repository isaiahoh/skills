# Slide Statements

```
const pricingPlan = retrievePricingPlan();
const order = retreiveOrder();
let charge;
const chargePerUnit = pricingPlan.unit;
```

```
const pricingPlan = retrievePricingPlan();
const chargePerUnit = pricingPlan.unit;
const order = retreiveOrder();
let charge;
```

## Motivation

Slide related statements next to each other so that code accessing the same data is contiguous, typically as a preparatory step for Extract Function.

## Mechanics

- Identify the target position to move the fragment to. Examine statements between source and target to see if there is interference for the candidate fragment. Abandon action if there is any interference.

  A fragment cannot slide backwards earlier than any element it references is declared.

  A fragment cannot slide forwards beyond any element that references it.

  A fragment cannot slide over any statement that modifies an element it references.

  A fragment that modifies an element cannot slide over any other element that references the modified element.

- Cut the fragment from the source and paste into the target position.

- Test.
- If the test fails, try breaking down the slide into smaller steps. Either slide over less code or reduce the amount of code in the fragment you're moving.

## Example

When sliding code fragments, there are two decisions involved: what slide I'd like to do and whether I can do it. The first decision is very context-specific. On the simplest level, I like to declare elements close to where I use them, so I'll often slide a declaration down to its usage. But almost always I slide some code because I want to do another refactoring -- perhaps to get a clump of code together to Extract Function.

Once I have a sense of where I'd like to move some code, the next part is deciding if I can do it. This involves looking at the code I'm sliding and the code I'm sliding over: Do they interfere with each other in a way that would change the observable behavior of the program?

Consider the following fragment of code:

```
 1 const pricingPlan = retrievePricingPlan();
 2 const order = retreiveOrder();
 3 const baseCharge = pricingPlan.base;
 4 let charge;
 5 const chargePerUnit = pricingPlan.unit;
 6 const units = order.units;
 7 let discount;
 8 charge = baseCharge + units * chargePerUnit;
 9 let discountableUnits = Math.max(units - pricingPlan.discountThreshold, 0);
10 discount = discountableUnits * pricingPlan.discountFactor;
11 if (order.isRepeat) discount += 20;
12 charge = charge - discount;
13 chargeOrder(charge);
```

The first seven lines are declarations, and it's relatively easy to move these. For example, I may want to move all the code dealing with discounts together, which would involve moving line 7 (`let discount`) to above line 10 (`discount = ...`). Since a declaration has no side effects and refers to no other variable, I can safely move this forwards as far as the first line that references `discount` itself. This is also a common move -- if I want to use Extract Function on the discount logic, I'll need to move the declaration down first.

I do similar analysis with any code that doesn't have side effects. So I can take line 2 (`const order = ...`) and move it down to above line 6 (`const units = ...`) without trouble.

In this case, I'm also helped by the fact that the code I'm moving over doesn't have side effects either. I can freely rearrange code that lacks side effects, which is one of the reasons why wise programmers prefer to use side-effect-free code as much as possible.

There is a wrinkle here, however. How do I know that line 2 is side-effect-free? To be sure, I'd need to look inside `retrieveOrder()` to ensure there are no side effects there. In practice, following the Command-Query Separation principle means any function that returns a value is free of side effects. I can only be confident of that when I know the code base; in an unknown code base, I'd have to be more cautious.

When sliding code that has a side effect, or sliding over code with side effects, I have to be much more careful. What I'm looking for is interference between the two code fragments. So, let's say I want to slide line 11 (`if (order.isRepeat) ...`) down to the end. I'm prevented from doing that by line 12 because it references the variable whose state I'm changing in line 11. Similarly, I can't take line 13 (`chargeOrder(charge)`) and move it up because line 12 modifies some state that line 13 references. However, I can slide line 8 (`charge = baseCharge + ...`) over lines 9-11 because they don't modify any common state.

The most straightforward rule to follow is that I can't slide one fragment of code over another if any data that both fragments refer to is modified by either one. But that's not a comprehensive rule; I can happily slide either of the following two lines over the other:

```
a = a + 10;
a = a + 5;
```

But judging whether a slide is safe means I have to really understand the operations involved and how they compose.

Since I need to worry so much about updating state, I look to remove as much of it as I can. So with this code, I'd be looking to apply Split Variable on `charge` before I indulge in any sliding around of that code.

Here, the analysis is relatively simple because I'm mostly just modifying local variables. With more complex data structures, it's much harder to be sure when I get interference. So tests play an important role: Slide the fragment, run tests, see if things break. If my test coverage is good, I can feel happy with the refactoring. But if tests aren't reliable, I need to be more wary -- or, more likely, to improve the tests for the code I'm working on.

The most important consequence of a test failure after a slide is to use smaller slides: Instead of sliding over ten lines, I'll just pick five, or slide up to what I reckon is a dangerous line. It may also mean that the slide isn't worth it, and I need to work on something else first.

## Example: Sliding with Conditionals

I can also do slides with conditionals. This will either involve removing duplicate logic when I slide out of a conditional, or adding duplicate logic when I slide in.

Here's a case where I have the same statements in both legs of a conditional:

```
let result;
if (availableResources.length === 0) {
  result = createResource();
  allocatedResources.push(result);
} else {
  result = availableResources.pop();
  allocatedResources.push(result);
}
return result;
```

I can slide these out of the conditional, in which case they turn into a single statement outside of the conditional block.

```
let result;
if (availableResources.length === 0) {
  result = createResource();
} else {
  result = availableResources.pop();
}
allocatedResources.push(result);
return result;
```

In the reverse case, sliding a fragment into a conditional means repeating it in every leg of the conditional.

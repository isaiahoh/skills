# Replace Conditional with Polymorphism

```
switch (bird.type) {
  case 'EuropeanSwallow':
    return "average";
  case 'AfricanSwallow':
    return (bird.numberOfCoconuts > 2) ? "tired" : "average";
  case 'NorwegianBlueParrot':
    return (bird.voltage > 100) ? "scorched" : "beautiful";
  default:
    return "unknown";
```

```
class EuropeanSwallow {
  get plumage() {
    return "average";
  }
class AfricanSwallow {
  get plumage() {
     return (this.numberOfCoconuts > 2) ? "tired" : "average";
  }
class NorwegianBlueParrot {
  get plumage() {
     return (this.voltage > 100) ? "scorched" : "beautiful";
  }
```

## Motivation

Replace complex conditional logic (especially repeated switches on a type code) with a class hierarchy where each subclass handles one variant, removing duplicated branching and letting you reason about the base case and each variant independently.

## Mechanics

- If classes do not exist for polymorphic behavior, create them together with a factory function to return the correct instance.
- Use the factory function in calling code.
- Move the conditional function to the superclass. If the conditional logic is not a self-contained function, use Extract Function to make it so.
- Pick one of the subclasses. Create a subclass method that overrides the conditional statement method. Copy the body of that leg of the conditional statement into the subclass method and adjust it to fit.
- Repeat for each leg of the conditional.
- Leave a default case for the superclass method. Or, if superclass should be abstract, declare that method as abstract or throw an error to show it should be the responsibility of a subclass.

## Example

My friend has a collection of birds and wants to know how fast they can fly and what they have for plumage. So we have a couple of small programs to determine the information.

```js
function plumages(birds) {
  return new Map(birds.map(b => [b.name, plumage(b)]));
}
function speeds(birds) {
  return new Map(birds.map(b => [b.name, airSpeedVelocity(b)]));
}
```

```js
function plumage(bird) {
  switch (bird.type) {
  case 'EuropeanSwallow':
    return "average";
  case 'AfricanSwallow':
    return (bird.numberOfCoconuts > 2) ? "tired" : "average";
  case 'NorwegianBlueParrot':
    return (bird.voltage > 100) ? "scorched" : "beautiful";
  default:
    return "unknown";
  }
}
```

```js
function airSpeedVelocity(bird) {
  switch (bird.type) {
  case 'EuropeanSwallow':
    return 35;
  case 'AfricanSwallow':
    return 40 - 2 * bird.numberOfCoconuts;
  case 'NorwegianBlueParrot':
    return (bird.isNailed) ? 0 : 10 + bird.voltage / 10;
  default:
    return null;
  }
}
```

We have a couple of different operations that vary with the type of bird, so it makes sense to create classes and use polymorphism for any type-specific behavior.

I begin by using Combine Functions into Class on `airSpeedVelocity` and `plumage`.

```js
function plumage(bird) {
  return new Bird(bird).plumage;
}
```

```js
function airSpeedVelocity(bird) {
  return new Bird(bird).airSpeedVelocity;
}
```

```js
class Bird {
  constructor(birdObject) {
    Object.assign(this, birdObject);
  }
  get plumage() {
    switch (this.type) {
    case 'EuropeanSwallow':
      return "average";
    case 'AfricanSwallow':
      return (this.numberOfCoconuts > 2) ? "tired" : "average";
    case 'NorwegianBlueParrot':
      return (this.voltage > 100) ? "scorched" : "beautiful";
    default:
      return "unknown";
    }
  }
  get airSpeedVelocity() {
    switch (this.type) {
    case 'EuropeanSwallow':
      return 35;
    case 'AfricanSwallow':
      return 40 - 2 * this.numberOfCoconuts;
    case 'NorwegianBlueParrot':
      return (this.isNailed) ? 0 : 10 + this.voltage / 10;
    default:
      return null;
    }
  }
}
```

I now add subclasses for each kind of bird, together with a factory function to instantiate the appropriate subclass.

```js
function plumage(bird) {
  return createBird(bird).plumage;
}
```

```js
function airSpeedVelocity(bird) {
  return createBird(bird).airSpeedVelocity;
}
```

```js
function createBird(bird) {
  switch (bird.type) {
  case 'EuropeanSwallow':
    return new EuropeanSwallow(bird);
  case 'AfricanSwallow':
    return new AfricanSwallow(bird);
  case 'NorweigianBlueParrot':
    return new NorwegianBlueParrot(bird);
  default:
    return new Bird(bird);
  }
}
```

```js
class EuropeanSwallow extends Bird {
}

class AfricanSwallow extends Bird {
}

class NorwegianBlueParrot extends Bird {
}
```

Now that I've created the class structure that I need, I can begin on the two conditional methods. I'll begin with plumage. I take one leg of the switch statement and override it in the appropriate subclass.

class EuropeanSwallow...

```js
get plumage() {
  return "average";
}
```

I put in the `throw` because I'm paranoid.

class Bird...

```js
get plumage() {
  switch (this.type) {
  case 'EuropeanSwallow':
    throw "oops";
  case 'AfricanSwallow':
    return (this.numberOfCoconuts > 2) ? "tired" : "average";
  case 'NorwegianBlueParrot':
    return (this.voltage > 100) ? "scorched" : "beautiful";
  default:
    return "unknown";
  }
}
```

I can compile and test at this point. Then, if all is well, I do the next leg.

class AfricanSwallow...

```js
get plumage() {
   return (this.numberOfCoconuts > 2) ? "tired" : "average";
}
```

Then, the Norwegian Blue:

class NorwegianBlueParrot...

```js
get plumage() {
   return (this.voltage > 100) ? "scorched" : "beautiful";
}
```

I leave the superclass method for the default case.

class Bird...

```js
get plumage() {
  return "unknown";
}
```

I repeat the same process for `airSpeedVelocity`. Once I'm done, I end up with the following code (I also inlined the top-level functions for `airSpeedVelocity` and `plumage`):

```js
function plumages(birds) {
  return new Map(birds
                 .map(b => createBird(b))
                 .map(bird => [bird.name, bird.plumage]));
}
function speeds(birds) {
  return new Map(birds
                 .map(b => createBird(b))
                 .map(bird => [bird.name, bird.airSpeedVelocity]));
}
```

```js
function createBird(bird) {
  switch (bird.type) {
  case 'EuropeanSwallow':
    return new EuropeanSwallow(bird);
  case 'AfricanSwallow':
    return new AfricanSwallow(bird);
  case 'NorwegianBlueParrot':
    return new NorwegianBlueParrot(bird);
  default:
    return new Bird(bird);
  }
}
```

```js
class Bird {
  constructor(birdObject) {
    Object.assign(this, birdObject);
  }
  get plumage() {
    return "unknown";
  }
  get airSpeedVelocity() {
    return null;
  }
}
class EuropeanSwallow extends Bird {
  get plumage() {
    return "average";
  }
  get airSpeedVelocity() {
    return 35;
  }
}

class AfricanSwallow extends Bird {
  get plumage() {
    return (this.numberOfCoconuts > 2) ? "tired" : "average";
  }
  get airSpeedVelocity() {
    return 40 - 2 * this.numberOfCoconuts;
  }
}
class NorwegianBlueParrot extends Bird {
  get plumage() {
    return (this.voltage > 100) ? "scorched" : "beautiful";
  }
  get airSpeedVelocity() {
    return (this.isNailed) ? 0 : 10 + this.voltage / 10;
  }
}
```

## Example: Using Polymorphism for Variation

Another case for inheritance is when I wish to indicate that one object is mostly similar to another, but with some variations.

Consider some code used by a rating agency to compute an investment rating for the voyages of sailing ships. The rating agency gives out either an "A" or "B" rating, depending of various factors due to risk and profit potential.

```js
function rating(voyage, history) {
  const vpf = voyageProfitFactor(voyage, history);
  const vr = voyageRisk(voyage);
  const chr = captainHistoryRisk(voyage, history);
  if (vpf * 3 > (vr + chr * 2)) return "A";
  else return "B";
}

function voyageRisk(voyage) {
  let result = 1;
  if (voyage.length > 4) result += 2;
  if (voyage.length > 8) result += voyage.length - 8;
  if (["china", "east-indies"].includes(voyage.zone)) result += 4;
  return Math.max(result, 0);
}
function captainHistoryRisk(voyage, history) {
  let result = 1;
  if (history.length < 5) result += 4;
  result += history.filter(v => v.profit < 0).length;
  if (voyage.zone === "china" && hasChina(history)) result -= 2;
  return Math.max(result, 0);
}
function hasChina(history) {
  return history.some(v => "china" === v.zone);
}
function voyageProfitFactor(voyage, history) {
  let result = 2;
  if (voyage.zone === "china") result += 1;
  if (voyage.zone === "east-indies") result += 1;
  if (voyage.zone === "china" && hasChina(history)) {
    result += 3;
    if (history.length > 10) result += 1;
    if (voyage.length > 12) result += 1;
    if (voyage.length > 18) result -= 1;
  }
  else {
    if (history.length > 8) result += 1;
    if (voyage.length > 14) result -= 1;
  }
  return result;
}
```

The calling code would look something like this:

```js
const voyage = {zone: "west-indies", length: 10};
const history = [
  {zone: "east-indies", profit:  5},
  {zone: "west-indies", profit: 15},
  {zone: "china",       profit: -2},
  {zone: "west-africa", profit:  7},
];

const myRating = rating(voyage, history);
```

What I want to focus on here is how a couple of places use conditional logic to handle the case of a voyage to China where the captain has been to China before. I will use inheritance and polymorphism to separate out the logic for handling these cases from the base logic. This is a particularly useful refactoring if I'm about to introduce more special logic for this case -- and the logic for these repeat China voyages can make it harder to understand the base case.

I'm beginning with a set of functions. To introduce polymorphism, I need to create a class structure, so I begin by applying Combine Functions into Class. This results in the following code:

```js
function rating(voyage, history) {
  return new Rating(voyage, history).value;
}
```

```js
class Rating {
  constructor(voyage, history) {
    this.voyage = voyage;
    this.history = history;
  }
  get value() {
    const vpf = this.voyageProfitFactor;
    const vr = this.voyageRisk;
    const chr = this.captainHistoryRisk;
    if (vpf * 3 > (vr + chr * 2)) return "A";
    else return "B";
  }
  get voyageRisk() {
    let result = 1;
    if (this.voyage.length > 4) result += 2;
    if (this.voyage.length > 8) result += this.voyage.length - 8;
    if (["china", "east-indies"].includes(this.voyage.zone)) result += 4;
    return Math.max(result, 0);
  }
  get captainHistoryRisk() {
    let result = 1;
    if (this.history.length < 5) result += 4;
    result += this.history.filter(v => v.profit < 0).length;
    if (this.voyage.zone === "china" && this.hasChinaHistory) result -= 2;
    return Math.max(result, 0);
  }
  get voyageProfitFactor() {
    let result = 2;

    if (this.voyage.zone === "china") result += 1;
    if (this.voyage.zone === "east-indies") result += 1;
    if (this.voyage.zone === "china" && this.hasChinaHistory) {
      result += 3;
      if (this.history.length > 10) result += 1;
      if (this.voyage.length > 12) result += 1;
      if (this.voyage.length > 18) result -= 1;
    }
    else {
      if (this.history.length > 8) result += 1;
      if (this.voyage.length > 14) result -= 1;
    }
    return result;
  }

  get hasChinaHistory() {
    return this.history.some(v => "china" === v.zone);
  }
}
```

That's given me the class for the base case. I now need to create an empty subclass to house the variant behavior.

```js
class ExperiencedChinaRating extends Rating {
}
```

I then create a factory function to return the variant class when needed.

```js
function createRating(voyage, history) {
  if (voyage.zone === "china" && history.some(v => "china" === v.zone))
    return new ExperiencedChinaRating(voyage, history);
  else return new Rating(voyage, history);
}
```

I need to modify any callers to use the factory function instead of directly invoking the constructor.

```js
function rating(voyage, history) {
  return createRating(voyage, history).value;
}
```

There are two bits of behavior I need to move into a subclass. I begin with the logic in `captainHistoryRisk`:

class Rating...

```js
get captainHistoryRisk() {
  let result = 1;
  if (this.history.length < 5) result += 4;
  result += this.history.filter(v => v.profit < 0).length;
  if (this.voyage.zone === "china" && this.hasChinaHistory) result -= 2;
  return Math.max(result, 0);
}
```

I write the overriding method in the subclass:

class ExperiencedChinaRating...

```js
get captainHistoryRisk() {
  const result =  super.captainHistoryRisk - 2;
  return Math.max(result, 0);
}
```

class Rating...

```js
get captainHistoryRisk() {
  let result = 1;
  if (this.history.length < 5) result += 4;
  result += this.history.filter(v => v.profit < 0).length;
  return Math.max(result, 0);
}
```

Separating the variant behavior from `voyageProfitFactor` is a bit more messy. I can't simply remove the variant behavior and call the superclass method since there is an alternative path here. I also don't want to copy the whole superclass method down to the subclass.

class Rating...

```js
get voyageProfitFactor() {
  let result = 2;

  if (this.voyage.zone === "china") result += 1;
  if (this.voyage.zone === "east-indies") result += 1;
  if (this.voyage.zone === "china" && this.hasChinaHistory) {
    result += 3;
    if (this.history.length > 10) result += 1;
    if (this.voyage.length > 12) result += 1;
    if (this.voyage.length > 18) result -= 1;
  }
  else {
    if (this.history.length > 8) result += 1;
    if (this.voyage.length > 14) result -= 1;
  }
  return result;
}
```

So my response is to first use Extract Function on the entire conditional block.

class Rating...

```js
get voyageProfitFactor() {
  let result = 2;

  if (this.voyage.zone === "china") result += 1;
  if (this.voyage.zone === "east-indies") result += 1;
  result += this.voyageAndHistoryLengthFactor;
  return result;
}

get voyageAndHistoryLengthFactor() {
  let result = 0;
  if (this.voyage.zone === "china" && this.hasChinaHistory) {
    result += 3;
    if (this.history.length > 10) result += 1;
    if (this.voyage.length > 12) result += 1;
    if (this.voyage.length > 18) result -= 1;
  }
  else {
    if (this.history.length > 8) result += 1;
    if (this.voyage.length > 14) result -= 1;
  }
  return result;
}
```

A function name with an "And" in it is a pretty bad smell, but I'll let it sit while I apply the subclassing.

class Rating...

```js
get voyageAndHistoryLengthFactor() {
  let result = 0;
  if (this.history.length > 8) result += 1;
  if (this.voyage.length > 14) result -= 1;
  return result;
}
```

class ExperiencedChinaRating...

```js
get voyageAndHistoryLengthFactor() {
  let result = 0;
  result += 3;
  if (this.history.length > 10) result += 1;
  if (this.voyage.length > 12) result += 1;
  if (this.voyage.length > 18) result -= 1;
  return result;
}
```

That's formally the end of the refactoring -- I've separated the variant behavior out into the subclass. The superclass's logic is simpler to understand and work with, and I only need to deal with the variant case when I'm working on the subclass code.

Introducing a method purely for overriding by a subclass is a common thing to do when doing this kind of base-and-variation inheritance. But a crude method like this obscures what's going on. The "And" gives away that there are really two separate modifications -- so I separate them using Extract Function on the history length modification, both in the superclass and subclass.

class Rating...

```js
get voyageAndHistoryLengthFactor() {
  let result = 0;
  result += this.historyLengthFactor;
  if (this.voyage.length > 14) result -= 1;
  return result;
}
get historyLengthFactor() {
  return (this.history.length > 8) ? 1 : 0;
}
```

I do the same with the subclass:

class ExperiencedChinaRating...

```js
get voyageAndHistoryLengthFactor() {
  let result = 0;
  result += 3;
  result += this.historyLengthFactor;
  if (this.voyage.length > 12) result += 1;
  if (this.voyage.length > 18) result -= 1;
  return result;
}
get historyLengthFactor() {
  return (this.history.length > 10) ? 1 : 0;
}
```

I can then use Move Statements to Callers on the superclass case.

class Rating...

```js
get voyageProfitFactor() {
  let result = 2;
  if (this.voyage.zone === "china") result += 1;
  if (this.voyage.zone === "east-indies") result += 1;
  result += this.historyLengthFactor;
  result += this.voyageLengthFactor;
  return result;
}

get voyageLengthFactor() {
  return (this.voyage.length > 14) ? - 1: 0;
}
```

class ExperiencedChinaRating...

```js
get voyageLengthFactor() {
  let result = 0;
  result += 3;
  if (this.voyage.length > 12) result += 1;
  if (this.voyage.length > 18) result -= 1;
  return result;
}
```

Adding 3 points doesn't make sense as part of the voyage length factor -- it's better added to the overall result.

class ExperiencedChinaRating...

```js
get voyageProfitFactor() {
  return super.voyageProfitFactor + 3;
}

get voyageLengthFactor() {
  let result = 0;
  if (this.voyage.length > 12) result += 1;
  if (this.voyage.length > 18) result -= 1;
  return result;
}
```

At the end of the refactoring, I have the following code. First, the basic rating class which can ignore any complications of the experienced China case:

```js
class Rating {
  constructor(voyage, history) {
    this.voyage = voyage;
    this.history = history;
  }
  get value() {
    const vpf = this.voyageProfitFactor;
    const vr = this.voyageRisk;
    const chr = this.captainHistoryRisk;
    if (vpf * 3 > (vr + chr * 2)) return "A";
    else return "B";
  }
  get voyageRisk() {
    let result = 1;
    if (this.voyage.length > 4) result += 2;
    if (this.voyage.length > 8) result += this.voyage.length - 8;
    if (["china", "east-indies"].includes(this.voyage.zone)) result += 4;
    return Math.max(result, 0);
  }
  get captainHistoryRisk() {
    let result = 1;
    if (this.history.length < 5) result += 4;
    result += this.history.filter(v => v.profit < 0).length;
    return Math.max(result, 0);
  }
  get voyageProfitFactor() {
    let result = 2;
    if (this.voyage.zone === "china") result += 1;
    if (this.voyage.zone === "east-indies") result += 1;
    result += this.historyLengthFactor;
    result += this.voyageLengthFactor;
    return result;
  }
  get voyageLengthFactor() {
    return (this.voyage.length > 14) ? - 1: 0;
  }
  get historyLengthFactor() {
    return (this.history.length > 8) ? 1 : 0;
  }
}
```

The code for the experienced China case reads as a set of variations on the base:

```js
class ExperiencedChinaRating extends Rating {
  get captainHistoryRisk() {
    const result =  super.captainHistoryRisk - 2;
    return Math.max(result, 0);
  }
  get voyageLengthFactor() {
    let result = 0;
    if (this.voyage.length > 12) result += 1;
    if (this.voyage.length > 18) result -= 1;
    return result;
  }
  get historyLengthFactor() {
    return (this.history.length > 10) ? 1 : 0;
  }
  get voyageProfitFactor() {
    return super.voyageProfitFactor + 3;
  }
}
```

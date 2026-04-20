# Replace Derived Variable with Query

```
get discountedTotal() {return this._discountedTotal;}
set discount(aNumber) {
  const old = this._discount;
  this._discount = aNumber;
  this._discountedTotal += old - aNumber; 
}
```

```
get discountedTotal() {return this._baseTotal - this._discount;}
set discount(aNumber) {this._discount = aNumber;}
```

## Motivation

Remove a mutable variable that duplicates a calculation by replacing it with a query, so the value cannot fall out of sync with its source data. If the source data is immutable and the result can also be made immutable, keeping the derived variable is acceptable.

## Mechanics

- Identify all points of update for the variable. If necessary, use Split Variable to separate each point of update.

- Create a function that calculates the value of the variable.

- Use Introduce Assertion to assert that the variable and the calculation give the same result whenever the variable is used.

  If necessary, use Encapsulate Variable to provide a home for the assertion.

- Test.

- Replace any reader of the variable with a call to the new function.

- Test.

- Apply Remove Dead Code to the declaration and updates to the variable.

## Example

class ProductionPlan...

```
get production() {return this._production;}
applyAdjustment(anAdjustment) {
  this._adjustments.push(anAdjustment);
  this._production += anAdjustment.amount;
}
```

I see duplication of data here. When I apply an adjustment, I'm not just storing that adjustment but also using it to modify an accumulator. I can just calculate that value, without having to update it.

I test my hypothesis by using Introduce Assertion:

class ProductionPlan...

```
get production() {
  assert(this._production === this.calculatedProduction);
  return this._production;
}
```

```
get calculatedProduction() {
  return this._adjustments
    .reduce((sum, a) => sum + a.amount, 0);
}
```

With the assertion in place, I run my tests. If the assertion doesn't fail, I can replace returning the field with returning the calculation:

class ProductionPlan...

```
get production() {
  assert(this._production === this.calculatedProduction);
  return this.calculatedProduction;
}
```

Then Inline Function:

class ProductionPlan...

```
get production() {
  return this._adjustments
    .reduce((sum, a) => sum + a.amount, 0);
}
```

I clean up any references to the old variable with Remove Dead Code:

class ProductionPlan...

```
applyAdjustment(anAdjustment) {
  this._adjustments.push(anAdjustment);
}
```

## Example: More Than One Source

The above example is nice and easy because there's clearly a single source for the value of `production`. But sometimes, more than one element can combine in the accumulator.

class ProductionPlan...

```
constructor (production) {
  this._production = production;
  this._adjustments = [];
}
get production() {return this._production;}
applyAdjustment(anAdjustment) {
  this._adjustments.push(anAdjustment);
  this._production += anAdjustment.amount;
}
```

If I do the same Introduce Assertion that I did above, it will now fail for any case where the initial value of the production isn't zero.

But I can still replace the derived data. The only difference is that I must first apply Split Variable.

```
constructor (production) {
  this._initialProduction = production;
  this._productionAccumulator = 0;
  this._adjustments = [];
}
get production() {
  return this._initialProduction + this._productionAccumulator;
}
```

Now I can Introduce Assertion:

class ProductionPlan...

```
get production() {
  assert(this._productionAccumulator === this.calculatedProductionAccumulator);
  return this._initialProduction + this._productionAccumulator;
}
```

```
get calculatedProductionAccumulator() {
  return this._adjustments
    .reduce((sum, a) => sum + a.amount, 0);
}
```

And continue pretty much as before. I'd be inclined, however, to leave `totalProductionAjustments` as its own property, without inlining it.

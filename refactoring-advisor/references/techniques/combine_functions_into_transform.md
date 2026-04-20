# Combine Functions into Transform

```
function base(aReading) {...}
function taxableCharge(aReading) {...}
```

```
function enrichReading(argReading) {
  const aReading = _.cloneDeep(argReading);
  aReading.baseCharge = base(aReading);
  aReading.taxableCharge = taxableCharge(aReading);
  return aReading;
}
```

## Motivation

Gathers all derived-value calculations into a single transformation function that enriches the source data record with computed fields. Don't apply if the source data gets updated within the code -- a transform stores derived data in the new record, so mutations to the source cause inconsistencies; use Combine Functions into Class instead.

## Mechanics

- Create a transformation function that takes the record to be transformed and returns the same values.

  This will usually involve a deep copy. It is often worthwhile to write a test to ensure the transform does not alter the original record.

- Pick some logic and move its body into the transform to create a new field in the record. Change the client code to access the new field.

  If the logic is complex, use Extract Function first.

- Test.

- Repeat for the other relevant functions.

## Example

Monthly tea meter readings produce records like:

```js
reading = {customer: "ivan", quantity: 10, month: 5, year: 2017};
```

One calculation computes the base monetary charge:

client 1...

```js
const aReading = acquireReading();
const baseCharge = baseRate(aReading.month, aReading.year) * aReading.quantity;
```

Another computes the taxable amount (allowing some tea tax-free):

client 2...

```js
const aReading = acquireReading();
const base = (baseRate(aReading.month, aReading.year) * aReading.quantity);
const taxableCharge =  Math.max(0, base - taxThreshold(aReading.year));
```

These calculations are repeated in several places. Elsewhere, someone already extracted the base charge calculation:

client 3...

```js
const aReading = acquireReading();
const basicChargeAmount = calculateBaseCharge(aReading);

function calculateBaseCharge(aReading) {
  return  baseRate(aReading.month, aReading.year) * aReading.quantity;
}
```

Move all derivations into a transformation step. Begin by creating a transformation function that merely copies the input:

```js
function enrichReading(original) {
  const result = _.cloneDeep(original);
  return result;
}
```

Using `cloneDeep` from lodash for a deep copy. Name it "enrich" when producing the same thing with additional information; use "transform" when producing something different.

Pick a calculation to change. First, enrich the reading:

client 3...

```js
const rawReading = acquireReading();
const aReading = enrichReading(rawReading);
const basicChargeAmount = calculateBaseCharge(aReading);
```

Use Move Function on `calculateBaseCharge` to move it into the enrichment:

```js
function enrichReading(original) {
  const result = _.cloneDeep(original);
  result.baseCharge = calculateBaseCharge(result);
  return result;
}
```

Change the client to use the enriched field:

client 3...

```js
const rawReading = acquireReading();
const aReading = enrichReading(rawReading);
const basicChargeAmount = aReading.baseCharge;
```

Once all calls to `calculateBaseCharge` are migrated, you can nest it inside `enrichReading` to make it clear that clients needing the base charge should use the enriched record.

Add a test to verify the original reading isn't changed:

```js
it('check reading unchanged', function() {
  const baseReading = {customer: "ivan", quantity: 15, month: 5, year: 2017};
  const oracle = _.cloneDeep(baseReading);
  enrichReading(baseReading);
  assert.deepEqual(baseReading, oracle);
});
```

Change client 1 to also use the enriched field:

client 1...

```js
const rawReading = acquireReading();
const aReading = enrichReading(rawReading);
const baseCharge = aReading.baseCharge;
```

(There is a good chance you can then use Inline Variable on `baseCharge`.)

Now turn to the taxable amount. First add the transformation function:

```js
const rawReading = acquireReading();
const aReading = enrichReading(rawReading);
const base = (baseRate(aReading.month, aReading.year) * aReading.quantity);
const taxableCharge =  Math.max(0, base - taxThreshold(aReading.year));
```

Replace the base charge calculation with the new field:

```js
const rawReading = acquireReading();
const aReading = enrichReading(rawReading);
const base = aReading.baseCharge;
const taxableCharge =  Math.max(0, base - taxThreshold(aReading.year));
```

Apply Inline Variable:

```js
const rawReading = acquireReading();
const aReading = enrichReading(rawReading);
const taxableCharge =  Math.max(0, aReading.baseCharge - taxThreshold(aReading.year));
```

Move that computation into the transformer:

```js
function enrichReading(original) {
  const result = _.cloneDeep(original);
  result.baseCharge = calculateBaseCharge(result);
  result.taxableCharge = Math.max(0, result.baseCharge - taxThreshold(result.year));
  return result;
}
```

Modify the original code to use the new field:

```js
const rawReading = acquireReading();
const aReading = enrichReading(rawReading);
const taxableCharge = aReading.taxableCharge;
```

One big problem with enriched readings: if a client changes a data value (e.g., the quantity field), the data becomes inconsistent. In JavaScript, the best option is to use Combine Functions into Class instead. In languages with immutable data structures, this isn't a problem, so transforms are more common there. Transforms are also fine when the data appears in a read-only context, such as deriving data to display on a web page.

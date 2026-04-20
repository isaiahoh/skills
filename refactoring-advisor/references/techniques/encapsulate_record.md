# Encapsulate Record

```
organization = {name: "Acme Gooseberries", country: "GB"};
```

```
class Organization {
  constructor(data) {
    this._name = data.name;
    this._country = data.country;
  }
  get name()    {return this._name;}
  set name(arg) {this._name = arg;}
  get country()    {return this._country;}
  set country(arg) {this._country = arg;}
}
```

## Motivation

Replaces a raw record (struct, hash, dict) with a class that hides internal storage behind accessor methods, so callers don't depend on the record's field layout.

## Mechanics

- Use Encapsulate Variable on the variable holding the record.

  Give the functions that encapsulate the record names that are easily searchable.

- Replace the content of the variable with a simple class that wraps the record. Define an accessor inside this class that returns the raw record. Modify the functions that encapsulate the variable to use this accessor.

- Test.
- Provide new functions that return the object rather than the raw record.

- For each user of the record, replace its use of a function that returns the record with a function that returns the object. Use an accessor on the object to get at the field data, creating that accessor if needed. Test after each change.

  If it's a complex record, such as one with a nested structure, focus on clients that update the data first. Consider returning a copy or read-only proxy of the data for clients that only read the data.

- Remove the class's raw data accessor and the easily searchable functions that returned the raw record.

- Test.
- If the fields of the record are themselves structures, consider using Encapsulate Record and Encapsulate Collection recursively.

## Example

I'll start with a constant that is widely used across a program.

```js
const organization = {name: "Acme Gooseberries", country: "GB"};
```

This is a JavaScript object used as a record structure, with accesses like this:

```js
result += `<h1>${organization.name}</h1>`;
```

and

```js
organization.name = newName;
```

The first step is Encapsulate Variable.

```js
function getRawDataOfOrganization() {return organization;}
```

example reader...

```js
result += `<h1>${getRawDataOfOrganization().name}</h1>`;
```

example writer...

```js
getRawDataOfOrganization().name = newName;
```

I gave the getter a name deliberately chosen to be both ugly and easy to search for, because I intend its life to be short.

Encapsulating a record means going deeper than just the variable itself; I want to control how it's manipulated. I do this by replacing the record with a class.

class Organization...

```js
class Organization {
  constructor(data) {
    this._data = data;
  }
}
```

top level

```js
const organization = new Organization({name: "Acme Gooseberries", country: "GB"});
```

```js
function getRawDataOfOrganization() {return organization._data;}
function getOrganization() {return organization;}
```

Now that I have an object in place, I start looking at the users of the record. Any one that updates the record gets replaced with a setter.

class Organization...

```js
set name(aString) {this._data.name = aString;}
```

client...

```js
getOrganization().name = newName;
```

Similarly, I replace any readers with the appropriate getter.

class Organization...

```js
get name()    {return this._data.name;}
```

client...

```js
result += `<h1>${getOrganization().name}</h1>`;
```

After I've done that, I can remove the ugly-named function.

```js
function getRawDataOfOrganization() {return organization._data;}
function getOrganization() {return organization;}
```

I'd also fold the `_data` field directly into the object.

```js
class Organization {
  constructor(data) {
    this._name = data.name;
    this._country = data.country;
  }
  get name()    {return this._name;}
  set name(aString) {this._name = aString;}
  get country()    {return this._country;}
  set country(aCountryCode) {this._country = aCountryCode;}
}
```

This breaks the link to the input data record, which is useful if a reference to it runs around and would break encapsulation. If you don't fold the data into individual fields, copy `_data` when you assign it.

## Example: Encapsulating a Nested Record

The core refactoring steps still apply to deeply nested data, and you must be equally careful with updates, but there are options around reads.

Here is some nested data: a collection of customers kept in a hashmap indexed by customer ID.

```js
"1920": {
  name: "martin",
  id: "1920",
  usages: {
    "2016": {
      "1": 50,
      "2": 55,
      // remaining months of the year
    },
    "2015": {
      "1": 70,
      "2": 63,
      // remaining months of the year
    }
  }
},
"38673": {
  name: "neal",
  id: "38673",
  // more customers in a similar form
```

With nested data, reads and writes dig into the structure.

sample update...

```js
customerData[customerID].usages[year][month] = amount;
```

sample read...

```js
function compareUsage (customerID, laterYear, month) {
  const later   = customerData[customerID].usages[laterYear][month];
  const earlier = customerData[customerID].usages[laterYear - 1][month];
  return {laterAmount: later, change: later - earlier};
}
```

To encapsulate, start with Encapsulate Variable.

```js
function getRawDataOfCustomers()    {return customerData;}
function setRawDataOfCustomers(arg) {customerData = arg;}
```

sample update...

```js
getRawDataOfCustomers()[customerID].usages[year][month] = amount;
```

sample read...

```js
function compareUsage (customerID, laterYear, month) {
  const later   = getRawDataOfCustomers()[customerID].usages[laterYear][month];
  const earlier = getRawDataOfCustomers()[customerID].usages[laterYear - 1][month];
  return {laterAmount: later, change: later - earlier};
}
```

Then make a class for the overall data structure.

```js
class CustomerData {
  constructor(data) {
    this._data = data;
  }
}
```

top level...

```js
function getCustomerData() {return customerData;}
function getRawDataOfCustomers()    {return customerData._data;}
function setRawDataOfCustomers(arg) {customerData = new CustomerData(arg);}
```

The most important area to deal with is the updates. For updates that dig into the structure, use Extract Function to pull out the code, then Move Function to move it into the new class.

sample update...

```js
setUsage(customerID, year, month, amount);
```

top level...

```js
function setUsage(customerID, year, month, amount) {
  getRawDataOfCustomers()[customerID].usages[year][month] = amount;
}
```

Then move it into the new customer data class.

sample update...

```js
getCustomerData().setUsage(customerID, year, month, amount);
```

class CustomerData...

```js
setUsage(customerID, year, month, amount) {
  this._data[customerID].usages[year][month] = amount;
}
```

When working with a big data structure, concentrate on the updates. Getting them visible and gathered in a single place is the most important part of the encapsulation.

To verify you've caught all updates, modify `getRawDataOfCustomers` to return a deep copy of the data; if test coverage is good, a test should break if you missed a modification.

top level...

```js
function getCustomerData() {return customerData;}
function getRawDataOfCustomers()    {return customerData.rawData;}
function setRawDataOfCustomers(arg) {customerData = new CustomerData(arg);}
```

class CustomerData...

```js
get rawData() {
  return _.cloneDeep(this._data);
}
```

Another approach is to return a read-only proxy that raises an exception on writes, or take a copy and recursively freeze it.

For readers, there are a few options:

**Option 1:** Extract all reads into functions and move them into the customer data class.

class CustomerData...

```js
usage(customerID, year, month) {
  return this._data[customerID].usages[year][month];
}
```

top level...

```js
function compareUsage (customerID, laterYear, month) {
  const later   = getCustomerData().usage(customerID, laterYear, month);
  const earlier = getCustomerData().usage(customerID, laterYear - 1, month);
  return {laterAmount: later, change: later - earlier};
}
```

This gives `customerData` an explicit API that captures all uses. But it can mean a lot of code for many special cases.

**Option 2:** Hand out a copy of the underlying data using `rawData`, so clients can use standard list-and-hash affordances without risking mutation of the original.

class CustomerData...

```js
get rawData() {
  return _.cloneDeep(this._data);
}
```

top level...

```js
function compareUsage (customerID, laterYear, month) {
  const later   = getCustomerData().rawData[customerID].usages[laterYear][month];
  const earlier = getCustomerData().rawData[customerID].usages[laterYear - 1][month];
  return {laterAmount: later, change: later - earlier};
}
```

The downside is the cost of copying a large structure (measure before worrying) and potential confusion if clients expect modifications to the copy to affect the original.

**Option 3:** Apply Encapsulate Record recursively--turn customer records into classes, apply Encapsulate Collection to usages, create a usage class. This offers the most control but can be a lot of effort for a large data structure.

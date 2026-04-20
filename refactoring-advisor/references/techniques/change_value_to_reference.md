# Change Value to Reference

```
let customer = new Customer(customerData);
```

```
let customer = customerRepository.get(customerData.id);
```

## Motivation

Replace multiple copies of the same logical data with a single shared reference (via a repository) so that updates are consistent and never missed.

## Mechanics

- Create a repository for instances of the related object (if one isn't already present).

- Ensure the constructor has a way of looking up the correct instance of the related object.

- Change the constructors for the host object to use the repository to obtain the related object. Test after each change.

## Example

I'll begin with a class that represents orders, which I might create from an incoming JSON document. Part of the order data is a customer ID from which I'm creating a customer object.

class Order...

```
constructor(data) {
  this._number = data.number;
  this._customer = new Customer(data.customer);
  // load other data
}
get customer() {return this._customer;}
```

class Customer...

```
constructor(id) {
  this._id = id;
}
get id() {return this._id;}
```

The customer object I create this way is a value. If I have five orders that refer to the customer ID of 123, I'll have five separate customer objects. Any change I make to one of them will not be reflected in the others. Should I want to enrich the customer objects, perhaps by gathering data from a customer service, I'd have to update all five customers with the same data. Having duplicate objects like this is confusing and particularly awkward if the customer object is mutable, which can lead to inconsistencies.

If I want to use the same customer object each time, I'll need a place to store it. For a simple case I use a repository object.

```
let _repositoryData;

export function initialize() {
  _repositoryData = {};
  _repositoryData.customers = new Map();
}

export function registerCustomer(id) {
  if (! _repositoryData.customers.has(id))
    _repositoryData.customers.set(id, new Customer(id));
  return findCustomer(id);
}

export function findCustomer(id) {
  return _repositoryData.customers.get(id);
}
```

The repository allows me to register customer objects with an ID and ensures I only create one customer object with the same ID. With this in place, I can change the order's constructor to use it.

Often, when doing this refactoring, the repository already exists, so I can just use it.

The next step is to figure out how the constructor for the order can obtain the correct customer object. In this case it's easy, since the customer's ID is present in the input data stream.

class Order...

```
constructor(data) {
  this._number = data.number;
  this._customer = registerCustomer(data.customer);
  // load other data
}
get customer() {return this._customer;}
```

Now, any changes I make to the customer of one order will be synchronized across all the orders sharing the same customer.

For this example, I created a new customer object with the first order that referenced it. Another common approach is to get a list of customers, populate the repository with them, and then link to them as I read the orders. In that case, an order that contains a customer ID not in the repository would indicate an error.

One problem with this code is that the constructor body is coupled to the global repository. Globals should be treated with care -- if I'm concerned about it, I can pass the repository as a parameter to the constructor.

# Extract Superclass

```
class Department {
  get totalAnnualCost() {...}
  get name() {...}
  get headCount() {...}
}

class Employee {
  get annualCost() {...}
  get name() {...}
  get id() {...}
}
```

```
class Party {
  get name() {...}
  get annualCost() {...}
}

class Department extends Party {
  get annualCost() {...}
  get headCount() {...}
}

class Employee extends Party {
  get annualCost() {...}
  get id() {...}
}
```

## Motivation

Pulls common data and behavior from two similar classes into a new superclass using Pull Up Field and Pull Up Method.

## Mechanics

- Create an empty superclass. Make the original classes its subclasses.

  If needed, use Change Function Declaration on the constructors.

- Test.
- One by one, use Pull Up Constructor Body, Pull Up Method, and Pull Up Field to move common elements to the superclass.

- Examine remaining methods on the subclasses. See if there are common parts. If so, use Extract Function followed by Pull Up Method.

- Check clients of the original classes. Consider adjusting them to use the superclass interface.

## Example

I'm pondering these two classes -- they share some common functionality: their name and the notions of annual and monthly costs:

```
class Employee {
  constructor(name, id, monthlyCost) {
    this._id = id;
    this._name = name;
    this._monthlyCost = monthlyCost;
  }
  get monthlyCost() {return this._monthlyCost;}
  get name() {return this._name;}
  get id() {return this._id;}
  
  get annualCost() {
    return this.monthlyCost * 12;
  }
}


class Department {
  constructor(name, staff){
    this._name = name;
    this._staff = staff;
  }
  get staff() {return this._staff.slice();}
  get name() {return this._name;}

  get totalMonthlyCost() {
    return this.staff
      .map(e => e.monthlyCost)
      .reduce((sum, cost) => sum + cost);
  }
  get headCount() {
    return this.staff.length;
  }
  get totalAnnualCost() {
    return this.totalMonthlyCost * 12;
  }
}
```

I begin by creating an empty superclass and letting them both extend from it.

```
class Party {}


class Employee extends Party {
  constructor(name, id, monthlyCost) {
    super();
    this._id = id;
    this._name = name;
    this._monthlyCost = monthlyCost;
  }
  // rest of class...


class Department extends Party {
  constructor(name, staff){
    super();
    this._name = name;
    this._staff = staff;
  }
  // rest of class...
```

I start with the data, using Pull Up Field to pull up the name.

class Party...

```
constructor(name){
  this._name = name;
}
```

class Employee...

```
constructor(name, id, monthlyCost) {
  super(name);
  this._id = id;
  this._monthlyCost = monthlyCost;
}
```

class Department...

```
constructor(name, staff){
  super(name);
  this._staff = staff;
}
```

As I get data up to the superclass, I can also apply Pull Up Method on associated methods. First, the name:

class Party...

```
get name() {return this._name;}
```

I delete `get name()` from both `Employee` and `Department`.

I have two methods with similar bodies.

class Employee...

```
get annualCost() {
  return this.monthlyCost * 12;
}
```

class Department...

```
get totalAnnualCost() {
  return this.totalMonthlyCost * 12;
}
```

The methods they use, `monthlyCost` and `totalMonthlyCost`, have different names and different bodies -- but do they represent the same intent? If so, I use Change Function Declaration to unify their names.

class Department...

```
get totalAnnualCost() {
  return this.monthlyCost * 12;
}

get monthlyCost() { ... }
```

I then do a similar renaming to the annual costs:

class Department...

```
get annualCost() {
  return this.monthlyCost * 12;
}
```

I can now apply Pull Up Method to the annual cost methods.

class Party...

```
get annualCost() {
  return this.monthlyCost * 12;
}
```

I delete `get annualCost()` from both `Employee` and `Department`.

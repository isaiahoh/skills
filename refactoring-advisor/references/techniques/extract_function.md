# Extract Function

```
function printOwing(invoice) {
  printBanner();
  let outstanding  = calculateOutstanding();

  //print details
  console.log(`name: ${invoice.customer}`);
  console.log(`amount: ${outstanding}`);
}
```

```
function printOwing(invoice) {
  printBanner();
  let outstanding  = calculateOutstanding();
  printDetails(outstanding);

  function printDetails(outstanding) {
    console.log(`name: ${invoice.customer}`);
    console.log(`amount: ${outstanding}`);
  }
}
```

## Motivation

Extracts a code fragment into its own function named after what it does, so the intent is immediately clear without reading the implementation.

## Mechanics

- Create a new function, and name it after the intent of the function (name it by _what_ it does, not by _how_ it does it).

  If you can't come up with a more meaningful name than the code itself, that's a sign you shouldn't extract. It's OK to extract, try to work with it, realize it isn't helping, and inline it back.

  If the language supports nested functions, nest the extracted function inside the source function to reduce out-of-scope variables. You can always use Move Function later.

- Copy the extracted code from the source function into the new target function.

- Scan the extracted code for references to any variables that are local in scope to the source function and will not be in scope for the extracted function. Pass them as parameters.

  If a variable is only used inside the extracted code but declared outside, move the declaration into the extracted code.

  Any variables that are assigned to need more care if they are passed by value. If there's only one, try to treat the extracted code as a query and assign the result to the variable concerned.

  If too many local variables are being assigned by the extracted code, abandon the extraction. Consider Split Variable or Replace Temp with Query to simplify variable usage and revisit the extraction later.

- Compile after all variables are dealt with.

- Replace the extracted code in the source function with a call to the target function.

- Test.

- Look for other code that's the same or similar to the code just extracted, and consider using Replace Inline Code with Function Call to call the new function.

## Example: No Variables Out of Scope

In the simplest case, Extract Function is trivially easy.

```js
function printOwing(invoice) {
  let outstanding = 0;

  console.log("***********************");
  console.log("**** Customer Owes ****");
  console.log("***********************");

  // calculate outstanding
  for (const o of invoice.orders) {
    outstanding += o.amount;
  }

  // record due date
  const today = Clock.today;
  invoice.dueDate = new Date(today.getFullYear(), today.getMonth(), today.getDate() + 30);

  //print details
  console.log(`name: ${invoice.customer}`);
  console.log(`amount: ${outstanding}`);
  console.log(`due: ${invoice.dueDate.toLocaleDateString()}`);
}
```

It's easy to extract the code that prints the banner. Cut, paste, and put in a call:

```js
function printOwing(invoice) {
  let outstanding = 0;

  printBanner();

  // calculate outstanding
  for (const o of invoice.orders) {
    outstanding += o.amount;
  }

  // record due date
  const today = Clock.today;
  invoice.dueDate = new Date(today.getFullYear(), today.getMonth(), today.getDate() + 30);


  //print details
  console.log(`name: ${invoice.customer}`);
  console.log(`amount: ${outstanding}`);
  console.log(`due: ${invoice.dueDate.toLocaleDateString()}`);
}
function printBanner() {
  console.log("***********************");
  console.log("**** Customer Owes ****");
  console.log("***********************");
}
```

Similarly, extract the printing of details:

```js
function printOwing(invoice) {
  let outstanding = 0;

  printBanner();

  // calculate outstanding
  for (const o of invoice.orders) {
    outstanding += o.amount;
  }

  // record due date
  const today = Clock.today;
  invoice.dueDate = new Date(today.getFullYear(), today.getMonth(), today.getDate() + 30);

  printDetails();

  function printDetails() {
    console.log(`name: ${invoice.customer}`);
    console.log(`amount: ${outstanding}`);
    console.log(`due: ${invoice.dueDate.toLocaleDateString()}`);
  }
```

Here `printDetails` is nested inside `printOwing`, so it can access all the variables defined in `printOwing`. If the language doesn't allow nested functions, you must extract to the top level and deal with variables that exist only in the scope of the source function -- the arguments and the temporaries.

## Example: Using Local Variables

The easiest case with local variables is when they are used but not reassigned. Just pass them in as parameters.

```js
function printOwing(invoice) {
  let outstanding = 0;

  printBanner();

  // calculate outstanding
  for (const o of invoice.orders) {
    outstanding += o.amount;
  }

  // record due date
  const today = Clock.today;
  invoice.dueDate = new Date(today.getFullYear(), today.getMonth(), today.getDate() + 30);

  //print details
  console.log(`name: ${invoice.customer}`);
  console.log(`amount: ${outstanding}`);
  console.log(`due: ${invoice.dueDate.toLocaleDateString()}`);
}
```

Extract the printing of details passing two parameters:

```js
function printOwing(invoice) {
  let outstanding = 0;

  printBanner();

  // calculate outstanding
  for (const o of invoice.orders) {
    outstanding += o.amount;
  }

  // record due date
  const today = Clock.today;
  invoice.dueDate = new Date(today.getFullYear(), today.getMonth(), today.getDate() + 30);

  printDetails(invoice, outstanding);
}
function printDetails(invoice, outstanding) {
  console.log(`name: ${invoice.customer}`);
  console.log(`amount: ${outstanding}`);
  console.log(`due: ${invoice.dueDate.toLocaleDateString()}`);
}
```

The same is true if the local variable is a structure (such as an array, record, or object) and you modify that structure. So you can similarly extract the setting of the due date:

```js
function printOwing(invoice) {
  let outstanding = 0;

  printBanner();

  // calculate outstanding
  for (const o of invoice.orders) {
    outstanding += o.amount;
  }

  recordDueDate(invoice);
  printDetails(invoice, outstanding);
}
function recordDueDate(invoice) {
  const today = Clock.today;
  invoice.dueDate = new Date(today.getFullYear(), today.getMonth(), today.getDate() + 30);
}
```

## Example: Reassigning a Local Variable

Assignment to local variables is where things get complicated. Only temps matter here -- if you see an assignment to a parameter, use Split Variable first to turn it into a temp.

For temps that are assigned to, the simpler case is where the variable is only used within the extracted code -- it just lives inside the extraction. Use Slide Statements to get all the variable manipulation together if needed.

The more awkward case is where the variable is used outside the extracted function. You need to return the new value.

```js
function printOwing(invoice) {
  let outstanding = 0;

  printBanner();

  // calculate outstanding
  for (const o of invoice.orders) {
    outstanding += o.amount;
  }

  recordDueDate(invoice);
  printDetails(invoice, outstanding);
}
```

First, slide the declaration next to its use.

```js
function printOwing(invoice) {
  printBanner();

  // calculate outstanding
  let outstanding = 0;
  for (const o of invoice.orders) {
    outstanding += o.amount;
  }

  recordDueDate(invoice);
  printDetails(invoice, outstanding);
}
```

Then copy the code into a target function.

```js
function printOwing(invoice) {
  printBanner();

  // calculate outstanding
  let outstanding = 0;
  for (const o of invoice.orders) {
    outstanding += o.amount;
  }

  recordDueDate(invoice);
  printDetails(invoice, outstanding);
}
function calculateOutstanding(invoice) {
  let outstanding = 0;
  for (const o of invoice.orders) {
    outstanding += o.amount;
  }
  return outstanding;
}
```

Since the declaration of `outstanding` moved into the extracted code, it doesn't need to be passed as a parameter. It is the only reassigned variable, so it can be returned.

Replace the original code with a call. Since a value is returned, store it in the original variable.

```js
function printOwing(invoice) {
  printBanner();
  let outstanding = calculateOutstanding(invoice);
  recordDueDate(invoice);
  printDetails(invoice, outstanding);
}
function calculateOutstanding(invoice) {
  let outstanding = 0;
  for (const o of invoice.orders) {
    outstanding += o.amount;
  }
  return outstanding;
}
```

Rename the return value to follow coding style, and change the original `outstanding` into a `const`.

```js
function printOwing(invoice) {
  printBanner();
  const outstanding = calculateOutstanding(invoice);
  recordDueDate(invoice);
  printDetails(invoice, outstanding);
}
function calculateOutstanding(invoice) {
  let result = 0;
  for (const o of invoice.orders) {
    result += o.amount;
  }
  return result;
}
```

If more than one variable needs to be returned, prefer picking different code to extract so each function returns one value. Use Replace Temp with Query and Split Variable to simplify the temporaries first.

# Split Phase

```
const orderData = orderString.split(/\s+/);
const productPrice = priceList[orderData[0].split("-")[1]];
const orderPrice = parseInt(orderData[1]) * productPrice;
```

```
const orderRecord = parseOrder(order);
const orderPrice = price(orderRecord, priceList);

function parseOrder(aString) {
  const values =  aString.split(/\s+/);
  return ({
    productID: values[0].split("-")[1],
    quantity: parseInt(values[1]),
  });
}
function price(order, priceList) {
  return order.quantity * priceList[order.productID];
}
```

## Motivation

Splits code that handles two different concerns into two sequential phases connected by an intermediate data structure, so each phase can be understood and modified independently.

## Mechanics

- Extract the second phase code into its own function.

- Test.

- Introduce an intermediate data structure as an additional argument to the extracted function.

- Test.

- Examine each parameter of the extracted second phase. If it is used by first phase, move it to the intermediate data structure. Test after each move.

  Sometimes a parameter should not be used by the second phase. In this case, extract the results of each usage into a field of the intermediate data structure and use Move Statements to Callers on the line that populates it.

- Apply Extract Function on the first-phase code, returning the intermediate data structure.

  It's also reasonable to extract the first phase into a transformer object.

## Example

Code to price an order:

```js
function priceOrder(product, quantity, shippingMethod) {
  const basePrice = product.basePrice * quantity;
  const discount = Math.max(quantity - product.discountThreshold, 0)
          * product.basePrice * product.discountRate;
  const shippingPerCase = (basePrice > shippingMethod.discountThreshold)
          ? shippingMethod.discountedFee : shippingMethod.feePerCase;
  const shippingCost = quantity * shippingPerCase;
  const price =  basePrice - discount + shippingCost;
  return price;
}
```

There are two phases here: the first couple of lines use product information to calculate the product-oriented price, while the later code uses shipping information for the shipping cost. If pricing and shipping calculations become more complex but work independently, splitting them is valuable.

Begin by applying Extract Function to the shipping calculation:

```js
function priceOrder(product, quantity, shippingMethod) {
  const basePrice = product.basePrice * quantity;
  const discount = Math.max(quantity - product.discountThreshold, 0)
          * product.basePrice * product.discountRate;
  const price =  applyShipping(basePrice, shippingMethod, quantity, discount);
  return price;
}
function applyShipping(basePrice, shippingMethod, quantity, discount) {
  const shippingPerCase = (basePrice > shippingMethod.discountThreshold)
          ? shippingMethod.discountedFee : shippingMethod.feePerCase;
  const shippingCost = quantity * shippingPerCase;
  const price =  basePrice - discount + shippingCost;
  return price;
}
```

Pass in all the data the second phase needs as individual parameters. They'll be whittled down later.

Introduce the intermediate data structure:

```js
function priceOrder(product, quantity, shippingMethod) {
  const basePrice = product.basePrice * quantity;
  const discount = Math.max(quantity - product.discountThreshold, 0)
          * product.basePrice * product.discountRate;
  const priceData = {};
  const price =  applyShipping(priceData, basePrice, shippingMethod, quantity, discount);
  return price;
}

function applyShipping(priceData, basePrice, shippingMethod, quantity, discount) {
  const shippingPerCase = (basePrice > shippingMethod.discountThreshold)
          ? shippingMethod.discountedFee : shippingMethod.feePerCase;
  const shippingCost = quantity * shippingPerCase;
  const price =  basePrice - discount + shippingCost;
  return price;
}
```

Move `basePrice` into the intermediate data structure, removing it from the parameter list:

```js
function priceOrder(product, quantity, shippingMethod) {
  const basePrice = product.basePrice * quantity;
  const discount = Math.max(quantity - product.discountThreshold, 0)
          * product.basePrice * product.discountRate;
  const priceData = {basePrice: basePrice};
  const price =  applyShipping(priceData, basePrice, shippingMethod, quantity, discount);
  return price;
}
function applyShipping(priceData, basePrice, shippingMethod, quantity, discount) {
  const shippingPerCase = (priceData.basePrice > shippingMethod.discountThreshold)
          ? shippingMethod.discountedFee : shippingMethod.feePerCase;
  const shippingCost = quantity * shippingPerCase;
  const price =  priceData.basePrice - discount + shippingCost;
  return price;
}
```

Leave `shippingMethod` as is -- it isn't used by the first-phase code.

Move `quantity` to the intermediate data structure. Even though it's used by the first phase but not created by it, prefer to move as much as possible to the intermediate structure:

```js
function priceOrder(product, quantity, shippingMethod) {
  const basePrice = product.basePrice * quantity;
  const discount = Math.max(quantity - product.discountThreshold, 0)
          * product.basePrice * product.discountRate;
  const priceData = {basePrice: basePrice, quantity: quantity};
  const price =  applyShipping(priceData, shippingMethod, quantity, discount);
  return price;
}
function applyShipping(priceData, shippingMethod, quantity, discount) {
  const shippingPerCase = (priceData.basePrice > shippingMethod.discountThreshold)
          ? shippingMethod.discountedFee : shippingMethod.feePerCase;
  const shippingCost = priceData.quantity * shippingPerCase;
  const price =  priceData.basePrice - discount + shippingCost;
  return price;
}
```

Do the same with `discount`:

```js
function priceOrder(product, quantity, shippingMethod) {
  const basePrice = product.basePrice * quantity;
  const discount = Math.max(quantity - product.discountThreshold, 0)
          * product.basePrice * product.discountRate;
  const priceData = {basePrice: basePrice, quantity: quantity, discount:discount};
  const price =  applyShipping(priceData, shippingMethod, discount);
  return price;
}
function applyShipping(priceData, shippingMethod, discount) {
  const shippingPerCase = (priceData.basePrice > shippingMethod.discountThreshold)
          ? shippingMethod.discountedFee : shippingMethod.feePerCase;
  const shippingCost = priceData.quantity * shippingPerCase;
  const price =  priceData.basePrice - priceData.discount + shippingCost;
  return price;
}
```

With the intermediate data structure fully formed, extract the first-phase code into its own function:

```js
function priceOrder(product, quantity, shippingMethod) {
  const priceData = calculatePricingData(product, quantity);
  const price =  applyShipping(priceData, shippingMethod);
  return price;
}
function calculatePricingData(product, quantity) {
  const basePrice = product.basePrice * quantity;
  const discount = Math.max(quantity - product.discountThreshold, 0)
          * product.basePrice * product.discountRate;
  return {basePrice: basePrice, quantity: quantity, discount:discount};
}
function applyShipping(priceData, shippingMethod) {
  const shippingPerCase = (priceData.basePrice > shippingMethod.discountThreshold)
          ? shippingMethod.discountedFee : shippingMethod.feePerCase;
  const shippingCost = priceData.quantity * shippingPerCase;
  const price =  priceData.basePrice - priceData.discount + shippingCost;
  return price;
}
```

Tidy out the final constants:

```js
function priceOrder(product, quantity, shippingMethod) {
  const priceData = calculatePricingData(product, quantity);
  return applyShipping(priceData, shippingMethod);
}

function calculatePricingData(product, quantity) {
  const basePrice = product.basePrice * quantity;
  const discount = Math.max(quantity - product.discountThreshold, 0)
          * product.basePrice * product.discountRate;
  return {basePrice: basePrice, quantity: quantity, discount:discount};
}
function applyShipping(priceData, shippingMethod) {
  const shippingPerCase = (priceData.basePrice > shippingMethod.discountThreshold)
          ? shippingMethod.discountedFee : shippingMethod.feePerCase;
  const shippingCost = priceData.quantity * shippingPerCase;
  return priceData.basePrice - priceData.discount + shippingCost;
}
```

## Example: Splitting a Command-Line Program (Java)

A Java program that counts orders in a JSON file:

```java
public static void main(String[] args) {
  try {
    if (args.length == 0) throw new RuntimeException("must supply a filename");
    String filename = args[args.length - 1];
    File input = Paths.get(filename).toFile();
    ObjectMapper mapper = new ObjectMapper();
    Order[] orders = mapper.readValue(input, Order[].class);
    if (Stream.of(args).anyMatch(arg -> "-r".equals(arg)))
      System.out.println(Stream.of(orders).filter(o -> "ready".equals(o.status)).count());
    else
      System.out.println(orders.length);
  } catch (Exception e) {
    System.err.println(e);
    System.exit(1);
  }
}
```

This code does two things: reads the list of orders and counts the appropriate ones, and reads command-line arguments to figure out what they mean. A first phase would parse the arguments; the second would process the data.

First, make testing easier by separating the slow command-line environment from the logic. Use Extract Function on all the main processing:

```java
public static void main(String[] args) {
  try {
    run(args);
  } catch (Exception e) {
    System.err.println(e);
    System.exit(1);
  }
}

static void run(String[] args) throws IOException {
  if (args.length == 0) throw new RuntimeException("must supply a filename");
  String filename = args[args.length - 1];
  File input = Paths.get(filename).toFile();
  ObjectMapper mapper = new ObjectMapper();
  Order[] orders = mapper.readValue(input, Order[].class);
  if (Stream.of(args).anyMatch(arg -> "-r".equals(arg)))
    System.out.println(Stream.of(orders).filter(o -> "ready".equals(o.status)).count());
  else
    System.out.println(orders.length);
}
```

Use Move Statements to Callers on `System.out` to make `run` return a value that can be tested with JUnit:

```java
public static void main(String[] args) {
  try {
    System.out.println(run(args));
  } catch (Exception e) {
    System.err.println(e);
    System.exit(1);
  }
}

static long run(String[] args) throws IOException {
  if (args.length == 0) throw new RuntimeException("must supply a filename");
  String filename = args[args.length - 1];
  File input = Paths.get(filename).toFile();
  ObjectMapper mapper = new ObjectMapper();
  Order[] orders = mapper.readValue(input, Order[].class);
  if (Stream.of(args).anyMatch(arg -> "-r".equals(arg)))
    return Stream.of(orders).filter(o -> "ready".equals(o.status)).count();
  else
    return orders.length;
}
```

Making `main` humble by removing logic makes testing easier and refactoring faster. Now extract the second-phase processing:

```java
static long run(String[] args) throws IOException {
  if (args.length == 0) throw new RuntimeException("must supply a filename");
  String filename = args[args.length - 1];
  return countOrders(args, filename);
}

private static long countOrders(String[] args, String filename) throws IOException {
  File input = Paths.get(filename).toFile();
  ObjectMapper mapper = new ObjectMapper();
  Order[] orders = mapper.readValue(input, Order[].class);
  if (Stream.of(args).anyMatch(arg -> "-r".equals(arg)))
    return Stream.of(orders).filter(o -> "ready".equals(o.status)).count();
  else
    return orders.length;
}
```

Add the intermediate data structure:

```java
static long run(String[] args) throws IOException {
  if (args.length == 0) throw new RuntimeException("must supply a filename");
  CommandLine commandLine = new CommandLine();
  String filename = args[args.length - 1];
  return countOrders(commandLine, args, filename);
}

private static long countOrders(CommandLine commandLine, String[] args, String filename) throws IOException {
  File input = Paths.get(filename).toFile();
  ObjectMapper mapper = new ObjectMapper();
  Order[] orders = mapper.readValue(input, Order[].class);
  if (Stream.of(args).anyMatch(arg -> "-r".equals(arg)))
    return Stream.of(orders).filter(o -> "ready".equals(o.status)).count();
  else
    return orders.length;
}
private static class CommandLine {}
```

The goal is to separate all use of `args` and isolate it to the first phase. Apply Extract Variable to the conditional that uses `args`:

```java
private static long countOrders(CommandLine commandLine, String[] args, String filename) throws IOException {
  File input = Paths.get(filename).toFile();
  ObjectMapper mapper = new ObjectMapper();
  Order[] orders = mapper.readValue(input, Order[].class);
  boolean onlyCountReady = Stream.of(args).anyMatch(arg -> "-r".equals(arg));
  if (onlyCountReady)
    return Stream.of(orders).filter(o -> "ready".equals(o.status)).count();
  else
    return orders.length;
}
private static class CommandLine {}
```

Move it into the intermediate data structure, then use Move Statements to Callers to populate it in the first phase:

```java
static long run(String[] args) throws IOException {
  if (args.length == 0) throw new RuntimeException("must supply a filename");
  CommandLine commandLine = new CommandLine();
  String filename = args[args.length - 1];
  commandLine.onlyCountReady = Stream.of(args).anyMatch(arg -> "-r".equals(arg));
  return countOrders(commandLine, args, filename);
}

private static long countOrders(CommandLine commandLine, String[] args, String filename) throws IOException {
  File input = Paths.get(filename).toFile();
  ObjectMapper mapper = new ObjectMapper();
  Order[] orders = mapper.readValue(input, Order[].class);
  if (commandLine.onlyCountReady)
    return Stream.of(orders).filter(o -> "ready".equals(o.status)).count();
  else
    return orders.length;
}
private static class CommandLine {
  boolean onlyCountReady;
}
```

Move `filename` into `CommandLine` as well:

```java
static long run(String[] args) throws IOException {
  if (args.length == 0) throw new RuntimeException("must supply a filename");
  CommandLine commandLine = new CommandLine();
  commandLine.filename = args[args.length - 1];
  commandLine.onlyCountReady = Stream.of(args).anyMatch(arg -> "-r".equals(arg));
  return countOrders(commandLine, filename);
}

private static long countOrders(CommandLine commandLine, String filename) throws IOException {
  File input = Paths.get(commandLine.filename).toFile();
  ObjectMapper mapper = new ObjectMapper();
  Order[] orders = mapper.readValue(input, Order[].class);
  if (commandLine.onlyCountReady)
    return Stream.of(orders).filter(o -> "ready".equals(o.status)).count();
  else
    return orders.length;
}
private static class CommandLine {
  boolean onlyCountReady;
  String filename;
}
```

With all parameters handled, extract the first phase into its own method:

```java
static long run(String[] args) throws IOException {
  CommandLine commandLine = parseCommandLine(args);
  return countOrders(commandLine);
}

private static CommandLine parseCommandLine(String[] args) {
  if (args.length == 0) throw new RuntimeException("must supply a filename");
  CommandLine commandLine = new CommandLine();
  commandLine.filename = args[args.length - 1];
  commandLine.onlyCountReady = Stream.of(args).anyMatch(arg -> "-r".equals(arg));
  return commandLine;
}

private static long countOrders(CommandLine commandLine) throws IOException {
  File input = Paths.get(commandLine.filename).toFile();
  ObjectMapper mapper = new ObjectMapper();
  Order[] orders = mapper.readValue(input, Order[].class);
  if (commandLine.onlyCountReady)
    return Stream.of(orders).filter(o -> "ready".equals(o.status)).count();
  else
    return orders.length;
}
private static class CommandLine {
  boolean onlyCountReady;
  String filename;
}
```

Tidy with a quick inline and rename:

```java
static long run(String[] args) throws IOException {
  return countOrders(parseCommandLine(args));
}

private static CommandLine parseCommandLine(String[] args) {
  if (args.length == 0) throw new RuntimeException("must supply a filename");
  CommandLine result = new CommandLine();
  result.filename = args[args.length - 1];
  result.onlyCountReady = Stream.of(args).anyMatch(arg -> "-r".equals(arg));
  return result;
}

private static long countOrders(CommandLine commandLine) throws IOException {
  File input = Paths.get(commandLine.filename).toFile();
  ObjectMapper mapper = new ObjectMapper();
  Order[] orders = mapper.readValue(input, Order[].class);
  if (commandLine.onlyCountReady)
    return Stream.of(orders).filter(o -> "ready".equals(o.status)).count();
  else
    return orders.length;
}
private static class CommandLine {
  boolean onlyCountReady;
  String filename;
}
```

The two phases are clearly separated: `parseCommandLine` handles making sense of the command line, while `countOrders` does the actual work. Both can be tested in isolation.

## Example: Using a Transformer for the First Phase (Java)

Instead of a dumb data structure, you can form a transformer object that transforms the command-line array into an interface suitable for the second phase.

Going back to the point where the command-line object was created, create a behavior-rich class instead:

class App...

```java
static long run(String[] args) throws IOException {
  if (args.length == 0) throw new RuntimeException("must supply a filename");
  CommandLine commandLine = new CommandLine(args);
  String filename = args[args.length - 1];
  return countOrders(commandLine, args, filename);
}

private static long countOrders(CommandLine commandLine, String[] args, String filename) throws IOException {
  File input = Paths.get(filename).toFile();
  ObjectMapper mapper = new ObjectMapper();
  Order[] orders = mapper.readValue(input, Order[].class);
  if (Stream.of(args).anyMatch(arg -> "-r".equals(arg)))
    return Stream.of(orders).filter(o -> "ready".equals(o.status)).count();
  else
    return orders.length;
}
```

class CommandLine...

```java
String[] args;

public CommandLine(String[] args) {
  this.args = args;
}
```

This class takes the array of arguments in its constructor and handles first-phase logic via methods that transform the data.

Apply Replace Temp with Query on `filename`:

class App...

```java
static long run(String[] args) throws IOException {
  if (args.length == 0) throw new RuntimeException("must supply a filename");
  CommandLine commandLine = new CommandLine(args);
  return countOrders(commandLine, args, filename(args));
}

private static String filename(String[] args) {
  return args[args.length - 1];
}
```

Apply Move Function to move it into `CommandLine`:

class App...

```java
static long run(String[] args) throws IOException {
  if (args.length == 0) throw new RuntimeException("must supply a filename");
  CommandLine commandLine = new CommandLine(args);
  return countOrders(commandLine, args, commandLine.filename());
}
```

class CommandLine...

```java
String[] args;

public CommandLine(String[] args) {
  this.args = args;
}
String filename() {
  return args[args.length - 1];
}
```

Use Change Function Declaration to adjust `countOrders` to use the new method. Then extract the condition for `args` and move it to the command-line class:

class App...

```java
static long run(String[] args) throws IOException {
  if (args.length == 0) throw new RuntimeException("must supply a filename");
  CommandLine commandLine = new CommandLine(args);
  return countOrders(commandLine);
}

private static long countOrders(CommandLine commandLine) throws IOException {
  File input = Paths.get(commandLine.filename()).toFile();
  ObjectMapper mapper = new ObjectMapper();
  Order[] orders = mapper.readValue(input, Order[].class);
  if (commandLine.onlyCountReady())
    return Stream.of(orders).filter(o -> "ready".equals(o.status)).count();
  else
    return orders.length;
}
```

class CommandLine...

```java
String[] args;

public CommandLine(String[] args) {
  this.args = args;
}
String filename() {
  return args[args.length - 1];
}
boolean onlyCountReady() {
  return Stream.of(args).anyMatch(arg -> "-r".equals(arg));
}
```

Move the empty-args check into the constructor using Move Statements into Function:

class CommandLine...

```java
String[] args;

public CommandLine(String[] args) {
  this.args = args;
  if (args.length == 0) throw new RuntimeException("must supply a filename");
}
String filename() {
  return args[args.length - 1];
}
boolean onlyCountReady() {
  return Stream.of(args).anyMatch(arg -> "-r".equals(arg));
}
```

Both approaches -- a dumb data structure and a transform object -- are reasonable. The key is to make the separation between the two phases clear.

# Replace Exception with Precheck

```
double getValueForPeriod (int periodNumber) {
  try {
    return values[periodNumber];
  } catch (ArrayIndexOutOfBoundsException e) {
    return 0;
  }
}
```

```
double getValueForPeriod (int periodNumber) {
  return (periodNumber >= values.length) ? 0 : values[periodNumber];
}
```

## Motivation

Exceptions should be used for exceptional behavior -- an unexpected error. If you can reasonably expect the caller to check the condition before calling the operation, provide a test for the caller to use instead of throwing an exception.

## Mechanics

- Add a condition that tests the case that triggers the exception. Move the code from the `catch` block into one leg of this condition and put the remaining `try` block in the other.

- Put an assertion into the catch block and test.

- Remove the `try` statement and `catch` block.

- Test.

## Example

I'm looking at a Resource Pool class that manages a collection of resources, such as database connections. Whenever some code wishes to use a resource, it requests one from the pool, and the pool tracks what resources are allocated and which ones are available. If it runs out of resources, it creates a new one.

class ResourcePool…

```
public Resource get() {
  Resource result;
  try {
    result = available.pop();
    allocated.add(result);
   } catch (NoSuchElementException e) {
    result = Resource.create();
    allocated.add(result);
  }
  return result;
}
```

```
private Deque<Resource> available;
private List<Resource> allocated;
```

Running out of resources for the pool isn't an unexpected condition, so exception handling is the wrong mechanism to use. Testing the allocated collection beforehand is just as easy, and more clearly communicates that this is expected behavior.

I add a condition test, moving the code from the `catch` block to one leg and the existing `try` block to the other.

class ResourcePool…

```
public Resource get() {
  Resource result;
  if (available.isEmpty()) {
    result = Resource.create();
    allocated.add(result);
  }
  else {
    try {
      result = available.pop();
      allocated.add(result);
    } catch (NoSuchElementException e) {
    }
  }
  return result;
}
```

Since the catch clause should now no longer be triggered, I'll put an assertion in there.

class ResourcePool…

```
public Resource get() {
  Resource result;
  if (available.isEmpty()) {
    result = Resource.create();
    allocated.add(result);
  }
  else {
    try {
      result = available.pop();
      allocated.add(result);
    } catch (NoSuchElementException e) {
      throw new AssertionError("unreachable");
    }
  }
  return result;
}
```

Once I've tested it with the assertion in place, I can remove the `try` keyword and `catch` block.

class ResourcePool…

```
public Resource get() {
  Resource result;
  if (available.isEmpty()) {
    result = Resource.create();
    allocated.add(result);
  } else {
    result = available.pop();
    allocated.add(result);
  }
  return result;
}
```

Running the tests completes this refactoring -- but the resulting code can be cleaned up further.

First, Slide Statements:

class ResourcePool…

```
public Resource get() {
  Resource result;
  if (available.isEmpty()) {
    result = Resource.create();
  } else {
    result = available.pop();
  }
  allocated.add(result);
  return result;
}
```

I then change the `if`/`else` pair into a ternary:

class ResourcePool…

```
public Resource get() {
  Resource result = available.isEmpty() ? Resource.create() : available.pop();
  allocated.add(result);
  return result;
}
```

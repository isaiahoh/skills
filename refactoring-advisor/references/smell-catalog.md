# Code Smell Catalog

Scan the code against each smell below. When you find one, report it and note which technique(s) to apply. The technique files in `techniques/` have the mechanics and examples — read only the ones you need.

> File path convention: technique name → `techniques/technique_name.md` (e.g., "Extract Function" → `techniques/extract_function.md`)

---

## Mysterious Name
**Spot it**: You have to puzzle over a function, variable, or class name to understand what it does.
**Apply**: Change Function Declaration (rename) · Rename Variable · Rename Field
**Guidance**:
- A good way to improve a name is to write a comment describing the function's purpose, then turn that comment into the name.
- The importance of a variable name depends on scope: single-letter in a one-line lambda is fine; persistent fields need careful naming.

## Duplicated Code
**Spot it**: Same code structure in more than one place — identical or near-identical blocks.
**Apply**:
- Same class → Extract Function, call from both places
- Sibling subclasses → Pull Up Method — *duplicate methods are a breeding ground for bugs; an alteration to one copy may not be made to the other*
- Similar but not identical → Slide Statements to align, then Extract Function
**Guidance**:
- If subclass methods differ, use Parameterize Function until they match, then Pull Up Method.
- If method body refers to features only on the subclass, Pull Up Field first.
- If methods have a similar overall flow but differ in details, consider Form Template Method.

## Long Function
**Spot it**: Function does too much. Comments explaining sections signal extraction points. Look for semantic distance between *what* it does and *how*.
**Apply**:
- Default → Extract Function — *if you have to spend effort figuring out what a fragment does, extract it and name it after the "what"*
- Temps blocking extraction → Replace Temp with Query — *only suitable when the variable is calculated once and only read afterwards*
- Too many parameters → Introduce Parameter Object · Preserve Whole Object
- Still too complex → Replace Function with Command — *only when you need undo, lifecycle hooks, or builder-style construction; prefer plain functions ~95% of the time*
- Conditionals → Decompose Conditional
- Switch on same type in multiple places → Replace Conditional with Polymorphism
- Loop doing multiple things → Split Loop, then Extract Function — *splitting forces two iterations but is rarely a bottleneck, and enables other optimizations*

## Long Parameter List
**Spot it**: Function takes many parameters, especially when several are related.
**Apply**:
- Parameter derivable from another → Replace Parameter with Query — *only if removing the parameter doesn't add an unwanted dependency; don't replace with an access to a mutable global (loses referential transparency)*
- Pulling fields out of an object → Preserve Whole Object — *handles future data needs without changing the parameter list; also a signal to look for Feature Envy*
- Parameters that travel together → Introduce Parameter Object — *the real power is raising new abstractions that simplify understanding of the domain*
- Boolean flag dispatching behavior → Remove Flag Argument — *boolean flags are worst since you can't figure out what `true` means at the call site; must be a literal value, not data flowing through the program*
- Multiple functions sharing same params → Combine Functions into Class

## Global Data
**Spot it**: Global variables, class variables, singletons — data modifiable from anywhere with no tracking.
**Apply**: Encapsulate Variable (always first), then move into a class/module to limit scope
**Guidance**: Encapsulation is much less important for immutable data — when data doesn't change, there's no need for validation or logic hooks.

## Mutable Data
**Spot it**: Data changed in place, especially when multiple code paths can update it or when it could be calculated instead.
**Apply**:
- Wrap updates → Encapsulate Variable — *provides a clear point to monitor changes and add validation or consequential logic*
- Variable reused for different purposes → Split Variable
- Separate pure logic from side effects → Extract Function · Slide Statements
- API that reads and writes → Separate Query from Modifier — *a pure query can be called as often as needed, moved freely, and tested easily*
- Field shouldn't change after init → Remove Setting Method — *removing the setter makes it obvious that updates after construction make no sense*
- Calculated value stored as mutable → Replace Derived Variable with Query — *exception: when the source data for the calculation is immutable and you can force the result to be immutable too*
- Limit update scope → Combine Functions into Class · Combine Functions into Transform
- Replace in-place mutation → Change Reference to Value — *value objects are easier to reason about and especially useful in distributed and concurrent systems*

**Choosing Class vs Transform**: Use a class when the source data gets updated within the code (a transform stores derived data, so if the source changes, you get inconsistencies). Use a transform when the data appears in a read-only context. A class also allows clients to mutate core data while keeping derivations consistent.

## Divergent Change
**Spot it**: One module changes in different ways for different reasons ("I change these functions for database stuff, those for business rules").
**Apply**:
- Sequential concerns → Split Phase — *split when code deals with two different things so you can deal with each separately*
- Back-and-forth between concerns → Move Function
- Mixed within functions → Extract Function first, then move
- Class level → Extract Class — *look for subsets of data that change together or are particularly dependent on each other*

## Shotgun Surgery
**Spot it**: One logical change requires edits scattered across many modules.
**Apply**:
- Consolidate → Move Function · Move Field
- Functions on similar data → Combine Functions into Class
- Data enrichment → Combine Functions into Transform
- Pull together first → Inline Function · Inline Class, then re-extract

**Guidance for Move Field**: Move a field when you always pass it together with another record, when changing one record forces a change in another's field, or when you must update the same field in multiple structures.

## Feature Envy
**Spot it**: A function uses another module's data more than its own — calling many getters on another object.
**Apply**:
- Whole function envious → Move Function
- Part of function envious → Extract Function on that part, then Move Function
- Heuristic: put function with the module that has most of its data
**Guidance**: When deciding where to move a function, examine what functions call it, what it calls, and what data it uses. When the best location is unclear, try a placement and learn from it — the harder the decision, the less it usually matters.

## Data Clumps
**Spot it**: Same 3-4 data items appear together repeatedly (fields, parameters). Test: delete one — do the others still make sense alone?
**Apply**:
- As fields → Extract Class
- As parameters → Introduce Parameter Object · Preserve Whole Object
**Guidance**: Once you identify a data clump and create a structure for it, look for behavior that can move onto that structure — this often raises new abstractions that simplify the domain model.

## Primitive Obsession
**Spot it**: Using strings/numbers/arrays for domain concepts (phone numbers, money, ranges). "Stringly typed" variables.
**Apply**:
- Single concept → Replace Primitive with Object — *at first it just wraps the primitive, but gives you a place for formatting, validation, and comparison logic; many experienced developers consider this one of the most valuable refactorings*
- Type code controlling behavior → Replace Type Code with Subclasses → Replace Conditional with Polymorphism
- Groups of primitives → Extract Class · Introduce Parameter Object

**Choosing direct vs indirect subclasses**: Direct subclassing (Engineer extends Employee) is simpler, but can't be used if inheritance is needed for another axis or if the type is mutable. Indirect subclassing (Employee has an EmployeeType property with its own hierarchy) handles both cases — apply Replace Primitive with Object on the type code first, then Replace Type Code with Subclasses on the new type class.

## Repeated Switches
**Spot it**: Same switch/case or if-else cascade on the same condition in multiple places.
**Apply**: Replace Conditional with Polymorphism — *put base logic in a superclass to reason about it without worrying about variants; put each variant in a subclass, expressed as its difference from the base case*

## Loops
**Spot it**: Traditional for/while loops that could be pipeline operations (map, filter, reduce).
**Apply**: Replace Loop with Pipeline — *describe processing as a series of operations (map, filter, slice, reduce); read top-to-bottom to see how objects flow through the pipeline*

## Lazy Element
**Spot it**: A function/class that doesn't earn its keep — function named same as its body, class that's just one function.
**Apply**: Inline Function · Inline Class · Collapse Hierarchy
**Guidance**:
- Inline Function when a function body is as clear as the name. Also useful when you have badly factored functions — inline them all, then re-extract properly.
- Inline Class when a class no longer pulls its weight. Also useful as a precursor to re-extracting two classes with better boundaries.

## Speculative Generality
**Spot it**: Hooks, abstract classes, extra parameters for "someday" use that never came. Only users are test cases.
**Apply**:
- Unused abstractions → Collapse Hierarchy
- Unnecessary delegation → Inline Function · Inline Class
- Unused parameters → Change Function Declaration
- Only test users → Remove Dead Code (delete the test too)

## Temporary Field
**Spot it**: Class field set/used only in certain circumstances. You expect objects to use all their fields.
**Apply**: Extract Class (home for orphan fields) · Move Function · Introduce Special Case

## Message Chains
**Spot it**: `a.getB().getC().getD()` — client navigates a chain, coupling to the entire structure.
**Apply**: Hide Delegate, or better: Extract Function on what the final object is used for, then Move Function down the chain
**Guidance**: The right amount of hiding changes over time — adjust freely using Hide Delegate and Remove Middle Man as the system evolves. You can mix approaches.

## Middle Man
**Spot it**: Half a class's methods just delegate to another class.
**Apply**: Remove Middle Man · Inline Function · Replace Superclass/Subclass with Delegate

## Insider Trading
**Spot it**: Modules exchanging too much data behind the scenes. Subclasses knowing too much about parents.
**Apply**: Move Function · Move Field · Hide Delegate · Replace Subclass/Superclass with Delegate

## Large Class
**Spot it**: Class with too many fields or too much code. Look for variable name prefixes/suffixes suggesting sub-groupings. Clients using only a subset of features.
**Apply**: Extract Class · Extract Superclass · Replace Type Code with Subclasses
**Guidance**:
- Extract Class when a subset of data and methods go together, or when subtyping affects only a few features.
- Extract Superclass when two classes do similar things — pull similarities together. An alternative is Extract Class with delegation; superclass is often simpler, and you can switch to delegation later.

## Alternative Classes with Different Interfaces
**Spot it**: Two classes doing similar things with different interfaces — can't substitute one for the other.
**Apply**: Change Function Declaration (align interfaces) · Move Function (align behavior) · Extract Superclass

## Data Class
**Spot it**: Class with nothing but fields and getters/setters — manipulated extensively by other classes.
**Apply**:
- Public fields → Encapsulate Record — *objects let you hide storage and provide methods for all values; the user doesn't need to know which is stored and which is calculated*
- Fields that shouldn't change → Remove Setting Method
- Move behavior in → Move Function · Extract Function
- Exception: immutable result records from Split Phase are fine

## Refused Bequest
**Spot it**: Subclass ignores most of what it inherits. Mild if reusing behavior, strong if rejecting the *interface*.
**Apply**:
- Mild → Push Down Method · Push Down Field to sibling
- Refusing interface → Replace Subclass with Delegate · Replace Superclass with Delegate

**Choosing Subclass vs Superclass with Delegate**: Use Replace Subclass with Delegate when inheritance can only model one axis of variation but you need multiple, or when you need to change the "type" dynamically. Use Replace Superclass with Delegate when the superclass's interface doesn't fully apply (e.g., Stack extending List exposes invalid operations). In both cases: use inheritance first when it fits (it's simpler), switch to delegation when it starts causing problems.

## Comments (as Deodorant)
**Spot it**: Thick comments that exist because the code is unclear. Comments explaining *what* (bad) vs *why* (good).
**Apply**:
- Comment explaining a block → Extract Function (name after the intent)
- Comment explaining a function → Change Function Declaration (better name)
- Comment stating required state → Introduce Assertion — *assertions communicate assumed state and are handy for debugging; only use for programmer errors, not external data validation*

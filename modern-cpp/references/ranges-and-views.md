# Ranges & Views (C++20/23)

## Table of Contents
1. Core Idea
2. Range Algorithms vs Classic Algorithms
3. Views & Lazy Evaluation
4. The Pipe Operator
5. C++23 View Additions
6. Projections
7. Practical Examples

---

## 1. Core Idea

A **range** is anything with `begin()` and `end()`. All standard containers are
ranges. The `<ranges>` library provides composable, lazy view adaptors and
range-aware algorithms.

Benefits over raw iterator pairs:
- Pass the container directly instead of `begin()`/`end()` pairs
- Compose operations with the pipe `|` operator
- Lazy evaluation — no intermediate allocations
- Projections — transform elements before applying an algorithm

## 2. Range Algorithms vs Classic Algorithms

Range algorithms live in `std::ranges::` and accept ranges directly:

```cpp
std::vector<int> v = {3, 1, 4, 1, 5, 9};

// Classic — verbose
std::sort(v.begin(), v.end());
auto it = std::find(v.begin(), v.end(), 4);

// Ranges — concise
std::ranges::sort(v);
auto it = std::ranges::find(v, 4);
```

Range algorithms also support **projections** (see section 6), which eliminate
the need for many custom comparators.

## 3. Views & Lazy Evaluation

A **view** is a lightweight range that doesn't own its elements. Views are cheap
to copy and compose. They evaluate lazily — elements are computed on demand.

```cpp
auto evens = v | std::views::filter([](int x) { return x % 2 == 0; });
// No work done yet — evens is just a lazy description
for (int x : evens) {
    // Elements are filtered one at a time during iteration
}
```

### Common C++20 Views

| View                        | What it does                              |
|----------------------------|-------------------------------------------|
| `views::filter(pred)`     | Keep elements satisfying pred             |
| `views::transform(f)`     | Apply f to each element                   |
| `views::take(n)`          | First n elements                          |
| `views::drop(n)`          | Skip first n elements                     |
| `views::reverse`          | Reverse order                             |
| `views::split(delim)`     | Split into subranges by delimiter         |
| `views::join`             | Flatten nested ranges                     |
| `views::elements<N>`      | Nth element of tuple-like elements        |
| `views::keys` / `values`  | Keys/values of pair-like elements         |
| `views::counted(it, n)`   | View n elements starting from iterator    |
| `views::iota(start)`      | Infinite sequence: start, start+1, ...    |
| `views::iota(start, end)` | Bounded sequence: [start, end)            |
| `views::single(val)`      | Single-element view                       |
| `views::empty<T>`         | Empty view of type T                      |

## 4. The Pipe Operator

Views compose left-to-right with `|`:

```cpp
auto result = employees
    | std::views::filter([](const auto& e) { return e.department == "Eng"; })
    | std::views::transform(&Employee::name)
    | std::views::take(5);
```

This reads naturally: "take employees, keep those in Eng, extract names, take 5."

The pipe operator is syntactic sugar for nesting:
```cpp
// Equivalent but less readable:
auto result = std::views::take(
    std::views::transform(
        std::views::filter(employees, pred),
        &Employee::name),
    5);
```

## 5. C++23 View Additions

C++23 significantly expands the views library:

| View                          | What it does                              |
|------------------------------|-------------------------------------------|
| `views::zip(r1, r2, ...)`   | Combine ranges element-wise into tuples   |
| `views::zip_transform(f, r1, r2)` | Zip + transform in one step        |
| `views::enumerate`          | Pairs each element with its index         |
| `views::chunk(n)`           | Split into non-overlapping chunks of n    |
| `views::slide(n)`           | Sliding window of size n                  |
| `views::chunk_by(pred)`     | Chunk where adjacent elements satisfy pred |
| `views::join_with(delim)`   | Flatten with delimiter between subranges  |
| `views::stride(n)`          | Every nth element                         |
| `views::as_rvalue`          | Cast elements to rvalue references        |
| `views::as_const`           | View elements as const                    |
| `views::repeat(val)`        | Infinite repetition of a value            |
| `views::repeat(val, n)`     | Repeat val n times                        |
| `views::cartesian_product`  | All combinations from multiple ranges     |

### C++23 Fold Algorithms
```cpp
auto sum = std::ranges::fold_left(v, 0, std::plus{});
auto product = std::ranges::fold_left(v, 1, std::multiplies{});
```

## 6. Projections

Projections let you "project" elements through a function before the algorithm
operates on them. This replaces many custom comparators:

```cpp
struct Employee {
    std::string name;
    int salary;
};

std::vector<Employee> team = /* ... */;

// Sort by salary — no custom comparator needed
std::ranges::sort(team, std::ranges::less{}, &Employee::salary);

// Find by name
auto it = std::ranges::find(team, "Alice", &Employee::name);

// Max salary
auto highest = std::ranges::max(team, {}, &Employee::salary);
```

Projections work with any callable, including lambdas:

```cpp
std::ranges::sort(team, {}, [](const Employee& e) {
    return std::pair{e.department, e.salary};
});
```

## 7. Practical Examples

### Flatten and deduplicate
```cpp
std::vector<std::vector<int>> nested = {{1,2,3},{2,3,4},{4,5,6}};
auto flat = nested | std::views::join;

std::vector<int> unique_sorted(std::ranges::begin(flat), std::ranges::end(flat));
std::ranges::sort(unique_sorted);
auto [erased, end] = std::ranges::unique(unique_sorted);
unique_sorted.erase(erased, unique_sorted.end());
```

### CSV-like processing
```cpp
std::string line = "Alice,30,Engineering";
auto fields = line
    | std::views::split(',')
    | std::views::transform([](auto&& chunk) {
        return std::string_view(chunk.begin(), chunk.end());
    });
```

### Enumerate with index
```cpp
for (auto [idx, val] : v | std::views::enumerate) {
    std::println("[{}] = {}", idx, val);
}
```

### Generate a range of values
```cpp
// First 10 squares
auto squares = std::views::iota(1, 11)
    | std::views::transform([](int n) { return n * n; });
```

### Zip two ranges
```cpp
std::vector<std::string> names = {"Alice", "Bob"};
std::vector<int> scores = {95, 87};

for (auto [name, score] : std::views::zip(names, scores)) {
    std::println("{}: {}", name, score);
}
```

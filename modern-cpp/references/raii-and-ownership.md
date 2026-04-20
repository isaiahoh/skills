# RAII & Ownership Patterns

## Table of Contents
1. The RAII Principle
2. Smart Pointer Selection Guide
3. unique_ptr Patterns
4. shared_ptr & weak_ptr Patterns
5. Custom RAII Wrappers
6. Rule of Zero / Five
7. Passing Smart Pointers to Functions
8. Common Mistakes

---

## 1. The RAII Principle

RAII (Resource Acquisition Is Initialization) ties a resource's lifetime to an
object's lifetime. Acquire in the constructor, release in the destructor.
Because C++ guarantees destructors run when objects leave scope — even during
stack unwinding from exceptions — RAII provides automatic, exception-safe
cleanup.

This applies to *all* resources, not just memory: file handles, mutexes,
database connections, GPU buffers, network sockets.

```cpp
// RAII mutex lock — already in the standard library
{
    std::scoped_lock lock(mtx);
    // critical section
}   // lock released here, even if an exception is thrown
```

## 2. Smart Pointer Selection Guide

Ask these questions in order:

1. **Does anyone need to own this on the heap?** If not, use a local value or
   a container. Avoid the heap entirely when possible.
2. **Is there exactly one owner?** Use `std::unique_ptr<T>`. This is the
   default choice for heap allocation.
3. **Do multiple independent owners need to share lifetime?** Use
   `std::shared_ptr<T>`. This is rarer than people think.
4. **Do you need a non-owning handle to a shared resource?** Use
   `std::weak_ptr<T>` to avoid reference cycles.
5. **Do you need a non-owning view of something owned elsewhere?** Use a raw
   `T*` (nullable) or `T&` (non-nullable). These are fine as function
   parameters.

## 3. unique_ptr Patterns

### Creation
```cpp
auto widget = std::make_unique<Widget>(42, "hello");
```

Always prefer `make_unique` over `new`. It is exception-safe and avoids
repeating the type.

### Transfer of Ownership
```cpp
void take_ownership(std::unique_ptr<Widget> w);
auto w = std::make_unique<Widget>();
take_ownership(std::move(w));  // w is now null
```

### Custom Deleter (for non-memory resources)
```cpp
struct FileCloser {
    void operator()(std::FILE* f) const {
        if (f) std::fclose(f);
    }
};
using UniqueFile = std::unique_ptr<std::FILE, FileCloser>;

auto f = UniqueFile(std::fopen("data.txt", "r"));
// file closes automatically when f goes out of scope
```

### unique_ptr to Array
```cpp
auto arr = std::make_unique<int[]>(100);  // array of 100 ints
arr[0] = 42;
```

Prefer `std::vector` unless you need a fixed-size heap array with no overhead.

### Factory Functions
```cpp
std::unique_ptr<Shape> create_shape(ShapeType type) {
    switch (type) {
        case ShapeType::Circle: return std::make_unique<Circle>();
        case ShapeType::Square: return std::make_unique<Square>();
    }
    std::unreachable();
}
```

## 4. shared_ptr & weak_ptr Patterns

### When shared_ptr Is Appropriate
- Caches where entries may outlive the cache itself
- Observer patterns where observers hold references to subjects
- Thread pools sharing task objects across workers
- Graph-like structures with shared nodes

### Breaking Cycles with weak_ptr
```cpp
struct Node {
    std::string name;
    std::vector<std::shared_ptr<Node>> children;
    std::weak_ptr<Node> parent;  // weak to prevent cycle
};

auto root = std::make_shared<Node>("root");
auto child = std::make_shared<Node>("child");
child->parent = root;  // non-owning back-reference
root->children.push_back(child);
```

### Checking weak_ptr Validity
```cpp
if (auto locked = weak.lock()) {
    // locked is a valid shared_ptr
    locked->do_work();
}
// else: the object has been destroyed
```

### enable_shared_from_this
When an object managed by `shared_ptr` needs to hand out `shared_ptr`s to
itself:

```cpp
class Session : public std::enable_shared_from_this<Session> {
public:
    void start() {
        auto self = shared_from_this();
        async_read(socket_, [self](auto ec, auto n) {
            self->handle_read(ec, n);
        });
    }
};
// Always create with make_shared:
auto session = std::make_shared<Session>();
```

## 5. Custom RAII Wrappers

For resources that don't fit smart pointers (e.g., OS handles, C library
resources), write a minimal RAII wrapper:

```cpp
class MappedFile {
public:
    explicit MappedFile(const std::filesystem::path& path)
        : fd_(::open(path.c_str(), O_RDONLY)) {
        if (fd_ < 0) throw std::system_error(errno, std::generic_category());
        struct stat st;
        ::fstat(fd_, &st);
        size_ = st.st_size;
        data_ = ::mmap(nullptr, size_, PROT_READ, MAP_PRIVATE, fd_, 0);
        if (data_ == MAP_FAILED) {
            ::close(fd_);
            throw std::system_error(errno, std::generic_category());
        }
    }

    ~MappedFile() {
        if (data_ != MAP_FAILED) ::munmap(data_, size_);
        if (fd_ >= 0) ::close(fd_);
    }

    // Non-copyable, movable
    MappedFile(const MappedFile&) = delete;
    MappedFile& operator=(const MappedFile&) = delete;
    MappedFile(MappedFile&& other) noexcept
        : fd_(std::exchange(other.fd_, -1))
        , data_(std::exchange(other.data_, MAP_FAILED))
        , size_(std::exchange(other.size_, 0)) {}
    MappedFile& operator=(MappedFile&&) noexcept;  // similar pattern

    std::span<const std::byte> data() const {
        return {static_cast<const std::byte*>(data_), size_};
    }

private:
    int fd_ = -1;
    void* data_ = MAP_FAILED;
    std::size_t size_ = 0;
};
```

## 6. Rule of Zero / Five

**Rule of Zero** (preferred): if all your members are RAII types, the
compiler-generated special members are correct. Write no destructor, no copy/move
constructors, no assignment operators.

```cpp
class Document {
    std::string title_;
    std::vector<Page> pages_;
    std::unique_ptr<Metadata> meta_;
    // No need for ~Document(), copy/move ctors, or assignment.
    // Compiler defaults do the right thing.
};
```

**Rule of Five**: if you *must* manage a resource directly (rare in modern C++),
define all five special members:

```cpp
class Buffer {
public:
    explicit Buffer(std::size_t n) : data_(new std::byte[n]), size_(n) {}
    ~Buffer() { delete[] data_; }
    Buffer(const Buffer& o) : data_(new std::byte[o.size_]), size_(o.size_) {
        std::copy_n(o.data_, size_, data_);
    }
    Buffer& operator=(const Buffer& o) {
        if (this != &o) { Buffer tmp(o); swap(tmp); }
        return *this;
    }
    Buffer(Buffer&& o) noexcept
        : data_(std::exchange(o.data_, nullptr))
        , size_(std::exchange(o.size_, 0)) {}
    Buffer& operator=(Buffer&& o) noexcept {
        if (this != &o) { Buffer tmp(std::move(o)); swap(tmp); }
        return *this;
    }
    void swap(Buffer& o) noexcept {
        std::swap(data_, o.data_);
        std::swap(size_, o.size_);
    }
private:
    std::byte* data_;
    std::size_t size_;
};
```

Better yet, replace this entire class with `std::vector<std::byte>` and apply
the Rule of Zero.

## 7. Passing Smart Pointers to Functions

Follow Herb Sutter's guidelines:

| You want to...                     | Pass as                          |
|------------------------------------|----------------------------------|
| Use the object (not the pointer)   | `const T&` or `T&`              |
| Take ownership                     | `std::unique_ptr<T>` by value   |
| Share ownership                    | `std::shared_ptr<T>` by value   |
| Maybe reseat a unique_ptr          | `std::unique_ptr<T>&`           |
| Observe, might be null             | `T*`                            |
| View a contiguous range            | `std::span<T>` or `std::span<const T>` |

**Do not** pass `const std::unique_ptr<T>&` just to access `T` — pass `const T&`
instead. The indirection through the smart pointer adds nothing and tightly
couples callers to a specific ownership strategy.

## 8. Common Mistakes

- **Creating `shared_ptr` from raw pointer twice**: this creates two independent
  reference counts and double-frees. Always create from `make_shared` or convert
  from `unique_ptr`.
- **Storing `shared_ptr` everywhere "just in case"**: shared ownership has
  overhead (atomic ref counting) and makes lifetime analysis harder. Default to
  `unique_ptr`.
- **Calling `.get()` and storing the raw pointer**: if you store the raw pointer
  and the smart pointer is destroyed, you have a dangling pointer.
- **Forgetting `noexcept` on move operations**: this prevents `std::vector` from
  using moves during reallocation, falling back to copies.

---
icon: thumbtack
---

# Understanding Unpin and Pinning in Rust Async Programming

### What is the Unpin Trait?

`Unpin` is a **marker trait** in Rust - it has **no methods** at all!

```rust
pub auto trait Unpin {}
```

#### Key Characteristics:

1. **Zero methods**: `Unpin` doesn't define any functionality
2. **Auto trait**: Most types automatically implement `Unpin`
3. **Marker only**: It just "marks" that a type is safe to move in memory

#### What it does:

`Unpin` is purely a **compile-time safety marker** that tells Rust:

* "This type is safe to move around in memory"
* "This type doesn't have self-references that would break if moved"

#### Types that are NOT Unpin:

* Async blocks/futures (they can have self-references)
* Types explicitly marked with `!Unpin` (like `PhantomPinned`)

#### How it's used:

```rust
// Functions can require Unpin as a bound
fn needs_unpin<T: Unpin>(value: T) { /* ... */ }

// Pin<P> implements Unpin when P::Target: Unpin
// This is why Pin<Box<T>> is Unpin even when T isn't
```

So `Unpin` is just a "permission slip" that says "yes, you can safely move this type around" - no actual methods involved!

***

### How Pin\<Box> Satisfies Unpin

`Pin<Box<T>>` satisfies the `Unpin` trait through a clever implementation detail:

#### The Key Insight

```rust
// Pin<Box<T>> implements Unpin regardless of whether T implements Unpin
impl<T: ?Sized> Unpin for Pin<Box<T>> {}
```

#### How it Works

**The Paradox**: Even though the inner `T` (your async block/future) is **NOT** `Unpin`, the `Pin<Box<T>>` wrapper **IS** `Unpin`.

**Why this works**:

1. **The Box itself can move**: The `Pin<Box<T>>` wrapper can be moved around in memory
2. **The contents stay pinned**: The `T` inside the `Box` remains at a fixed memory location on the heap
3. **Safe to move the pointer**: Moving the `Pin<Box<T>>` just moves the pointer, not the actual future

#### Example

```rust
// ❌ Future itself cannot move (not Unpin)
let future = async { /* self-referential state */ };

// ✅ Pin<Box<Future>> can move (implements Unpin)
let pinned = Box::pin(future);
//           ^^^^^^^^^^ This wrapper can move around
//                     The future inside stays put on heap
```

**Bottom line**: `Pin<Box<T>>` is `Unpin` because moving the `Pin<Box<T>>` doesn't move the pinned data - it just moves a pointer to that data. The actual future remains safely pinned in its heap location.

***

### Determining if a Type is Unpin

You can determine if a type can be `Unpin` by checking these criteria:

#### 🟢 Types that ARE `Unpin` (automatically):

1. **Most basic types**: `i32`, `String`, `Vec<T>`, `Option<T>`, etc.
2. **Types with no self-references**: Structs that don't reference their own fields
3. **Types that explicitly implement `Unpin`**

#### 🔴 Types that are NOT `Unpin`:

1. **Async blocks/futures** - they can have self-references
2. **Types containing `PhantomPinned`**
3. **Types explicitly marked `!Unpin`**

#### 🔍 How to Check:

**Method 1: Compiler Test**

```rust
fn is_unpin<T: Unpin>(_: T) {}

// This will compile if the type is Unpin
let future = async { /* ... */ };
is_unpin(future); // ❌ Compile error - futures are !Unpin
```

**Method 2: Check Documentation**

* Look for `impl Unpin for YourType` in docs
* Most types implement it automatically via `auto trait Unpin`

**Method 3: Look for Self-References**

```rust
// ❌ NOT Unpin - self-referential
struct SelfRef {
    data: String,
    ptr: *const String, // Points to data field
}

// ✅ IS Unpin - no self-references  
struct Normal {
    data: String,
    count: i32,
}
```

#### 🧪 Practical Test:

```rust
use std::marker::Unpin;

fn test_unpin<T: Unpin>() {}

// Test different types
test_unpin::<i32>();      // ✅ Compiles
test_unpin::<String>();   // ✅ Compiles  
test_unpin::<Vec<u8>>();  // ✅ Compiles

// This would fail to compile:
// test_unpin::<impl Future<Output = ()>>();  // ❌ Error
```

***

### Self-Referential Structs Explained

"Structs that don't reference their own fields" means the struct doesn't contain pointers/references that point back to other fields within the same struct instance.

#### ✅ **Safe (Unpin) - No Self-References**

```rust
// Normal struct - fields are independent
struct Person {
    name: String,
    age: u32,
    email: String,
}

// No field points to another field in the same struct
// Safe to move this struct around in memory
```

```rust
// Another safe example
struct Config {
    host: String,
    port: u16,
    buffer: Vec<u8>,
}
// Each field is independent - no internal pointers
```

#### ❌ **Unsafe (!Unpin) - Self-Referential**

```rust
// DANGEROUS: Self-referential struct
struct SelfReferential {
    data: String,
    ptr: *const String,  // Points to the 'data' field above!
}

impl SelfReferential {
    fn new(s: String) -> Self {
        let mut result = SelfReferential {
            data: s,
            ptr: std::ptr::null(),
        };
        // Make ptr point to our own data field
        result.ptr = &result.data as *const String;
        result
    }
}
```

#### 🔍 **Why Self-References Break When Moved**

<pre class="language-rust"><code class="lang-rust">let mut original = SelfReferential::new("hello".to_string());
// ptr points to original.data at memory address 0x1000

<a data-footnote-ref href="#user-content-fn-1">let moved = original</a>; // Move the struct
// Now moved.data is at address 0x2000
// But moved.ptr still points to 0x1000 (old location!)
// ❌ DANGLING POINTER!
</code></pre>

#### 🔧 **Async Blocks Create Hidden Self-References**

```rust
async fn example() {
    let data = String::from("hello");
    let reference = &data;  // ⚠️ Reference to local variable
    
    some_async_call().await; // Await point creates state machine
    
    println!("{}", reference); // Still using the reference
}
```

The compiler transforms this into something like:

```rust
// Simplified version of what compiler generates
enum ExampleFuture {
    State1 {
        data: String,
        reference: *const String, // Points to data field!
    },
    State2 { /* ... */ },
}
```

**Key Point**: The `reference` field points to the `data` field within the same enum variant - making it self-referential and therefore `!Unpin`.

This is why async blocks need `Pin<>` to prevent them from moving in memory!

***

### Practical Application: Fixing the join\_all Error

When working with collections of futures, you'll encounter the `Unpin` requirement:

```rust
// ❌ This fails because futures are !Unpin
let futures = vec![Box::new(future1), Box::new(future2)];
trpl::join_all(futures).await; // Error!

// ✅ This works because Pin<Box<Future>> is Unpin
let futures = vec![Box::pin(future1), Box::pin(future2)];
trpl::join_all(futures).await; // Success!
```

#### The Fix Explained:

1. `Box::new(future)` creates `Box<dyn Future>` which is `!Unpin`
2. `Box::pin(future)` creates `Pin<Box<dyn Future>>` which is `Unpin`
3. `join_all()` requires `Unpin` futures, so only the pinned version works

This pattern is essential when working with dynamic collections of futures in async Rust programming.

***

### Conclusion

Understanding `Unpin` and `Pin` is crucial for effective async Rust programming:

* **Unpin** is a marker trait indicating safe-to-move types
* **Pin** prevents movement while still allowing the wrapper to be `Unpin`
* **Self-references** are the main reason types become `!Unpin`
* **Async blocks** create hidden self-references, requiring pinning for collections

When you see `Unpin` trait bound errors, remember: `Box::pin()` is often the solution!

[^1]: \`data\` is string in this case, to technically it a value on stack pointing to a memory (where the string content is) in heap. This statement move the data itself not the string content in the heap.&#x20;

# Operators

Move programs can create, delete, and update resources in global storage using the following five instructions:

| Operation                               | Description                                                     | Aborts?                                 |
| --------------------------------------- | --------------------------------------------------------------- | --------------------------------------- |
| `move_to<T>(&signer,T)`                 | Publish `T` under `signer.address`                              | If `signer.address` already holds a `T` |
| `move_from<T>(address): T`              | Remove `T` from `address` and return it                         | If `address` does not hold a `T`        |
| `borrow_global_mut<T>(address): &mut T` | Return a mutable reference to the `T` stored under `address`    | If `address` does not hold a `T`        |
| `borrow_global<T>(address): &T`         | Return an immutable reference to the `T` stored under `address` | If `address` does not hold a `T`        |
| `exists<T>(address): bool`              | Return `true` if a `T` is stored under `address`                | Never                                   |

Each of these instructions is parameterized by a type `T` with the `key` ability. However, each type `T` _must be declared in the current module_. This ensures that a resource can only be manipulated via the API exposed by its defining module. The instructions also take either an `address` or `&signer` representing the account address where the resource of type `T` is stored.

## References to resources

References to global resources returned by `borrow_global` or `borrow_global_mut` mostly behave like references to local storage: they can be extended, read, and written using ordinary reference operators and passed as arguments to other function. However, there is one important difference between local and global references: **a function cannot return a reference that points into global storage**. For example, these two functions will each fail to compile:

```move
module 0x42::example {
  struct R has key { f: u64 }
  // will not compile
  fun ret_direct_resource_ref_bad(a: address): &R {
    borrow_global<R>(a) // error!
  }
  // also will not compile
  fun ret_resource_field_ref_bad(a: address): &u64 {
    &borrow_global<R>(a).f // error!
  }
}
```

Move must enforce this restriction to guarantee absence of dangling references to global storage. [This section](broken-reference) contains much more detail for the interested reader.

## Global storage operators with generics

Global storage operations can be applied to generic resources with both instantiated and uninstantiated generic type parameters:

```move
module 0x42::example {
  struct Container<T> has key { t: T }

  // Publish a Container storing a type T of the caller's choosing
  fun publish_generic_container<T>(account: &signer, t: T) {
    move_to<Container<T>>(account, Container { t })
  }

  /// Publish a container storing a u64
  fun publish_instantiated_generic_container(account: &signer, t: u64) {
    move_to<Container<u64>>(account, Container { t })
  }
}
```

The ability to index into global storage via a type parameter chosen at runtime is a powerful Move feature known as _storage polymorphism_. For more on the design patterns enabled by this feature, see Move generics.

## Example: `Counter`

The simple `Counter` module below exercises each of the five global storage operators. The API exposed by this module allows:

* Anyone to publish a `Counter` resource under their account
* Anyone to check if a `Counter` exists under any address
* Anyone to read or increment the value of a `Counter` resource under any address
* An account that stores a `Counter` resource to reset it to zero
* An account that stores a `Counter` resource to remove and delete it

```move
module 0x42::counter {
  use std::signer;

  /// Resource that wraps an integer counter
  struct Counter has key { i: u64 }

  /// Publish a `Counter` resource with value `i` under the given `account`
  public fun publish(account: &signer, i: u64) {
    // "Pack" (create) a Counter resource. This is a privileged operation that
    // can only be done inside the module that declares the `Counter` resource
    move_to(account, Counter { i })
  }

  /// Read the value in the `Counter` resource stored at `addr`
  public fun get_count(addr: address): u64 acquires Counter {
    borrow_global<Counter>(addr).i
  }

  /// Increment the value of `addr`'s `Counter` resource
  public fun increment(addr: address) acquires Counter {
    let c_ref = &mut borrow_global_mut<Counter>(addr).i;
    *c_ref = *c_ref + 1
  }

  /// Reset the value of `account`'s `Counter` to 0
  public fun reset(account: &signer) acquires Counter {
    let c_ref = &mut borrow_global_mut<Counter>(signer::address_of(account)).i;
    *c_ref = 0
  }

  /// Delete the `Counter` resource under `account` and return its value
  public fun delete(account: &signer): u64 acquires Counter {
    // remove the Counter resource
    let c = move_from<Counter>(signer::address_of(account));
    // "Unpack" the `Counter` resource into its fields. This is a
    // privileged operation that can only be done inside the module
    // that declares the `Counter` resource
    let Counter { i } = c;
    i
  }

  /// Return `true` if `addr` contains a `Counter` resource
  public fun exists_at(addr: address): bool {
    exists<Counter>(addr)
  }
}
```

## Annotating functions with `acquires`

In the `counter` example, you might have noticed that the `get_count`, `increment`, `reset`, and `delete` functions are annotated with `acquires Counter`. A Move function `m::f` must be annotated with `acquires T` if and only if:

* The body of `m::f` contains a `move_from<T>`, `borrow_global_mut<T>`, or `borrow_global<T>` instruction, or
* The body of `m::f` invokes a function `m::g` declared in the same module that is annotated with `acquires`

For example, the following function inside `Counter` would need an `acquires` annotation:

```move
module 0x42::example {
  // Needs `acquires` because `increment` is annotated with `acquires`
  fun call_increment(addr: address): u64 acquires Counter {
    counter::increment(addr)
  }
}
```

However, the same function _outside_ `Counter` would not need an annotation:

```move
module 0x43::m {
  use 0x42::counter;

  // Ok. Only need annotation when resource acquired by callee is declared
  // in the same module
  fun call_increment(addr: address): u64 {
    counter::increment(addr)
  }
}
```

If a function touches multiple resources, it needs multiple `acquires`:

```move
module 0x42::two_resources {
  struct R1 has key { f: u64 }
  struct R2 has key { g: u64 }

  fun double_acquires(a: address): u64 acquires R1, R2 {
    borrow_global<R1>(a).f + borrow_global<R2>(a).g
  }
}
```

The `acquires` annotation does not take generic type parameters into account:

```move
module 0x42::m {
  struct R<T> has key { t: T }

  // `acquires R`, not `acquires R<T>`
  fun acquire_generic_resource<T: store>(a: addr) acquires R {
    let _ = borrow_global<R<T>>(a);
  }

  // `acquires R`, not `acquires R<u64>
  fun acquire_instantiated_generic_resource(a: addr) acquires R {
    let _ = borrow_global<R<u64>>(a);
  }
}
```

Finally: redundant `acquires` are not allowed. Adding this function inside `Counter` will result in a compilation error:

```move
module 0x42::m {
  // This code will not compile because the body of the function does not use a global
  // storage instruction or invoke a function with `acquires`
  fun redundant_acquires_bad() acquires Counter {}
}
```

For more information on `acquires`, see Move functions.

## Reference Safety For Global Resources

Move prohibits returning global references and requires the `acquires` annotation to prevent dangling references. This allows Move to live up to its promise of static reference safety (i.e., no dangling references, no `null` or `nil` dereferences) for all reference types.

This example illustrates how the Move type system uses `acquires` to prevent a dangling reference:

```move
module 0x42::dangling {
  struct T has key { f: u64 }

  fun borrow_then_remove_bad(a: address) acquires T {
    let t_ref: &mut T = borrow_global_mut<T>(a);
    let t = remove_t(a); // type system complains here
    // t_ref now dangling!
    let uh_oh = *&t_ref.f;
  }

  fun remove_t(a: address): T acquires T {
    move_from<T>(a)
  }
}
```

In this code, line 6 acquires a reference to the `T` stored at address `a` in global storage. The callee `remove_t` then removes the value, which makes `t_ref` a dangling reference.

Fortunately, this cannot happen because the type system will reject this program. The `acquires` annotation on `remove_t` lets the type system know that line 7 is dangerous, without having to recheck or introspect the body of `remove_t` separately!

The restriction on returning global references prevents a similar, but even more insidious problem:

```move
address 0x42 {
  module m1 {
    struct T has key {}

    public fun ret_t_ref(a: address): &T acquires T {
      borrow_global<T>(a) // error! type system complains here
    }

    public fun remove_t(a: address) acquires T {
      let T {} = move_from<T>(a);
    }
  }

  module m2 {
    fun borrow_then_remove_bad(a: address) {
      let t_ref = m1::ret_t_ref(a);
      let t = m1::remove_t(a); // t_ref now dangling!
    }
  }
}
```

Line 16 acquires a reference to a global resource `m1::T`, then line 17 removes that same resource, which makes `t_ref` dangle. In this case, `acquires` annotations do not help us because the `borrow_then_remove_bad` function is outside the `m1` module that declares `T` (recall that `acquires` annotations can only be used for resources declared in the current module). Instead, the type system avoids this problem by preventing the return of a global reference at line 6.

Fancier type systems that would allow returning global references without sacrificing reference safety are possible, and we may consider them in future iterations of Move. We chose the current design because it strikes a good balance between being expressive, annotation burden, and type system complexity.

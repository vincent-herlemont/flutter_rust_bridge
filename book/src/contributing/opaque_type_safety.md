# Opaque type safety


## Restrictions

A `Opaque type` can be created from any Rust structure that implements `Send` and `Sync`. Such restrictions due to async dart api `flutter_rust_bridge` allowing sharing a `Opaque type` by multiple `flutter_rust_bridge executor` threads.


## Ownership and GC

From the moment an opaque type is passed to Dart, it has full ownership of it.
Dart implements a finalizer for opaque types, but the memory usage of opaque types is not monitored by Dart and can accumulate, so in order to prevent memory leaks, opaque pointers must be `dispose`d.


## Opaque type like funtion args

When calling a function with an opaque type argument, the Dart thread safely shares ownership of the opaque type with Rust. This is safe because `Opaque<T>` requires that T be `Send` and `Sync`, furthermore Rust's `Opaque<T>` can only hand out immutable references through `Deref`. If dispose is called on the Dart side before the function call completes, Rust takes full ownership.


## Example

### Case 1: Simple call. 

Rust `api.rs`:
```rust,noplayground
pub use crate::data::HideData; // `pub` for bridge_generated.rs

pub fn create_opaque() -> Opaque<OpaqueStruct> {
    // [`HideData`] has private fields.
    Opaque::new(HideData::new())
}

pub fn run_opaque(opaque: Opaque<HideData>) -> String {
    // Opaque impl Deref trait.
    opaque.hide_data()
}
```

Dart: (test:'Simple call' frb_example/pure_dart/dart/lib/main.dart)
```dart
// (Arc counter = 1) Dart has full ownership.
var opaque = await api.createOpaque();

// (Arc counter = 2) for the duration of the function 
// and after (Arc counter = 1).
// 
// Dart and Rust share the opaque type.
String hideData = await api.runOpaque(opaque);

// (Arc counter = 0) opaque type is dropped (deallocated).
opaque.dispose();
```



### Case 2: Call after dispose.

Rust `api.rs`:
```rust,noplayground
pub use crate::data::HideData; // `pub` for bridge_generated.rs

pub fn create_opaque() -> Opaque<OpaqueStruct> {
    // [`HideData`] has private fields.
    Opaque::new(HideData::new())
}

pub fn run_opaque(opaque: Opaque<HideData>) -> String {
    // Opaque impl Deref trait.
    opaque.hide_data()
}
```

Dart: (test:'Call after dispose' frb_example/pure_dart/dart/lib/main.dart)
```dart
// (Arc counter = 1) Dart has full ownership.
var opaque = await api.createOpaque();

// (Arc counter = 0) opaque type dropped (deallocated)
opaque.dispose();

// (Arc counter = 0) Dart throws StateError('Use after dispose.')
try {
    await api.runOpaque(opaque: opaque);
} on StateError catch (e) {
    expect(e.toString(), 'Bad state: Use after dispose.');
}
```


### Case 3: Dispose before complete.

Rust `api.rs`:
```rust,noplayground
pub use crate::data::HideData; // `pub` for bridge_generated.rs

pub fn create_opaque() -> Opaque<OpaqueStruct> {
    // [`HideData`] has private fields.
    Opaque::new(HideData::new())
}

pub fn run_opaque(opaque: Opaque<HideData>) -> String {
    // Opaque impl Deref trait.
    opaque.hide_data()
}

pub fn run_opaque_with_delay(opaque: Opaque<HideData>) -> String {
    sleep(Duration::from_millis(1000));
    opaque.hide_data()
}
```

Dart:
```dart
// (Arc counter = 1) Dart has full ownership.
var opaque = await api.createOpaque();

// (Arc counter = 2) increases immediately. 
// Dart and Rust share the opaque type.
// Safely because opaque type has `Send` `Sync` Rust trait.
var unawait_task = api.runOpaqueWithDelay(opaque: opaque);

// (Arc counter = 1) Rust has full ownership.
// Dart stops owning the opaque type. 
// Trying to use an opaque type will throw StateError('Use after dispose.')
opaque.dispose();

// Successfully completed.
//
// Rust:
// `executes run_opaque_with_delay.`
// after complete (Arc counter = 0) 
// opaque type is dropped (deallocated)
await unawait_task;
```


### Case 4: Multi call.

Rust `api.rs`:
```rust,noplayground
pub use crate::data::HideData; // `pub` for bridge_generated.rs

pub fn create_opaque() -> Opaque<OpaqueStruct> {
    // [`HideData`] has private fields.
    Opaque::new(HideData::new())
}

pub fn run_opaque(opaque: Opaque<HideData>) -> String {
    // Opaque impl Deref trait.
    opaque.hide_data()
}
```

Dart: (test:'Double Call' frb_example/pure_dart/dart/lib/main.dart)
```dart

// (Arc counter = 1) Dart has full ownership.
var opaque = await api.createOpaque();

// (Arc counter = 2) increases immediately.
// (Arc counter = 1) after complete
String hideData1 = await api.runOpaque(opaque: opaque);

// (Arc counter = 2) increases immediately.
// (Arc counter = 1) after complete
String hideData2 = await api.runOpaque(opaque: opaque);

// (Arc counter = 0) opaque type is dropped (deallocated)
opaque.dispose();
```



### Case 5: Double call with dispose before complete.

Rust `api.rs`:
```rust,noplayground
pub use crate::data::HideData; // `pub` for bridge_generated.rs

pub fn create_opaque() -> Opaque<OpaqueStruct> {
    // [`HideData`] has private fields.
    Opaque::new(HideData::new())
}

pub fn run_opaque(opaque: Opaque<HideData>) -> String {
    // Opaque impl Deref trait.
    opaque.hide_data()
}
```

Dart:
```dart

// (Arc counter = 1) Dart has full ownership.
var opaque = await api.createOpaque();

// (Arc counter = 2) increases immediately. 
var unawait_task1 = api.runOpaque(opaque); *1

// (Arc counter = 3) increases immediately. 
var unawait_task2 = api.runOpaque(opaque); *2

// (Arc counter = 2) Rust has full ownership
opaque.dispose();

// (*1 is complete) (Arc counter = 1)
//
// Rust:
//
//`executes rust_call_example and counter decreases.`

// (*2 is complete) (Arc counter = 0) 
// opaque type is dropped (deallocated)
//
// Rust:
//
//`executes rust_call_example and drop opaque type.`
```


### Case 6: Dispose was not called (native).

Rust `api.rs`:
```rust,noplayground
pub use crate::data::HideData; // `pub` for bridge_generated.rs

pub fn create_opaque() -> Opaque<OpaqueStruct> {
    // [`HideData`] has private fields.
    Opaque::new(HideData::new())
}

pub fn run_opaque(opaque: Opaque<HideData>) -> String {
    // Opaque impl Deref trait.
    opaque.hide_data()
}
```

Dart:
```dart

// (Arc counter = 1) Dart has full ownership.
var opaque = await api.createOpaque();

// (Arc counter = 2) increases immediately. 
String hideData = await api.runOpaque(opaque);

// (Arc counter = 1)
//
// Rust:
//
// `executes rust_call_example and counter decreases.`

// memory of opaque types is not monitoring by dart and can accumulate.
// (Arc counter = 0) 
// opaque type is dropped (deallocated)
// 
// Dart:
//
// `the finalizer is guaranteed to be called before the program terminates.`
```

### Case 7: Dispose was not called (web).

Rust `api.rs`:
```rust,noplayground
pub use crate::data::HideData; // `pub` for bridge_generated.rs

pub fn create_opaque() -> Opaque<OpaqueStruct> {
    // [`HideData`] has private fields.
    Opaque::new(HideData::new())
}

pub fn run_opaque(opaque: Opaque<HideData>) -> String {
    // Opaque impl Deref trait.
    opaque.hide_data()
}
```

Dart:
```dart

// (Arc counter = 1) Dart has full ownership.
var opaque = await api.createOpaque(); 

// (Arc counter = 2) increases immediately. 
String hideData = await api.rustOpaque(opaque);

// (Arc counter = 1)
//
// Rust:
//
//`executes rust_call_example and counter decreases.`

// memory of opaque types is not monitoring by Dart and can accumulate.
// (Arc count can be 0 or 1) don't count on automatic clearing.
//
// Dart:
//
//`the finalizer is NOT guaranteed to be called before the program terminates.`
```

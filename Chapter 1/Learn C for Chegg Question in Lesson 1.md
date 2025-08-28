# C Control Flow

> A **C program** is a **finite, ordered sequence** of **explicit statements**:

$$
S = \{s_1, s_2, \dots, s_n\}
$$

> Each statement $s \in S$ denotes a (partial) transformation of the program’s **abstract machine state**:

$$
s : \text{State} \rightharpoonup \text{State}
$$

> where **State** includes:
>
 * memory (objects, call stack)
 * registers
 * control position

> The partial function arrow ($\rightharpoonup$) reflects that execution may not terminate (e.g. infinite loops) or may be undefined.

**Important:** All guarantees below hold **only for defined behavior** per the C standard. If a program exhibits *undefined behavior (UB)*, **the standard imposes no requirements whatsoever** on execution.

##  Control-flow Model

* **Sequential control flow:** By default, execution proceeds in order. If:

$$
(s_i, s_{i+1}) \in R,\quad R \subseteq S \times S
$$

Then $R$ is the **control-flow relation**.

* Programs can be modeled as **control-flow graphs (CFGs)**:

```math
G = (S, R)
```

Where:

* Nodes = statements or basic blocks

* Edges = valid control transfers (fall-through, branches, jumps, returns)[^2]

* **Compile-time control flow:** Constructs like `_Generic` alter control paths *during translation*, yielding distinct CFGs per instantiation.

## Distinctive Features and Caveats

## 1. **Sequencing and Evaluation Order**

C defines several categories that govern how the evaluation of expressions and the application of side effects are ordered. These distinctions are essential for determining whether an expression's semantics are defined, undefined, or dependent on the implementation.

### • **Unspecified Behavior**

> The **order of evaluation** between subexpressions is **not specified** by the standard.
> The compiler is free to choose any order, but it is **not required to document or be consistent across translation units**.

**Example (unspecified order of function arguments):**

```c
#include <stdio.h>

int add(int a, int b) {
    return a + b;
}

int main(void) {
    int i = 1;
    int x = add(i++, i);
    printf("%d\n", x); // Result may differ across compilers
}
```

This expression modifies and accesses `i` in the same statement without a sequence between them. The order in which `i++` and `i` are evaluated is unspecified, potentially producing different values passed to `add(...)`.



### • **Unsequenced Behavior**

> There is **no defined sequencing** between operations.
> If two subexpressions both modify the same object (or one modifies and the other accesses it) and are unsequenced relative to each other, the behavior is **undefined**.

**Example (undefined behavior due to unsequenced modifications):**

```c
int i = 1;
int x = i++ + ++i; // Undefined behavior
```

Both `i++` and `++i` attempt to modify `i`, and there is no sequence point between them. The result is undefined — the compiler is not constrained in how it compiles or executes this code.



### • **Implementation-Defined Behavior**

> The implementation must choose a behavior and **document it**.
> The behavior is consistent **within the same platform and compiler configuration**, but **may differ across systems**.

**Example (signedness of `char`):**

```c
char c = (char)255;
printf("%d\n", c); // May print -1 or 255 depending on platform
```

Whether `char` is treated as signed or unsigned is implementation-defined. The result is predictable only if the implementation's documentation is consulted.



### Caution on Side Effects and Evaluation Order

Expressions that rely on the order in which side effects occur — especially when modifying and reading the same object — can result in either unspecified behavior or undefined behavior.

**Avoid expressions such as:**

```c
int i = 1;
int x = i++ + i; // Unspecified
int y = i++ + ++i; // Undefined
```

**Preferred alternative:**

```c
int i = 1;
int a = i++;
int b = i;
int x = a + b; // Well-defined
```

Separating modifications from evaluations ensures defined, portable behavior and simplifies reasoning about program state.


### 2. **Undefined Behavior Dominates Semantics**

Operations like:

* Signed overflow
* Out-of-bounds access
* Unsequenced modifications

...invoke undefined behaviour. After undefined behaviour, **all guarantees are off** — compilers may emit any code.

**Example:**

```c
i = i++;
```

---

### 3. **Non-Local Control Flow**

Beyond standard jumps (`goto`, `break`, etc.), C also supports:

* `setjmp` / `longjmp`
* Asynchronous signal handlers

These can bypass normal stack unwinding and skip cleanup[^3].

---

### 4. **Compiler Transformations vs Abstract Machine**

Compiler optimizations like:

* Inlining
* Tail-call elimination
* Instruction reordering

...may alter the generated CFG.
But they **must preserve observable behavior** under defined semantics.

### 5. **Concurrency and Memory Model (C11)**

* Introduces **atomic operations**, **memory orderings**, and **sequence points**
* Defines **happens-before** relations
* **Data races** (unsynchronized access to shared memory) = UB[^4]

### 6. **Implementation and ABI Specifics**

* Endianness
* Calling conventions
* Stack alignment
* Object lifetimes via storage class specifiers

These affect how the abstract machine maps to hardware behavior.

### 7. **Variable-Length Arrays (VLAs)**

* Bounds only known at runtime
* Affects stack allocation and deallocation
* Introduces **runtime-dependent control flow**

## Control Structures — Minimal Orthogonal Set

| Category  | Keywords                              | Control-flow effect                         |
| --------- | ------------------------------------- | ------------------------------------------- |
| Selection | `if`, `else`, `switch`, `_Generic`    | Branch edges $(s\_i, s\_j), (s\_i, s\_k)$ |
| Iteration | `while`, `for`, `do`                  | Loops / back edges $(s\_j, s\_i)$         |
| Jumps     | `break`, `continue`, `goto`, `return` | Unconditional edges $(s\_i, s\_k)$        |
| Non-local | `setjmp`, `longjmp`, signals          | Cross-frame control jumps                   |

## Functions, Calls, and the Stack

A **function** is a named subgraph:

$$
f : \text{Input}_f \rightharpoonup \text{Output}_f
$$

### Function Call Behavior

* Creates a new **activation record** (also called a **stack frame**), representing the function’s dynamic state
  → This includes:

  * Parameter bindings (values passed to the function)
  * Storage for local (automatic) variables
  * Return address (to resume execution after return)
  * Saved machine registers (as mandated by the calling convention)

* Follows the standard **call/return discipline**
  → Control transfers to the function body, and returns to the caller upon completion, restoring the previous control context and deallocating the activation record.

**Exceptions:**

* Tail-call optimization (may eliminate frames)
* `setjmp` / `longjmp` (may skip frames entirely)
* Signal handlers (may asynchronously interrupt execution)
* Storage class (`auto`, `static`, etc.) affects lifetime/visibility

## Control-Flow Graph

The CFG $G = (S, R)$ must respect:

* C sequencing & evaluation rules
* Undefined behaviour invalidation (e.g., edges relying on UB are removed)
* The memory model (in concurrent contexts)
* Implementation-defined behavior (e.g. call order)

## Short Examples

### 1. **Unsequenced Modification — Undefined Behavior**

```c
int i = 1;
int x = i + i++; // undefined
```

> `i` is both **read** (for `i`) and **modified** (via `i++`) in a single expression with **no sequence** constraining the order.
> There is **no defined evaluation order** between the operands of `+`, and **modifying and accessing the same object unsequenced** results in **undefined behavior**.

#### Abstract Execution Model:

* Let `e1 = i` and `e2 = i++`
* Evaluation: `x = e1 + e2`, with `(e1, e2)` unsequenced
* Both `e1` and `e2` reference the same variable `i`, with one causing a write → violates sequencing constraints

---

### 2. **Implementation-Defined vs Undefined Evaluation**

```c
int result = f() + g(); // order of f(), g() is implementation-defined
```

> The **order in which `f()` and `g()` are called** is **implementation-defined**: the compiler must pick one, document it, and be consistent.
> However, **if either function has side effects**, relying on this order compromises portability.

#### Abstract Execution Model:

* Let `f() ⇒ x`, `g() ⇒ y`
* Expression becomes `x + y`, where calls to `f()` and `g()` are **sequenced only relative to `+`**, not each other
* Concrete order (`f` before `g`, or vice versa) is chosen per implementation

---

```c
int i = 5;
int bad = i++ + ++i; // UB
```

> `i` is modified **twice** (`i++`, `++i`) in a single expression with **no intervening sequence**.
> This is **unsequenced modification**, which causes **undefined behavior**.

#### Abstract Execution Model:

* Two writes to `i`: one post-increment (`i++`), one pre-increment (`++i`)
* No sequence determines which applies first → results in non-deterministic or invalid machine state

---

### 3. **Non-local Jump Skipping Cleanup**

```c
#include <setjmp.h>
#include <stdlib.h>

jmp_buf env;

int main(void) {
    char *buffer = malloc(1000);

    if (setjmp(env) == 0) {
        longjmp(env, 42); // Skips malloc'd cleanup
    }

    free(buffer); // May be skipped!
    return 0;
}
```

> `setjmp` saves the current **execution context**, including stack position and register state.
> `longjmp` restores that context non-locally — **bypassing control flow**, including intervening calls and cleanups.
> Here, `free(buffer)` is skipped because `longjmp` transfers control past its sequencing point.

#### Abstract Execution Model:

* `setjmp(env)` stores a snapshot of the current control state: `State₀`
* `longjmp(env, 42)` jumps directly to `State₀`, **without executing any statements in between**
* Resource management (e.g. `free`) must be done explicitly, or use mechanisms like `goto cleanup` or `defer` (in extensions)

---

### 4. **Atomic Synchronization (C11)**

```c
#include <stdatomic.h>

atomic_int data = 0, flag = 0;

// Thread 1
void producer(void) {
    atomic_store_explicit(&data, 42, memory_order_relaxed);
    atomic_store_explicit(&flag, 1, memory_order_release);
}

// Thread 2
void consumer(void) {
    while (atomic_load_explicit(&flag, memory_order_acquire) == 0);
    int value = atomic_load_explicit(&data, memory_order_relaxed);
}
```

> This example uses **C11 atomics** to coordinate memory access between threads.
> The `flag` variable acts as a **synchronization point** — a **release-acquire pair** establishes a **happens-before** relation.

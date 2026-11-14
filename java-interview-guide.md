# Java Interview Guide (9+ Years Experience)

> A complete, Java-only interview playbook: concepts → gotchas → coding problems → production scenarios.
> Every question uses short, plain sentences: **what** it is, **why** it exists, **how** it works, and what the interviewer will probe next.
> Each answer ends with a **Remember** line; revise those the night before the interview.
> Framework topics (Spring, Kafka, Redis, SQL, Docker, Kubernetes, AWS) live in their own guides in this folder.

---

## Table of Contents

1. [Core Java Fundamentals](#1-core-java-fundamentals)
2. [Collections Framework (Deep Dive)](#2-collections-framework-deep-dive)
3. [Java 8 → Java 21+ Features](#3-java-8--java-21-features)
4. [Exception Handling](#4-exception-handling)
5. [JVM, Memory & Garbage Collection](#5-jvm-memory--garbage-collection)
6. [Multithreading & Concurrency](#6-multithreading--concurrency)
7. [Design Patterns & Design Principles](#7-design-patterns--design-principles)
8. [Generics, Annotations, Reflection, I/O, Serialization & Date-Time](#8-generics-annotations-reflection-io-serialization--date-time)
9. [Unit Testing in Java (JUnit 5 & Mockito)](#9-unit-testing-in-java-junit-5--mockito)
10. [Production Debugging & Scenario-Based Questions](#10-production-debugging--scenario-based-questions)
11. [Java Coding Round — Problems You Will Be Asked](#11-java-coding-round--problems-you-will-be-asked)
12. [Tricky Output & Gotcha Questions (What Does This Print?)](#12-tricky-output--gotcha-questions-what-does-this-print)
13. [Rapid-Fire Round & Last-Minute Cheat Sheet](#13-rapid-fire-round--last-minute-cheat-sheet)

---

# 1. Core Java Fundamentals

### Q1. Explain the four OOP pillars with a real-world example.

**Answer:**

| Pillar | Meaning | Real-world (Java / Spring Boot) example |
|---|---|---|
| **Encapsulation** | Hide state, expose behaviour | `@Entity` with private fields + getters; `@Value` fields injected privately |
| **Inheritance** | Reuse via "is-a" | `MyException extends RuntimeException`; `JpaRepository extends CrudRepository` |
| **Polymorphism** | One interface, many implementations | `PaymentService` interface with `UpiPayment`, `CardPayment` beans, chosen via `@Qualifier` |
| **Abstraction** | Expose *what*, hide *how* | `JdbcTemplate` hides connection handling; you just call `query()` |

**In plain words (one `OrderService` walk-through):**
- *Encapsulation* — `Order` keeps `status` private and only exposes `order.markPaid()`, which checks the state is legal. Nobody can set `status = "PAID"` from outside, so invariants can't be broken.
- *Inheritance* — a `NotFoundException` **is a** `RuntimeException`, so it automatically gets message/cause/stack-trace behaviour without rewriting it.
- *Polymorphism* — the controller calls `paymentService.pay(order)` against the interface; at runtime Spring injects `UpiPayment` or `CardPayment`. Adding `WalletPayment` needs **no change** in the caller (Open/Closed principle).
- *Abstraction* — callers depend on *what* a `PaymentService` does, never on *how* (HTTP call, SDK, retry logic).

**Senior-level add-on:** "In practice I favour **composition over inheritance** — Spring itself is built on composition (beans wired into beans), which is why it stays testable. Deep inheritance trees break Liskov and make mocking painful."

**Remember:** **E-I-P-A** = *Every Interview Prefers Abstraction.*

---

### Q2. `equals()` and `hashCode()` — the contract, and what breaks if you violate it.

**Answer:**

**Contract:**
1. If `a.equals(b)` is true → `a.hashCode() == b.hashCode()` **must** be true.
2. Equal hashcodes do **not** imply equality (collisions are legal).
3. `equals` must be reflexive, symmetric, transitive, consistent, and `x.equals(null) == false`.

**What breaks:**
- Override `equals` but not `hashCode` → object goes into a `HashMap` bucket A, lookup computes bucket B → **you can't find your own key** (silent data loss).
- Use a **mutable field** in `hashCode` and then mutate it → the object is lost in the map forever.

```java
public final class Money {
    private final String currency;
    private final BigDecimal amount;

    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Money m)) return false;      // pattern matching, Java 16+
        return currency.equals(m.currency) && amount.compareTo(m.amount) == 0;
    }
    @Override public int hashCode() { return Objects.hash(currency, amount.stripTrailingZeros()); }
}
```

**JPA gotcha (asked a lot at 9 YOE):** Never use the auto-generated `@Id` alone in `equals`/`hashCode`. Before `persist()` the id is `null`, after flush it changes → the entity behaves like two different objects inside a `HashSet`. Use a **business key** or a **UUID assigned in the constructor**.

**Remember:** *Equal objects must share a hashcode; mutable fields must never enter one.*

---

### Q3. Why are Strings immutable? Explain the String pool.

**Answer:**
Reasons for immutability:
1. **Security** — a file path or SQL string can't be mutated after a security check (TOCTOU attack).
2. **Thread safety** — shareable without synchronization.
3. **Caching of hashCode** — computed once, makes `String` the perfect `HashMap` key.
4. **String pool interning** — safe sharing is only possible if nobody can mutate.

**Pool behaviour:**
```java
String a = "java";              // pool
String b = "java";              // same pooled reference
String c = new String("java");  // heap, new object
String d = c.intern();          // back to pool reference

a == b   // true
a == c   // false
a == d   // true
a.equals(c) // true
```
Since Java 7 the pool lives in the **heap** (not PermGen), so it is garbage-collectable.

**Concatenation:** `"a" + "b"` on literals is folded at compile time. In a loop, `s += x` creates a new `StringBuilder` **per iteration** → O(n²). Use an explicit `StringBuilder`.

**Remember:** *Immutable → safe to share → safe to pool → safe to cache the hash.*

---

### Q4. `String` vs `StringBuilder` vs `StringBuffer`.

| | String | StringBuilder | StringBuffer |
|---|---|---|---|
| Mutable | No | Yes | Yes |
| Thread-safe | Yes (immutable) | No | Yes (synchronized) |
| Speed | Slow for edits | Fastest | Slower (lock overhead) |
| Use when | Keys, constants | 99% of concatenation | Legacy shared buffer |

**Why the three exist:** `String` is immutable, so every "change" (`s = s + "x"`) allocates a brand-new object and copies the old characters. `StringBuilder` keeps an internal resizable `char`/`byte` array and appends **in place**, so building a big string in a loop is O(n) instead of O(n²). `StringBuffer` is the same thing from Java 1.0 with every method `synchronized` — the lock is pure overhead because a half-built string is almost never shared between threads (build it in a local variable, then share the finished `String`).

```java
StringBuilder sb = new StringBuilder(256);       // pre-size if you roughly know the length
for (Order o : orders) sb.append(o.getId()).append(',');
if (sb.length() > 0) sb.setLength(sb.length() - 1);   // trim last comma
String csv = sb.toString();
```
**Follow-up probe:** "Does `String s = a + b + c;` need a `StringBuilder`?" — No, a single expression is compiled efficiently (`StringBuilder` pre-Java 9, `invokedynamic` `StringConcatFactory` since 9). The problem is only concatenation **inside loops**. Also `String.join(",", list)` or `Collectors.joining` is the cleanest option.

**Remember:** *Builder for speed, Buffer for safety, String for stability.*

---

### Q5. How do you create a truly immutable class?

**Answer — the 5 rules:**
1. Class `final` (or all constructors private + static factory).
2. All fields `private final`.
3. No setters, no mutating methods.
4. **Defensive copy in the constructor** for mutable inputs (`List`, `Date`, arrays).
5. **Defensive copy in getters** — or return `List.copyOf(...)` / `Collections.unmodifiableList(...)`.

```java
public final class Order {
    private final String id;
    private final List<String> items;

    public Order(String id, List<String> items) {
        this.id = id;
        this.items = List.copyOf(items);        // rule 4
    }
    public List<String> getItems() { return items; }   // already unmodifiable
}
```

**Java 16+ shortcut:** `record Order(String id, List<String> items) {}` — but records are only *shallowly* immutable, so still copy in a compact constructor:
```java
record Order(String id, List<String> items) {
    Order { items = List.copyOf(items); }
}
```

**Remember:** *Final class, final fields, no setters, copy in, copy out.*

---

### Q6. `==` vs `equals()` vs `Objects.equals()` vs `compareTo()`.

- `==` → reference identity (or primitive value comparison).
- `equals()` → logical equality; NPE if left side is null.
- `Objects.equals(a, b)` → null-safe equals.
- `compareTo()` → ordering; must be **consistent with equals** for `TreeSet`/`TreeMap` to behave.

**Classic trap:** `BigDecimal("1.0").equals(new BigDecimal("1.00"))` is **false** (scale differs) but `compareTo` returns 0. In a `TreeSet` they'd be duplicates; in a `HashSet` they'd be two entries. Always compare money with `compareTo`.

**Integer caching trap:**
```java
Integer a = 127, b = 127;   a == b   // true  (cached -128..127)
Integer c = 128, d = 128;   c == d   // false (new objects)
```

**Remember:** *`==` asks "same object?", `equals` asks "same value?", `compareTo` asks "which comes first?"*

---

### Q7. Method overloading vs overriding.

| | Overloading | Overriding |
|---|---|---|
| When resolved | **Compile time** (static binding) | **Runtime** (dynamic dispatch) |
| Signature | Must differ (params) | Must be identical |
| Return type | Can differ | Same or covariant |
| Access | Any | Cannot reduce visibility |
| Exceptions | Any | Cannot add broader **checked** exceptions |
| `static`/`private`/`final` | Can be overloaded | Cannot be overridden (static = *hidden*) |

**Example:**
```java
class Animal { String sound() { return "..."; } }
class Dog extends Animal { @Override String sound() { return "Woof"; } }

Animal a = new Dog();
a.sound();                 // "Woof" — OVERRIDING: decided at runtime by the real object type (Dog)

void print(Object o) { }   void print(String s) { }
Object x = "hi";
print(x);                  // calls print(Object) — OVERLOADING: decided at compile time by the declared type
```
Rule of thumb: **overloading looks at the variable's declared type; overriding looks at the object's actual type.** Always put `@Override` so the compiler catches a signature typo that would silently create an overload instead.

**Trap:** `null` passed to overloaded `f(String)` and `f(Object)` → picks the **most specific** type, `String`. Add `f(Integer)` too and it becomes ambiguous → compile error.

**Remember:** *Overloading = compile-time cleverness; Overriding = runtime behaviour.*

---

### Q8. Abstract class vs interface — which do you pick in 2024+?

| | Abstract class | Interface |
|---|---|---|
| Multiple inheritance | No | Yes |
| State (fields) | Yes | Only `static final` constants |
| Constructors | Yes | No |
| Methods | abstract, concrete | abstract, `default`, `static`, `private` |
| Purpose | Shared **identity + state** | Shared **capability / contract** |

**How I choose:** interface for the contract (so it's mockable and multiple-implementable), abstract class only when several implementations genuinely share **state and template logic** (Template Method pattern). Spring is interface-first — that's what allows JDK dynamic proxies for `@Transactional`.

**Diamond problem with defaults:** if two interfaces declare the same `default` method, the implementing class **must** override it and can disambiguate with `InterfaceA.super.method()`.

**Remember:** *Interface = "can do"; abstract class = "is a, partially built".*

---

### Q9. Static vs instance initialization order.

Order when a class is first loaded and instantiated:
1. Static variables + static blocks (once, in declaration order)
2. Instance variables + instance init blocks
3. Constructor

With inheritance: **parent static → child static → parent instance/ctor → child instance/ctor.**

**Why it works this way:** *static* members belong to the **class**, so they run once when the JVM loads/initializes the class (first `new`, first static call, or `Class.forName`). *Instance* members belong to each **object**, so they run on every `new`. A child object can't be built before the parent part exists, hence the parent runs first at both levels.

```java
class Parent {
    static { System.out.println("1 Parent static"); }
    { System.out.println("3 Parent instance block"); }
    Parent() { System.out.println("4 Parent ctor"); }
}
class Child extends Parent {
    static { System.out.println("2 Child static"); }
    { System.out.println("5 Child instance block"); }
    Child() { System.out.println("6 Child ctor"); }
}
new Child();   // prints 1,2,3,4,5,6 — a second new Child() prints only 3,4,5,6
```
**Classic trap:** calling an overridable method from the parent constructor — it runs the **child's** override *before* the child's fields are initialized (they're still `null`/`0`). Never call overridable methods from constructors.

**Remember:** *Static once, top-down; instance every time, parent-first.*

---

### Q10. What is `final`, `finally`, `finalize`?

- `final` — non-reassignable variable / non-overridable method / non-extendable class.
- `finally` — block that always runs (except `System.exit`, JVM crash, or infinite loop). Runs **before** the method returns; a `return` inside `finally` swallows exceptions — never do it.
- `finalize()` — **deprecated since Java 9, removed in 18**. Use `try-with-resources` + `AutoCloseable`, or `Cleaner`.

**Why they matter (with examples):**
- `final` on a *reference* freezes the variable, **not** the object: `final List<String> l = new ArrayList<>(); l.add("x");` is legal. `final` methods stop subclasses changing critical behaviour; `final` classes (e.g. `String`) can't be subclassed, which is what makes them safe to rely on.
- `finally` is for guaranteed cleanup (closing a connection, releasing a lock). Prefer try-with-resources, which does this for you. The `return`-in-`finally` trap: `try { throw new RuntimeException(); } finally { return 1; }` returns `1` and the exception silently vanishes.
- `finalize()` was a GC callback — unpredictable timing (maybe never), slowed GC, and could "resurrect" objects, so it was removed. Today: `AutoCloseable` for deterministic cleanup; `java.lang.ref.Cleaner` as a safety net for native resources.

**Remember:** *final = constant, finally = cleanup, finalize = forget it.*

---

### Q11. Pass by value or pass by reference in Java?

Java is **always pass by value** — but for objects, the *value copied is the reference*. So you can mutate the object's state, but reassigning the parameter has no effect outside.

```java
void mutate(List<String> l) { l.add("x"); }       // caller sees "x"
void reassign(List<String> l) { l = new ArrayList<>(); } // caller sees nothing
```

**Step by step:** when you call `mutate(myList)`, Java copies the *address* held in `myList` into the parameter `l`. Two variables now point to the **same** object, so `l.add("x")` is visible to the caller. In `reassign`, `l = new ArrayList<>()` only repoints the *local copy* to a new object; the caller's variable still holds the old address. (True pass-by-reference, as in C++ `&`, would let the callee repoint the caller's variable — Java can't.)

**Interview probe:** "Can a `swap(a, b)` method for two `Integer`s work in Java?" — No, for exactly this reason. Primitives are copied by value; `String`/wrappers are immutable, so you can't even mutate them through the copy.

**Remember:** *Java copies the remote control, not the TV.*

---

### Q12. Shallow vs deep copy — how do you deep copy in production?

- **Shallow:** new outer object, shared inner references (`clone()` default, copy constructor).
- **Deep:** whole graph duplicated.

Production approaches (best → worst):
1. **Copy constructor / builder** that recursively copies — explicit, fast, type-safe.
2. **Records + `List.copyOf`** for value objects.
3. **Serialization round-trip** (Jackson `objectMapper.readValue(writeValueAsString(o), T.class)`) — easy but slow and drops transient state; fine for tests/DTOs.
4. `Cloneable`/`clone()` — broken design (protected, no constructor call, shallow by default). Avoid.

**Why it matters:** if two objects share an inner `List`/`Address`, changing it through one silently changes the other — a nasty aliasing bug (e.g. a "snapshot" of an order that changes later).

```java
class Order {
    List<Item> items; Address shipTo;
    Order(Order o) {                                   // deep copy constructor
        this.items  = o.items.stream().map(Item::new).toList(); // copy each element, not just the list
        this.shipTo = new Address(o.shipTo);
    }
}
```
**Pitfall:** `new ArrayList<>(other.items)` is only a shallow copy of the list — the `Item` objects inside are still shared. Deep copy must recurse down to immutable leaves (`String`, `Integer`, records of immutables need no copying). For JPA entities, don't deep-copy via serialization; map to a DTO instead.

**Remember:** *Prefer a copy constructor; Cloneable is a historical mistake.*

---

### Q13. What are Records, and when do you use them?

Java 16+. A record is a **transparent carrier for immutable data**: final class, final fields, canonical constructor, accessors, `equals`, `hashCode`, `toString` auto-generated.

```java
public record CustomerDto(String id, String name, Address address) {
    public CustomerDto {                       // compact constructor: validation
        Objects.requireNonNull(id);
        if (name.isBlank()) throw new IllegalArgumentException("name required");
    }
    public String initials() { return name.substring(0, 1); }  // extra behaviour allowed
}
```
**Use for:** DTOs, API request/response, events, value objects, map keys, `@ConfigurationProperties` (Spring Boot 3 supports record-based binding).
**Don't use for:** JPA `@Entity` (Hibernate needs a no-arg ctor and mutable proxies), or anything needing inheritance.

**Remember:** *Records = DTOs with zero boilerplate; never entities.*

---

### Q14. `Sealed` classes and pattern matching.

```java
public sealed interface PaymentEvent permits Authorized, Captured, Failed {}
record Authorized(String txnId, BigDecimal amount) implements PaymentEvent {}
record Captured(String txnId) implements PaymentEvent {}
record Failed(String txnId, String reason) implements PaymentEvent {}

String describe(PaymentEvent e) {
    return switch (e) {                                  // exhaustive, no default needed
        case Authorized a -> "Auth " + a.amount();
        case Captured c   -> "Captured " + c.txnId();
        case Failed f     -> "Failed: " + f.reason();
    };
}
```
**Value:** the compiler enforces exhaustiveness — add a new event type and every `switch` fails to compile instead of failing at runtime. Perfect for domain events and state machines.

**Remember:** *Sealed = closed hierarchy = compiler-checked switch.*

---

### Q15. JDK vs JRE vs JVM — and how does Java achieve platform independence?

**Say this first:** "I compile once to bytecode; each OS has its own JVM that turns bytecode into native instructions. The *bytecode* is portable, the *JVM* is not."

**Answer:**

| Piece | What it is | Contains |
|---|---|---|
| **JVM** | The virtual machine that loads and runs bytecode | Class loader, bytecode verifier, interpreter, JIT compiler, GC |
| **JRE** | What you need to *run* Java programs | JVM + core class library (`java.base` etc.) |
| **JDK** | What you need to *build* Java programs | JRE + tools: `javac`, `jar`, `javadoc`, `jlink`, `jshell`, `jcmd`, `jfr` |

**Why it exists:** C/C++ compile to machine code for one CPU/OS. Java adds a middle step — **bytecode** (`.class` files, an instruction set for an imaginary machine) — so the same file runs anywhere a JVM exists. This is "Write Once, Run Anywhere" (WORA).

**How it works, step by step:**
1. `javac Order.java` → `Order.class` (bytecode).
2. `java Order` starts a JVM process.
3. The **class loader** finds and loads the `.class` file.
4. The **verifier** checks the bytecode is well-formed and safe (no stack corruption, no illegal casts).
5. The **interpreter** starts executing; the **JIT compiler** turns hot methods into native machine code at runtime.

**Why the language itself is portable:** the Java Language Spec fixes the size and behaviour of types — `int` is always 32 bits, `long` 64 bits, `char` is UTF-16, floating point follows IEEE 754. In C, `int` size depends on the platform.

**Things that are *not* automatically portable (probe areas):**
- File separators and line endings (use `Path`, `File.separator`, `System.lineSeparator()`).
- Default charset — before **Java 18** it came from the OS; **JEP 400 (Java 18)** made UTF-8 the default. Always pass a charset explicitly.
- JNI / native libraries (`.dll` vs `.so`), process/OS-specific commands, time zone and locale defaults.

**Version facts worth knowing:**
- JVM is a *specification*; implementations include HotSpot (default in OpenJDK/Oracle), Eclipse OpenJ9, GraalVM.
- Since **Java 9** (modules) you can use `jlink` to build a small custom runtime image; since **Java 11** there is no separate JRE download from most vendors — you ship a JDK or a jlink image.
- Class-file major versions: Java 8 = 52, 11 = 55, 17 = 61, 21 = 65. Running newer bytecode on an older JVM gives `UnsupportedClassVersionError`. Use `javac --release 11` to target an older API level safely (better than `-source/-target`, which does not check the API).
- LTS releases: 8, 11, 17, 21.

**Pitfalls / follow-up probes:**
- "Is Java interpreted or compiled?" — Both. Compiled to bytecode, then interpreted and JIT-compiled.
- "Is the JVM platform independent?" — No. Each OS/CPU needs its own JVM build.
- "Can other languages run on the JVM?" — Yes (Kotlin, Scala, Groovy) because the JVM only cares about bytecode.

**Remember:** *JDK ⊃ JRE ⊃ JVM — bytecode is portable, the JVM is the per-OS translator.*

---

### Q16. Primitives vs wrapper classes, autoboxing pitfalls, and the Integer cache.

**Say this first:** "Autoboxing is convenient but hides object creation, `==` surprises and `NullPointerException` on unboxing."

**Answer:**
Java has 8 **primitives** (`byte short int long float double char boolean`) — raw values, no methods, never `null`. Each has a **wrapper class** (`Byte … Boolean`) — a real object. Wrappers exist because generics, collections and `Optional` only work with objects, and because they can represent "absent" (`null`).

**Autoboxing** = compiler converts `int → Integer` by calling `Integer.valueOf(int)`. **Unboxing** = converts back by calling `intValue()`.

**Integer cache (the classic trap):**
`Integer.valueOf(i)` returns a **cached** object for values **-128 to 127**; outside that range it creates a new object.
```java
Integer a = 127, b = 127;
System.out.println(a == b);        // true  (same cached object)
Integer c = 128, d = 128;
System.out.println(c == d);        // false (two objects)
System.out.println(c.equals(d));   // true
```
| Wrapper | Cached range |
|---|---|
| `Byte`, `Short`, `Integer`, `Long` | -128 … 127 (`Integer` upper bound tunable with `-XX:AutoBoxCacheMax`) |
| `Character` | 0 … 127 |
| `Boolean` | `TRUE`, `FALSE` |
| `Float`, `Double` | **never** cached |

**Pitfalls:**
1. **NPE on unboxing:** `Map<String,Integer> m = ...; int n = m.get("missing");` → `NullPointerException`. Same for `flag ? someNullInteger : 0`.
2. **`==` on wrappers** compares references. Always use `equals` (or unbox deliberately). `Integer == int` is fine — it unboxes.
3. **Hidden allocation in loops:** `Long sum = 0L; for (...) sum += i;` creates a new `Long` per iteration. Use primitive `long`.
4. **`List.remove` overload:** `List<Integer> l; l.remove(1)` removes **index** 1; `l.remove(Integer.valueOf(1))` removes the **value**.
5. **Cross-type equals:** `Long.valueOf(1).equals(1)` is `false` (the `1` boxes to `Integer`). `Long x = 1;` does not compile; `Long x = 1L;` does.
6. **Memory/perf:** an `Integer` is ~16 bytes vs 4 for an `int`, and a `List<Integer>` is an array of pointers. For big numeric data use arrays, `IntStream`, or a primitive-collection library (fastutil, Eclipse Collections).
7. `new Integer(5)` is deprecated (for removal since Java 16) — use `Integer.valueOf(5)`.

**Remember:** *Wrappers: compare with `equals`, never unbox `null`, and keep them out of hot loops.*

---

### Q17. Integer overflow, float/double precision, widening and narrowing casts.

**Answer:**

**1. Integer overflow is silent.** `int` is 32-bit two's complement, so it wraps around.
```java
int max = Integer.MAX_VALUE;
System.out.println(max + 1);                  // -2147483648
long wrong = 24 * 60 * 60 * 1000 * 1000;      // int math first! = 500654080
long right = 24L * 60 * 60 * 1000 * 1000;     // 86400000000
Math.addExact(max, 1);                        // throws ArithmeticException (Java 8+)
Math.toIntExact(3_000_000_000L);              // throws ArithmeticException
```
Other gotchas: `Math.abs(Integer.MIN_VALUE)` is still negative; the midpoint `(lo + hi) / 2` can overflow — use `lo + (hi - lo) / 2` or `(lo + hi) >>> 1`; `%` takes the sign of the dividend (`-7 % 3 == -1`; use `Math.floorMod(-7, 3) == 2` for array indexes/hash buckets).

**2. Floating point is binary, not decimal.** `float`/`double` follow IEEE 754. Many decimals (0.1) have no exact binary form.
```java
System.out.println(0.1 + 0.2);            // 0.30000000000000004
System.out.println(0.1 + 0.2 == 0.3);     // false
Double.NaN == Double.NaN                  // false; use Double.isNaN(x)
0.0 == -0.0                               // true, but Double.valueOf(0.0).equals(-0.0) is false
1.0 / 0                                   // Infinity (no exception); 1 / 0 (ints) throws ArithmeticException
```
Compare with a tolerance (`Math.abs(a - b) < 1e-9`), and use `BigDecimal` for money.

**3. Widening (safe, implicit) vs narrowing (risky, explicit cast).**
- Widening: `byte → short → int → long → float → double`, `char → int`. No cast needed. Note `int → float` and `long → float/double` can **lose precision** (`(int)(float)16777217 == 16777216`).
- Narrowing needs a cast and can lose data:
```java
(byte) 200        // -56 (keeps low 8 bits)
(int) 3.99        // 3   (truncates toward zero, does not round)
(int) 1e20        // Integer.MAX_VALUE (saturates)
(int) Double.NaN  // 0
(char) 65         // 'A'
```
- **Compound assignment has a hidden cast:** `byte b = 10; b += 300;` compiles (it means `b = (byte)(b + 300)` → 54) but `b = b + 1;` does not.
- Arithmetic on `byte/short/char` is done in `int` (binary numeric promotion), which is why `byte c = a + b;` fails.
- Reference casts: upcast is implicit and safe; downcast is checked at runtime → `ClassCastException`. Guard with `instanceof` (pattern matching since Java 16).

**Remember:** *Ints wrap quietly, doubles lie about decimals, and narrowing casts just chop bits.*

---

### Q18. Why use `BigDecimal` for money, and what are its pitfalls?

**Say this first:** "Because `double` can't represent 0.1 exactly. But `BigDecimal` has its own traps: the constructor, `equals`, `divide` and rounding."

**Answer:**
`BigDecimal` = an arbitrary-precision **unscaled integer value** plus a **scale** (number of digits after the decimal point). `1.50` is unscaled `150`, scale `2`. It is **immutable**, so every operation returns a new object.

**Pitfall 1 — constructor from double:**
```java
new BigDecimal(0.1);          // 0.1000000000000000055511151231257827021181583404541015625
new BigDecimal("0.1");        // 0.1   (best: from String)
BigDecimal.valueOf(0.1);      // 0.1   (uses Double.toString — OK when you already have a double)
```

**Pitfall 2 — `equals` includes scale:**
```java
new BigDecimal("2.0").equals(new BigDecimal("2.00"));       // false
new BigDecimal("2.0").compareTo(new BigDecimal("2.00"));    // 0  → use compareTo
```
`hashCode` is consistent with `equals`, so `2.0` and `2.00` land in different `HashMap` buckets. Normalise with `stripTrailingZeros()` (note `new BigDecimal("100").stripTrailingZeros()` prints `1E+2`; use `toPlainString()`), or `setScale` to a fixed scale first.

**Pitfall 3 — `divide` can throw:**
```java
BigDecimal.ONE.divide(new BigDecimal("3"));                          // ArithmeticException: non-terminating decimal expansion
BigDecimal.ONE.divide(new BigDecimal("3"), 2, RoundingMode.HALF_UP); // 0.33
BigDecimal.ONE.divide(new BigDecimal("3"), MathContext.DECIMAL64);   // 16 significant digits
```

**Pitfall 4 — rounding:** `setScale(2)` with no `RoundingMode` throws if digits would be lost. Choose the mode on purpose:
| Mode | Behaviour | Typical use |
|---|---|---|
| `HALF_UP` | 2.5 → 3, -2.5 → -3 | Schoolbook, many tax rules |
| `HALF_EVEN` | 2.5 → 2, 3.5 → 4 | "Banker's rounding", reduces bias in sums |
| `DOWN` / `UP` | toward / away from zero | Truncation rules |
| `FLOOR` / `CEILING` | toward -∞ / +∞ | Min/max guarantees |

**Scale rules:** `add/subtract` → max of the two scales; `multiply` → sum of the scales; so repeated multiplications grow scale — round at defined points.

**More pitfalls:**
- Forgetting immutability: `price.add(tax);` alone does nothing — assign the result.
- Zero check: use `signum() == 0` or `compareTo(BigDecimal.ZERO) == 0`, not `equals(BigDecimal.ZERO)`.
- `toString()` may give scientific notation; `toPlainString()` never does.
- It is slower and heavier than `long`. Many systems store money as **minor units in a `long`** (paise/cents) plus a currency code.

**Remember:** *Build from String, compare with `compareTo`, divide with a scale and a RoundingMode.*

---

### Q19. Operator gotchas: short-circuit evaluation, ternary type promotion, compound assignment.

**Answer:**

**1. Short-circuit `&&` / `||` vs non-short-circuit `&` / `|`.**
`a && b` skips `b` if `a` is `false`; `a || b` skips `b` if `a` is `true`. `&` and `|` on booleans **always evaluate both**.
```java
if (s != null && s.length() > 0) { ... }     // safe: length() not called on null
if (s != null & s.length() > 0) { ... }      // NPE when s == null
if (check() | audit()) { ... }               // audit() always runs (side effect) — legal but a code smell
```
Short-circuiting is also what makes `x != 0 && 10 / x > 1` safe.

**2. Ternary `cond ? a : b` — the result type follows promotion rules, not "whatever I meant".**
```java
System.out.println(true ? 1 : 2.0);                // 1.0  (int promoted to double)
Object o = true ? Integer.valueOf(1) : Double.valueOf(2);   // 1.0  (both unbox, then promote)
Integer n = null;  int r = flag ? n : 0;           // NPE when flag is true (n unboxed)
System.out.println(false ? 1 : 'a');               // a    (constant 1 fits in char → type char)
int i = 1;  System.out.println(false ? i : 'a');   // 97   (non-constant int → type int)
```
Rule of thumb: if one operand is a primitive and the other a wrapper, the wrapper is **unboxed** — null → NPE. Avoid mixing types in a ternary.

**3. Compound assignment hides a cast.** `x op= y` means `x = (T)(x op y)`.
```java
byte b = 10;  b += 300;     // OK, b = (byte)310 = 54, silently truncated
short s = 1;  s = s + 1;    // compile error: int → short
```

**4. Pre/post increment and evaluation order.** Java evaluates left to right.
```java
int i = 5;  i = i++;           // i stays 5 (old value is assigned back)
int j = 5;  j = j++ + ++j;     // 5 + 7 = 12
```
Never write code like this in production; it is only an interview trap.

**5. String `+` with chars and null.**
```java
"" + 'a' + 'b'     // "ab"
'a' + 'b' + ""     // "195"  (char + char is int addition first)
String s = null;  s + "x"   // "nullx"
```

**6. `==` precedence:** `a & 1 == 0` parses as `a & (1 == 0)` → compile error; write `(a & 1) == 0`.

**Remember:** *`&&`/`||` stop early; ternary promotes types; `+=` casts silently.*

---

### Q20. Bitwise operators, tricks, and labeled `break` / `continue`.

**Answer:**

**Operators** (work on `int`/`long`, after promoting smaller types):
| Op | Meaning | Example |
|---|---|---|
| `&` `\|` `^` `~` | AND, OR, XOR, NOT | `5 & 3 = 1`, `5 ^ 3 = 6` |
| `<<` | left shift (×2 per step) | `1 << 4 = 16` |
| `>>` | **signed** right shift (keeps sign bit) | `-8 >> 1 = -4` |
| `>>>` | **unsigned** right shift (fills with 0) | `-8 >>> 1 = 2147483644` |

**Useful tricks:**
```java
boolean isPow2 = n > 0 && (n & (n - 1)) == 0;   // a power of two has exactly one bit set
int lowestBit  = n & -n;                         // isolate lowest set bit
flags |=  (1 << k);                              // set bit k
flags &= ~(1 << k);                              // clear bit k
flags ^=  (1 << k);                              // toggle bit k
boolean on = (flags >> k & 1) == 1;              // test bit k
int mid = (lo + hi) >>> 1;                       // overflow-safe midpoint
int bits = Integer.bitCount(n);                  // population count
int ub = someByte & 0xFF;                        // treat a signed byte as unsigned 0..255
```
Real use: permission masks, feature flags, `HashMap` index = `(n - 1) & hash` (works because the table size is a power of two), and the hash spread `h ^ (h >>> 16)`. For readable flag sets prefer `EnumSet`.

**Gotchas:**
- Shift distance is masked: for `int` only the low 5 bits count, so `1 << 32 == 1`; for `long` the low 6 bits.
- `byte b = -1; b >>>= 1;` still gives `-1`: `b` is promoted to int, shifted, then cast back to byte.
- There is no unsigned `int` type; use `Integer.toUnsignedString`, `Integer.compareUnsigned`, `Integer.divideUnsigned` (Java 8+).
- XOR swap (`a ^= b; b ^= a; a ^= b`) is a party trick — don't use it in real code.

**Labeled `break` / `continue`:** a label names a statement so you can exit or skip an **outer** loop from an inner one, without a flag variable.
```java
outer:
for (int i = 0; i < rows; i++) {
    for (int j = 0; j < cols; j++) {
        if (grid[i][j] == target) { found = true; break outer; }   // leaves both loops
        if (grid[i][j] < 0)       { continue outer; }              // next row
    }
}
```
You can also label a plain block and `break label;` out of it. `goto` is a reserved word but is **not** implemented. Use labels sparingly; extracting a method with `return` is usually clearer.

**Remember:** *`>>` keeps the sign, `>>>` doesn't; `n & (n-1)` clears the lowest bit; labels break out of outer loops.*

---

### Q21. `static` vs instance — and static nested vs inner vs local vs anonymous classes.

**Answer:**

**`static` = belongs to the class, not to any object.** One copy per class (per class loader), created when the class is initialised.
- A static method has no `this`, so it cannot touch instance fields directly.
- Static methods are **not polymorphic**: they are *hidden*, not overridden (resolved at compile time by the declared type).
- Static mutable state is global state → hard to test and a thread-safety risk.

**Four kinds of nested classes:**
| Kind | Declared | Holds hidden ref to outer instance? | Typical use |
|---|---|---|---|
| **Static nested** | `static class X` inside a class | **No** | Builders, `Map.Entry`, helper types |
| **Inner (member)** | `class X` inside a class (no `static`) | **Yes** (`Outer.this`) | Iterators that need the outer's state |
| **Local** | Inside a method/block | Yes (if in instance method) + captured locals | Rare; one-off helper |
| **Anonymous** | `new Iface() { ... }` inline | Yes (if in instance context) + captured locals | Quick one-off implementation (lambdas replaced most uses) |

```java
class Outer {
    private int x = 1;
    static class Nested { }                    // new Outer.Nested()
    class Inner { int read() { return x; } }   // needs an Outer: outer.new Inner()
}
Outer o = new Outer();
Outer.Inner in = o.new Inner();
```
- Local/anonymous classes can capture local variables only if they are **final or effectively final** (never reassigned).
- Since **Java 16**, inner classes may declare static members. Since Java 9, anonymous classes can use the diamond `<>`.

**The memory-leak risk:** an inner or anonymous class created in an instance context silently keeps a reference to the whole outer object (compiler field `this$0`). If the inner object lives long (registered listener, cache entry, task in a long-lived executor, thread), the outer object — and everything it references — **cannot be garbage collected**.
```java
class Screen {
    byte[] hugeBuffer = new byte[50_000_000];
    Runnable task() { return new Runnable() { public void run() { /* holds Screen */ } }; }
}
```
**Fix:** make it `static` nested (pass in only what it needs), or use a lambda that does not touch `this`. (Newer javac versions may omit an unused `this$0`, but don't rely on it.) Also avoid the old "double-brace initialisation" (`new ArrayList<>() {{ add(1); }}`) — it creates an anonymous subclass that captures the outer `this`.

**Rule of thumb:** make every nested class `static` unless it genuinely needs the outer instance.

**Remember:** *Inner classes carry an invisible `Outer.this` — static nested classes don't.*

---

### Q22. Enums in depth — fields, constant-specific bodies, `EnumMap`, singleton, `switch`.

**Say this first:** "A Java enum is a full class with a fixed set of instances created by the JVM — not just named integers."

**Answer:**
An `enum` is a special class: its constants are `public static final` instances, the constructor is implicitly `private`, it extends `java.lang.Enum` (so it **cannot extend another class**, but **can implement interfaces**).
```java
public enum Operation {
    ADD("+")  { public int apply(int a, int b) { return a + b; } },
    MUL("*")  { public int apply(int a, int b) { return a * b; } };   // constant-specific body

    private final String symbol;                 // fields + constructor
    Operation(String symbol) { this.symbol = symbol; }
    public abstract int apply(int a, int b);     // each constant must implement it
    public String symbol() { return symbol; }

    private static final Map<String, Operation> BY_SYMBOL = new HashMap<>();
    static { for (Operation o : values()) BY_SYMBOL.put(o.symbol, o); }   // lookup table
    public static Optional<Operation> of(String s) { return Optional.ofNullable(BY_SYMBOL.get(s)); }
}
```
- Built-ins: `values()` (returns a **new array copy each call**), `valueOf(String)` (throws `IllegalArgumentException` if unknown), `name()`, `ordinal()`, `compareTo` (by ordinal).
- Compare enums with `==` — safe, null-safe, and compile-time type checked.
- A constructor/instance initialiser cannot read the enum's own static fields (they are not initialised yet) — that is why the lookup map is filled in a `static` block after the constants.

**`EnumMap` and `EnumSet`:** specialised collections for enum keys. `EnumMap` is backed by an array indexed by `ordinal()`; `EnumSet` by a bit vector (a `long` for ≤ 64 constants). They are faster and lighter than `HashMap`/`HashSet`, and iterate in declaration order.
```java
Map<Day, List<Task>> byDay = new EnumMap<>(Day.class);
Set<Permission> perms = EnumSet.of(READ, WRITE);
```

**Enum singleton (Effective Java's recommended way):**
```java
public enum Registry { INSTANCE; public void register(String s) { ... } }
```
The JVM guarantees one instance, it is thread-safe via class initialisation, and it is immune to reflection (`Constructor.newInstance` on an enum is rejected) and serialisation attacks (enums serialise by name). Downside: cannot extend a class, and it is eagerly initialised on first use of the class.

**`switch` and enums:** `case ADD:` uses the bare constant name. With **switch expressions (Java 14+)** the compiler checks exhaustiveness for enums — no `default` needed, and adding a constant later breaks compilation (good!).
```java
int priority = switch (level) { case LOW -> 1; case MEDIUM -> 2; case HIGH -> 3; };
```

**Pitfalls / follow-up probes:**
- **Never persist `ordinal()`** (e.g. in a DB) — reordering or inserting a constant corrupts data. Persist `name()` or an explicit code field.
- Enums are implicitly `final` unless constants have bodies (then each is an anonymous subclass).
- Adding a constant to a public enum can break consumers' `switch` statements that have no `default`.

**Remember:** *Enum = fixed set of singletons with fields and behaviour; use `EnumMap`, `==`, and never persist the ordinal.*

---

### Q23. Interface `default`, `static` and `private` methods — and the diamond problem.

**Answer:**
**Why they exist:** before Java 8, adding a method to a published interface broke every implementor. **Default methods (Java 8)** let an interface ship a body, so the library can evolve — that is how `Collection.stream()`, `List.sort()`, `Map.forEach()` were added without breaking your `List` implementations.

| Feature | Since | Notes |
|---|---|---|
| `default` method | 8 | Has a body; inherited; can be overridden by implementors |
| `static` method | 8 | Called as `Iface.m()`; **not inherited** by implementing classes |
| `private` method | 9 | Shares code between default/static methods; invisible to implementors |
| Fields | — | Always `public static final` (constants) |

```java
interface Validator {
    boolean isValid(String s);
    default Validator and(Validator other) { return s -> isValid(s) && other.isValid(s); }
    static Validator notBlank() { return s -> s != null && !s.isBlank(); }
    private void log(String m) { /* helper shared by defaults (Java 9+) */ }
}
```

**The diamond problem:** a class inherits the **same default method signature** from two interfaces. Java resolves it with three rules:
1. **Class wins** — a method from the superclass (even abstract) beats any interface default.
2. **More specific interface wins** — a sub-interface's default beats its parent's.
3. Otherwise **compile error** — you must override and choose explicitly:
```java
interface A { default String hi() { return "A"; } }
interface B { default String hi() { return "B"; } }
class C implements A, B {
    @Override public String hi() { return A.super.hi() + B.super.hi(); }   // "AB"
}
```
Java still has **no multiple inheritance of state**, so the true diamond problem (conflicting fields/constructors) cannot happen.

**Pitfalls / follow-up probes:**
- A default method **cannot** override `equals`, `hashCode` or `toString` (class methods always win).
- Default methods cannot be `final` or `synchronized`, and cannot touch instance state — only call other interface methods.
- A sub-interface can **re-abstract** an inherited default by redeclaring it without a body.
- Interface vs abstract class: interfaces have no constructors or instance fields; use an abstract class when you need shared state.
- Adding a default method is *source/binary compatible*, but it can still clash with a method an implementor already declared.

**Remember:** *Default = evolve APIs; class beats interface, specific beats general, else override and call `X.super.m()`.*

---

### Q24. Access modifiers — including package-private and the `protected` nuances.

**Answer:**

| Modifier | Same class | Same package | Subclass (other pkg) | Everywhere |
|---|---|---|---|---|
| `private` | ✅ | ❌ | ❌ | ❌ |
| *(none)* package-private | ✅ | ✅ | ❌ | ❌ |
| `protected` | ✅ | ✅ | ✅ (with a catch) | ❌ |
| `public` | ✅ | ✅ | ✅ | ✅ |

- **Top-level** classes can only be `public` or package-private. Nested classes can use all four.
- **Package-private is the default and underrated:** it hides helpers inside a package without the ceremony of modules — a good default for implementation classes.
- Inside one top-level class, nested classes can access each other's `private` members.

**The `protected` catch:** outside the package, a subclass may access a protected member **only through a reference of its own type (or a subtype)** — not through a plain superclass reference.
```java
package a;  public class Base { protected void hook() { } }

package b;
public class Sub extends Base {
    void test() {
        hook();              // OK
        this.hook();         // OK
        new Sub().hook();    // OK
        new Base().hook();   // COMPILE ERROR — different package, via Base reference
    }
}
```
Also `protected` includes package access, so it is *wider* than package-private. Protected **constructors** can be called by `super(...)` but not by `new` from another package.

**Overriding rule:** an overriding method may **widen** visibility (`protected` → `public`) but never narrow it. `private` methods are not inherited, so they are not overridden — a same-named method in a subclass is a brand-new method.

**Modules (Java 9+):** `public` no longer means "everyone". A public type in a package that its module does not `exports` is inaccessible to other modules. Since **Java 17 (JEP 403)** the JDK's internal packages are strongly encapsulated — `setAccessible(true)` on `java.*` internals throws `InaccessibleObjectException` unless `--add-opens` is used.

**Design guidance:** start `private`, widen only when needed; prefer `public` API surface small; avoid `protected` fields (they are a public API for subclasses and break encapsulation).

**Remember:** *private < package < protected < public; protected outside the package works only through your own subclass type.*

---

### Q25. Methods of `java.lang.Object` — what does every class inherit, and which matter?

**Answer:**
Every class extends `Object`, so every object has these 11 methods:

| Method | What it does | Interview notes |
|---|---|---|
| `toString()` | Text form | Default is `ClassName@hex(hashCode)`; always override for logs |
| `equals(Object)` | Logical equality | Default is `==`; pair with `hashCode` |
| `hashCode()` | Hash for hash-based collections | Default is an identity hash (implementation-specific; not guaranteed to be the memory address) |
| `getClass()` | Runtime `Class<?>` object | `final`; returns the *actual* type, not the declared type |
| `clone()` | Field-by-field copy | `protected`; needs `Cloneable`; see the clone question |
| `finalize()` | Called by GC before reclaim | **Deprecated Java 9, deprecated for removal Java 18 (JEP 421)** — never use |
| `wait()` / `wait(ms)` / `notify()` / `notifyAll()` | Thread coordination on the object's monitor | `final`; must hold the monitor |

**`wait/notify` rules** (see the concurrency section for alternatives):
```java
synchronized (lock) {
    while (!condition) lock.wait();   // ALWAYS a loop: spurious wakeups + re-check
    // ... use the shared state
}
// producer: synchronized (lock) { condition = true; lock.notifyAll(); }
```
Calling them without owning the monitor throws `IllegalMonitorStateException`. `wait` **releases** the lock; `sleep` does not. Prefer `notifyAll` unless you are sure only one waiter can proceed.

**`getClass()` vs `instanceof` in `equals`:**
- `getClass() != o.getClass()` → strict: a subclass instance is never equal to a parent instance (keeps symmetry), but violates Liskov in frameworks that use proxies/subclasses.
- `o instanceof Money m` → allows subclasses; safe when the class is `final` (records, value types).

**`toString` good practice:** include identity-relevant fields, exclude secrets (passwords, tokens) and avoid recursive references (parent ↔ child) that cause `StackOverflowError`. `Objects.toString(o, "default")` is null-safe.

**Pitfalls / follow-up probes:**
- "Why are `wait/notify` on `Object` and not `Thread`?" — Because the monitor (lock) belongs to the *object*, and any object can be a lock.
- `System.identityHashCode(o)` returns the default identity hash even if `hashCode` is overridden.
- `hashCode()` may legitimately change between JVM runs — never persist it.

**Remember:** *Object gives you `toString/equals/hashCode/getClass/clone/finalize` + the `wait/notify` family — override the first three, avoid `clone` and `finalize`.*

---

### Q26. Why is `clone()` considered broken, and what are the alternatives?

**Say this first:** "`clone()` is an awkward, extralinguistic mechanism. I prefer a copy constructor or static factory."

**Answer:**
`Object.clone()` is `protected` and native. How it works:
1. Your class must implement the marker interface `Cloneable` (it has **no methods**). If not, `Object.clone()` throws `CloneNotSupportedException` (a *checked* exception).
2. `Object.clone()` allocates a new object of the **same runtime class** and copies every field bit-for-bit — **without calling any constructor**.
3. Result is a **shallow copy**: reference fields point to the same objects as the original.

```java
class Order implements Cloneable {
    private int id;
    private List<String> items = new ArrayList<>();

    @Override public Order clone() {                       // covariant return, widened to public
        try {
            Order copy = (Order) super.clone();            // always start with super.clone()
            copy.items = new ArrayList<>(items);           // manual deep copy of mutable fields
            return copy;
        } catch (CloneNotSupportedException e) { throw new AssertionError(e); }
    }
}
```

**Why it is problematic (Effective Java, Item 13):**
- `Cloneable` doesn't declare `clone()` — the marker changes the behaviour of a *different* class's protected method. Awkward contract.
- **No constructor runs**, so invariants established there are bypassed.
- **`final` fields** cannot be reassigned in `clone()`, so you cannot deep-copy a final mutable field.
- Every subclass must keep the chain `super.clone()`, or the copy has the wrong runtime type.
- A checked exception for a feature you have just implemented.
- Easy to get shallow when you wanted deep (silent shared state bugs).

**Exception:** **arrays** — `int[] b = a.clone();` is public, returns the right type, and is the idiomatic array copy (still shallow for object arrays).

**Alternatives:**
```java
public Order(Order other) {                         // copy constructor
    this.id = other.id;
    this.items = new ArrayList<>(other.items);
}
public static Order copyOf(Order o) { return new Order(o); }   // static factory
List<String> safe = List.copyOf(source);            // Java 10+ immutable copy
```
- Copy constructor/factory: explicit, works with `final` fields, can accept interface types (`new ArrayList<>(anyCollection)`).
- Immutable types / **records** need no copying at all.
- Builder with `toBuilder()` for "copy with a change".
- Serialisation-based or mapper-based deep copy only when graph is big (slow, has risks).

**Remember:** *`clone()` = shallow, constructor-less, marker-driven; use a copy constructor or just make the type immutable.*

---

### Q27. Marker interfaces vs annotations — when do you use which?

**Answer:**
A **marker interface** has no methods; it just tags a class (`Serializable`, `Cloneable`, `RandomAccess`, `EventListener`). An **annotation** (`@interface`) attaches metadata to classes, methods, fields, parameters, even type uses.

| | Marker interface | Annotation |
|---|---|---|
| Defines a **type** | ✅ you can write `void save(Serializable s)` | ❌ |
| Compile-time type checking | ✅ | ❌ (needs a processor) |
| Carries data (attributes) | ❌ | ✅ (`@Retry(times = 3)`) |
| Applies to methods/fields/params | ❌ (classes only) | ✅ |
| Inheritance by subclasses | Always | Only with `@Inherited` (classes only) |
| Added to a class you don't own | ❌ | via config/other mechanisms |

**Effective Java Item 41 rule:** use a marker **interface** if you want a *type* that callers can use in signatures; use an **annotation** if the mark is about elements other than classes or may need parameters/evolution.

**How annotations work:**
```java
@Retention(RetentionPolicy.RUNTIME)       // SOURCE (compiler only) | CLASS (in .class) | RUNTIME (reflection)
@Target(ElementType.METHOD)               // where it may be placed
@Documented
public @interface Timed { String name() default ""; int warnMs() default 500; }

for (Method m : service.getClass().getDeclaredMethods()) {
    if (m.isAnnotationPresent(Timed.class)) {
        Timed t = m.getAnnotation(Timed.class);   // read attributes reflectively
    }
}
```
- Annotation members may only be primitives, `String`, `Class`, enums, other annotations, or arrays of those; defaults must be constants.
- Three consumers: **the compiler** (`@Override`, `@FunctionalInterface`, `@SafeVarargs`), **annotation processors** at build time (Lombok, MapStruct), and **runtime reflection** (frameworks).
- Java 8 added `@Repeatable` and type annotations (`ElementType.TYPE_USE`); Java 9 added `@Deprecated(since=..., forRemoval=...)`.

**Real examples of marker use inside the JDK:** `ObjectOutputStream` checks `instanceof Serializable` (otherwise `NotSerializableException`); `Collections.binarySearch` picks an index-based or iterator-based algorithm based on `RandomAccess`.

**Pitfalls:** forgetting `RetentionPolicy.RUNTIME` (annotation invisible at runtime); annotations are not inherited by overriding methods or interfaces; reflection-based processing costs startup time (a reason for compile-time processing).

**Remember:** *Need a type? Marker interface. Need metadata or non-class targets? Annotation.*

---

### Q28. Java serialization — `transient`, `serialVersionUID`, why it is dangerous, and alternatives.

**Answer:**
**Serialization** converts an object graph into bytes (`ObjectOutputStream`) and back (`ObjectInputStream`). The class opts in by implementing `Serializable`.
```java
class User implements Serializable {
    private static final long serialVersionUID = 1L;   // version stamp
    private String name;
    private transient String password;                  // skipped → null/0/false after reading
    private static int counter;                         // static fields are never serialized
}
```

**How it behaves:**
- All non-`transient`, non-`static` fields are written, and **every referenced object must be `Serializable`** too, otherwise `NotSerializableException`.
- On read, the constructor of the `Serializable` class is **not called**; only the **no-arg constructor of the first non-serializable superclass** runs. So invariants set in constructors are bypassed.
- **`serialVersionUID`**: if the stream's value differs from the class's, you get `InvalidClassException`. If you don't declare one, the JVM computes it from the class structure (name, fields, methods), so even a harmless change breaks compatibility. Always declare it explicitly.
- Compatible changes: adding fields, adding methods. Incompatible: removing/changing field types, changing class hierarchy.
- Customisation hooks: private `writeObject/readObject`, `readResolve()` (fix singletons/enums), `writeReplace()`, or `Externalizable` for full control.

**Why it is dangerous (a favourite security question):**
1. **Remote code execution:** deserializing untrusted bytes can trigger "gadget chains" in classes on your classpath (Commons-Collections-style exploits). `readObject` runs code *before* you can inspect the object.
2. **Bypasses constructors/validation**, creating objects in states the code never intended.
3. Exposes private fields; the byte format is tied to the class's internals — painful to evolve.
4. Large, slow, and Java-only.
5. DoS by crafted deeply nested/huge graphs.

**Mitigations:** never deserialize untrusted data; use **serialization filters** — **JEP 290 (Java 9, backported to 8u121)** with `ObjectInputFilter`, and **JEP 415 (Java 17)** for JVM-wide, context-specific filters; use an allow-list.
```java
ObjectInputFilter f = ObjectInputFilter.Config.createFilter("com.acme.model.*;java.base/*;!*");
ois.setObjectInputFilter(f);
```
The **serialization proxy pattern** (write a tiny immutable proxy via `writeReplace`) is the safe way to serialize an invariant-heavy class. **Records** are deserialized through their canonical constructor, so validation runs.

**Alternatives:** JSON (Jackson/Gson), Protocol Buffers, Avro, Kryo/FST (careful: still a gadget risk if polymorphic) — schema-based formats that don't execute code and allow versioning.

**Remember:** *`transient` skips a field, `serialVersionUID` versions the class, and never deserialize untrusted bytes — prefer JSON/Protobuf.*

---

### Q29. Varargs — how they work and what goes wrong.

**Answer:**
`void log(String fmt, Object... args)` lets callers pass zero or more arguments. **Under the hood the compiler creates an array** and passes it: `args` is just `Object[]` inside the method. Rules: only one varargs parameter, and it must be **last**.

```java
static int sum(int... n) { int t = 0; for (int x : n) t += x; return t; }
sum();            // n is an EMPTY array (length 0), never null
sum(1, 2, 3);     // new int[]{1,2,3}
sum(arr);         // an existing int[] is passed as is (no copy!)
```

**Overload resolution has three phases:** (1) exact/widening, no boxing, no varargs; (2) with boxing; (3) varargs. So a fixed-arity method always beats a varargs one if applicable.
```java
void f(int a, int b) { }
void f(int... a)     { }
f(1, 2);   // calls the fixed-arity one
```
**Ambiguity:** `f(int a, int... b)` vs `f(int... a)` with `f(1, 2)` → "reference to f is ambiguous".

**Pitfalls:**
1. **`null` passing:** `foo(null)` for `Object...` passes a `null` *array* (with a compiler warning), not an array containing `null`. Use `(Object) null` or `new Object[]{null}`.
2. **Primitive arrays aren't spread:** `Arrays.asList(new int[]{1,2})` returns `List<int[]>` of size 1. With `Integer[]` or `Arrays.asList(1, 2)` it works as expected.
3. **Aliasing:** when a caller passes an existing array, mutation inside the method changes the caller's array. Copy it if you keep it.
4. **Allocation cost:** every call allocates an array — avoid in hot paths. That is why `List.of(...)` and `EnumSet.of(...)` have fixed-arity overloads for small counts.
5. **Generic varargs → heap pollution:** `static <T> void add(List<T>... lists)` makes a generic array; the compiler warns "unchecked generic array creation". If the method only reads the array, annotate `@SafeVarargs` — allowed on `static`, `final` methods and constructors, and on `private` instance methods since **Java 9**.

**Remember:** *Varargs = an array in disguise: never null by default, allocated per call, and lowest priority in overload resolution.*

---

### Q30. `this` vs `super`, constructor chaining, private constructors, and constructor traps.

**Answer:**
- `this` = reference to the **current object**. `this.x` resolves shadowed fields; `this(...)` calls **another constructor of the same class**.
- `super` = view of the **parent class** part. `super.m()` calls the parent's overridden method; `super(...)` calls the parent constructor.
- `this(...)` or `super(...)` must be the **first statement** in a constructor (through Java 21; flexible constructor bodies arrived only as a *preview* in Java 22). You cannot use both in the same constructor.

**Chaining:** if you write neither, the compiler inserts an implicit `super();`. If the parent has no no-arg constructor → compile error. Funnel overloads into one "master" constructor so validation lives in one place:
```java
class Account {
    private final String id; private final BigDecimal balance;
    Account(String id)                    { this(id, BigDecimal.ZERO); }
    Account(String id, BigDecimal b)      { this.id = Objects.requireNonNull(id); this.balance = b; }
    Account(Account other)                { this(other.id, other.balance); }   // copy constructor
}
```
**Facts:** constructors are **not inherited**, have no return type, cannot be `abstract/static/final`. The compiler adds a default no-arg constructor **only if you declare none**.

**Private constructors — why:**
- Utility classes (`Math`, `Collections`) — stop instantiation.
- Singletons — only the class creates the instance.
- Static factory methods (`List.of`, `Optional.of`) — named, cacheable, can return subtypes.
- Builders — force use of the builder.
- Also prevents subclassing (no accessible super constructor).

**Classic trap — overridable method in a constructor:**
```java
class Base  { Base()  { init(); }            void init() { } }
class Child extends Base {
    private String name = "ready";           // assigned AFTER Base() finishes
    @Override void init() { System.out.println(name); }   // prints null!
}
new Child();
```
The parent constructor runs first and calls the child's override while the child's fields are still default values. **Never call overridable methods from a constructor.**

**More probes:**
- Letting `this` escape (registering `this` as a listener or starting a thread in the constructor) exposes a partially built object to other threads.
- An exception in a constructor leaves no object, but a half-built one may already be visible if `this` escaped.
- Enum constructors are always private.

**Remember:** *`this(...)` chains within the class, `super(...)` goes up; the parent constructor runs first — so never call overridable methods in it.*

---

### Q31. Covariant return types, method hiding and field hiding.

**Answer:**
**Covariant return (since Java 5):** an overriding method may return a **subtype** of the parent's return type. It removes casts for callers.
```java
class Animal { Animal self() { return this; } }
class Dog extends Animal { @Override Dog self() { return this; } }   // narrower return type
Dog d = new Dog().self();   // no cast
```
- Works for **reference types only**; primitives must match exactly (`int` ≠ `long`).
- **How the JVM allows it:** the compiler generates a synthetic **bridge method** `Animal self()` in `Dog` that calls `Dog self()`. You see it with reflection (`Method.isBridge()`) and in stack traces. Generics use the same trick.
- Parameters are **not** covariant — changing a parameter type makes an *overload*, not an override (so always use `@Override`).
- The `throws` clause may be narrower or absent, never broader checked exceptions.
- Real use: `clone()` returning the concrete type, fluent builders, `Object`-returning factories.

**Method hiding (static):** `static` methods are not overridden — the subclass's same-signature static method **hides** the parent's. Resolution uses the **declared (compile-time) type**:
```java
class Animal { static String kind() { return "animal"; }  String name = "A"; }
class Dog extends Animal { static String kind() { return "dog"; }     String name = "D"; }

Animal a = new Dog();
a.kind();    // "animal"  — static: chosen by reference type (and misleading when called on an instance)
a.name;      // "A"       — fields are never polymorphic
((Dog) a).name;   // "D"
```
**Field hiding:** a same-named field in a subclass doesn't replace the parent's — both exist in the object. Fields are chosen by the declared type. Avoid it; use getters if you need polymorphism.

**What cannot be overridden:** `private`, `static`, `final` methods and constructors. A `private` method with the same name in a subclass is unrelated.

**Probe:** "Can you override a static method?" → No — you can only hide it. "Can you override with `static`/non-`static` mismatch?" → compile error.

**Remember:** *Methods are polymorphic, fields and statics are not — they follow the declared type; covariant returns use a hidden bridge method.*

---

### Q32. `var` (local-variable type inference) and text blocks — usage and limits.

**Answer:**

**`var` (Java 10):** the compiler infers the type of a **local variable** from its initialiser. It is still **static typing** — not `dynamic` or `Object`.
```java
var orders = new ArrayList<Order>();          // ArrayList<Order>
var map    = Map.of("a", 1);                  // Map<String,Integer>
for (var e : map.entrySet()) { ... }          // Map.Entry<String,Integer>
(var x, var y) -> x + y                       // var in lambda params (Java 11), allows annotations
```
**Where it is NOT allowed:** fields, method parameters, return types, `catch` parameters, variables without an initialiser (`var x;`), `var x = null;`, array initialiser `var a = {1,2};`, or lambda/method reference with no target (`var f = () -> 1;`).

**Pitfalls:**
- `var list = new ArrayList<>();` infers `ArrayList<Object>` (diamond has nothing to infer from).
- Type changes silently: `var x = repo.find();` — if the return type changes, downstream code may still compile with different behaviour. Hurts readability when the right side doesn't reveal the type; use it when the type is obvious (`new`, factory, generics noise).
- `var` is a **reserved type name**, not a keyword: `var var = 1;` compiles, but a class named `var` is forbidden.
- Inferred integer literals are `int`: `var n = 1; n = 2L;` is an error.
- It can infer non-denotable types (anonymous class, intersection types) — a niche power feature.

**Text blocks (preview 13/14, standard in Java 15):** multi-line string literals between `"""`.
```java
String json = """
        {
          "id": %d,
          "name": "%s"
        }
        """.formatted(42, "Asha");
```
Rules:
1. The opening `"""` must be followed by a line break.
2. **Incidental indentation is stripped**: the compiler removes the common leading whitespace, measured against the least-indented line *including the closing delimiter line*. Moving the closing `"""` left keeps more indentation.
3. Trailing spaces on each line are removed; line endings are normalised to `\n`.
4. A trailing `\` suppresses the newline (long single-line strings); `\s` keeps a space; `\"""` escapes the delimiter; normal escapes (`\n`, `\t`) work.
5. Great for SQL, JSON, HTML — no `+` and no `\"` noise. Helpers: `String.formatted()`, `stripIndent()`, `translateEscapes()` (Java 15).
6. No built-in interpolation. **String Templates** were only a *preview* in 21/22 and are not for production.

**Remember:** *`var` = inferred but still static; text blocks = indented multi-line strings with automatic indent stripping.*

---

### Q33. Quick tour of modifiers/keywords: `transient`, `volatile`, `native`, `strictfp`, `synchronized`, `assert`.

**Answer:**

| Keyword | Applies to | Meaning |
|---|---|---|
| `transient` | field | Skipped by **Java** serialization (JSON libraries have their own rules) |
| `volatile` | field | Every read sees the latest write by any thread (visibility + ordering). **No atomicity** — `count++` is still unsafe |
| `synchronized` | method/block | Mutual exclusion + visibility via the object's monitor (reentrant) |
| `native` | method | No Java body; implemented in C/C++ through **JNI** (e.g. `System.arraycopy`, `Object.hashCode`) |
| `strictfp` | class/method | Forces strict IEEE 754 floating point on every platform |
| `assert` | statement | Checks an invariant; **off by default** (`-ea` to enable) |
| `abstract` | class/method | No implementation / cannot be instantiated |
| `final` | var/method/class | Not reassignable / not overridable / not subclassable |

**Details worth saying aloud:**
- **`strictfp`**: obsolete since **Java 17 (JEP 306)** — all floating-point is strict by default, and the keyword only produces a compiler warning. Before 17 (on x87 hardware) intermediate results could use extended precision and differ across machines.
- **`native`**: crosses the Java/OS boundary; costs a JNI transition, can crash the whole JVM, hurts portability. Modern replacement: the **Foreign Function & Memory API** (finalised in Java 22, preview in 19–21).
- **`volatile`** is perfect for a one-writer **flag** (`volatile boolean running`) and safe publication; it is not a replacement for `synchronized`/`Atomic*` for compound actions (check-then-act, increment). `volatile long/double` guarantees atomic 64-bit reads/writes.
- **`transient`** + `final`: a `transient final` field comes back as default (`null`/0) after deserialization and can't be fixed in `readObject` — avoid.
- **`assert`**: `assert x > 0 : "x must be positive";`. Don't use it to validate public-method arguments (use exceptions) and never put side effects in it — it may not run.
- **Reserved but unused:** `goto`, `const`. Since Java 9 a single `_` is reserved (unnamed variables arrive as a preview in Java 21).
- **Contextual keywords** (only special in context, still valid identifiers elsewhere): `var`, `record`, `sealed`, `permits`, `non-sealed`, `yield`, `module`.

**Remember:** *`transient` = not serialised, `volatile` = visible but not atomic, `native` = written in C, `strictfp` = dead since 17.*

---

### Q34. Composition vs inheritance — when is inheritance a mistake?

**Say this first:** "Inheritance breaks encapsulation: a subclass depends on its parent's *implementation*, not just its contract."

**Answer:**
- **Inheritance** ("is-a"): the subclass inherits the parent's code *and* its internal behaviour.
- **Composition** ("has-a"): a class holds a reference to another object and **delegates** to it, exposing only what it chooses.

**The fragile base class problem (Effective Java's `InstrumentedHashSet`):**
```java
class CountingSet<E> extends HashSet<E> {
    private int added;
    @Override public boolean add(E e)               { added++; return super.add(e); }
    @Override public boolean addAll(Collection<? extends E> c) { added += c.size(); return super.addAll(c); }
    int added() { return added; }
}
var s = new CountingSet<String>();
s.addAll(List.of("a", "b", "c"));
s.added();   // 6, not 3!  HashSet.addAll() internally calls add() — an undocumented detail.
```
Fix with composition (the **forwarding/decorator** approach):
```java
class CountingSet<E> implements Set<E> {
    private final Set<E> delegate;                 // has-a
    private int added;
    CountingSet(Set<E> delegate) { this.delegate = delegate; }
    public boolean add(E e)  { added++; return delegate.add(e); }
    public boolean addAll(Collection<? extends E> c) { added += c.size(); return delegate.addAll(c); }
    // other Set methods simply delegate
}
```
Now it works with *any* `Set` (HashSet, TreeSet, a synchronized set) and changes in the delegate's internals can't break it.

**Why prefer composition:**
- Behaviour can be **swapped at runtime** (Strategy, Decorator); inheritance is fixed at compile time.
- Easier testing — inject a fake delegate.
- Java has single class inheritance; composition has no such limit.
- Parent changes (new methods, changed self-calls) can silently break subclasses; composition insulates you.
- Avoids exposing methods that don't make sense (the classic mistake: `Stack extends Vector`, `Properties extends Hashtable` — now `Stack` allows `insertElementAt`).

**When inheritance is right:**
- A genuine, stable "is-a" (Liskov Substitution holds: a subclass can stand in everywhere the parent is used).
- The parent was **designed and documented for extension** (document self-use, provide hooks), e.g. `AbstractList`, template-method skeletons.
- Otherwise make classes `final` (or sealed, Java 17) — "design for inheritance or prohibit it".

**Java 8+ note:** interface default methods give you reusable behaviour (mixin-like) without class-inheritance coupling.

**Remember:** *Inherit only for a true, documented "is-a"; otherwise hold the object and delegate.*

---

### Q35. String gotchas every senior should know: `split`, `substring`, `strip`, `format`, `chars`, `intern`, `hashCode`.

**Answer:**

**`split(regex)` — takes a regex, drops trailing empties:**
```java
"a,b,,".split(",")        // ["a","b"]            trailing empty strings removed
"a,b,,".split(",", -1)    // ["a","b","",""]      negative limit keeps them
"a.b".split(".")          // []                   "." is a regex "any char"!
"a.b".split("\\.")        // ["a","b"]            or Pattern.quote(".")
"".split(",")             // [""]  (length 1, not 0)
```
Same for `|`, `$`, `*`, `+`. For hot paths use a precompiled `Pattern` or a manual indexOf loop (`split` has a fast path only for single non-regex chars).

**`substring`:** up to Java 7u5 it shared the parent's `char[]`, so a tiny substring could keep a huge string alive (memory leak). Since **7u6** it **copies**. Since **Java 9 compact strings (JEP 254)**, a `String` is stored as `byte[]` (Latin-1 = 1 byte/char, else UTF-16).

**`trim()` vs `strip()` (Java 11):** `trim` removes chars ≤ `U+0020` only; `strip` removes Unicode whitespace (`Character.isWhitespace`), e.g. an em-space. Also `isBlank()`, `stripLeading/Trailing`, `repeat(n)`, `lines()` (all Java 11). `String.join(",", list)` is Java 8.

**Case and format are locale-sensitive:**
```java
"TITLE".toLowerCase();                  // in a Turkish locale → "tıtle" (dotless ı)! 
"TITLE".toLowerCase(Locale.ROOT);       // always use an explicit locale for machine-readable text
String.format(Locale.US, "%.2f", 1234.5);   // "1234.50"; default locale could give "1234,50"
String.format("%s%n", x);               // %n = platform line separator
```
**`chars()` and Unicode:** `length()` counts UTF-16 **code units**, not characters. `"😀".length()` is 2. `s.chars()` is an `IntStream` of code units; use `s.codePoints()` for real characters. `new StringBuilder(s).reverse()` handles surrogate pairs correctly; a manual `char` loop doesn't.

**`intern()`:** returns the canonical pooled instance. Since Java 7 the pool is on the heap and sized by `-XX:StringTableSize`. Use sparingly; for many duplicate strings prefer G1's `-XX:+UseStringDeduplication` or your own `Map`.

**`hashCode()`:** `s[0]*31^(n-1) + s[1]*31^(n-2) + … + s[n-1]` — cached in the object (a hash of `0` is recomputed in old JDKs, fixed with a flag in Java 13). Collisions are easy: `"Aa".hashCode() == "BB".hashCode()` (2112). Crafted collisions are why `HashMap` treeifies long buckets (Java 8). Don't rely on it for ordering or persistence — the algorithm is specified, but it's a poor hash for security.

**Other traps:**
- `String.valueOf((Object) null)` → `"null"`; `String.valueOf((char[]) null)` → NPE. `"x" + null` → `"xnull"`.
- Keep passwords in `char[]` (can be wiped) rather than `String` (stays in the pool/heap).
- `s1.compareTo(s2)` compares UTF-16 values, not human/locale order — use `Collator` for display sorting.
- `equals` is case-sensitive; use `equalsIgnoreCase`. Put the literal first (`"ok".equals(s)`) or use `Objects.equals` for null safety.
- `==` on strings only "works" for literals/interned values.

**Remember:** *`split` is regex, `strip` is Unicode-aware, case/format need a Locale, and `length()` counts UTF-16 units.*

---

### Q36. How does `switch` work on Strings — and how has `switch` evolved (Java 7 → 21)?

**Answer:**
**Strings in `switch` (Java 7):** syntactic sugar. The compiler generates a two-step switch:
1. `switch (s.hashCode())` — jump to the matching hash bucket (a `lookupswitch`).
2. `s.equals("literal")` — confirm (to handle hash collisions) and set an index.
3. A second `switch` on that index runs your code.

So it is O(1)-ish and case-sensitive. Consequences:
- `switch (null)` → **NullPointerException** (it calls `hashCode()`); check for null first (or use `case null` in Java 21).
- Case labels must be **compile-time constants** (literals or `static final` constants).
- Allowed selector types (classic): `byte, short, char, int`, their wrappers, `String`, `enum`. **Not** `long`, `float`, `double`, `boolean`.
- Enum switches compile to a synthetic `$SwitchMap$` array of ordinals — recompiling the enum without recompiling callers can break it.

**Classic statement traps:**
```java
switch (code) {
    case "A":  System.out.println("a");     // fall-through! no break → also prints "b"
    case "B":  System.out.println("b");  break;
    default:   System.out.println("other");
}
```
Fall-through is legal and the #1 switch bug; `default` can be anywhere but still runs only if nothing matched (or via fall-through into it).

**Switch expressions (standard in Java 14):** returns a value, arrow form has **no fall-through**, multiple labels, `yield` for blocks, and the compiler enforces **exhaustiveness**.
```java
int days = switch (month) {
    case "FEB"                      -> 28;
    case "APR", "JUN", "SEP", "NOV" -> 30;
    default -> {
        if (month.isEmpty()) throw new IllegalArgumentException();
        yield 31;                    // yield returns from a block
    }
};
```
**Pattern matching for `switch` (standard in Java 21):** works on any reference type, with type patterns, guards (`when`), `case null`, record deconstruction, and exhaustiveness for `sealed` hierarchies.
```java
String describe(Object o) {
    return switch (o) {
        case null                        -> "null";
        case Integer i when i > 100      -> "big int " + i;
        case Integer i                   -> "int " + i;
        case String s                    -> "string of length " + s.length();
        default                          -> "something else";
    };
}
```
Order matters: a more general pattern before a specific one is a **dominance** compile error.

**Pitfalls / probes:**
- Mixing arrow (`->`) and colon (`:`) styles in one switch is not allowed.
- For case-insensitive matching switch on `s.toLowerCase(Locale.ROOT)`.
- For a very large or dynamic set of branches, a `Map<String, Handler>` or enum with behaviour is cleaner than a giant switch.

**Remember:** *String switch = hashCode then equals (NPE on null); prefer arrow-form switch expressions — no fall-through and exhaustive.*

---

# 2. Collections Framework (Deep Dive)

### Q37. Draw the Collection hierarchy and pick the right implementation.

```
Iterable
 └── Collection
      ├── List      → ArrayList, LinkedList, CopyOnWriteArrayList, Vector
      ├── Set       → HashSet, LinkedHashSet, TreeSet, CopyOnWriteArraySet
      └── Queue     → ArrayDeque, PriorityQueue, LinkedList,
                      BlockingQueue (ArrayBlocking, LinkedBlocking, SynchronousQueue,
                      DelayQueue, PriorityBlockingQueue)

Map (separate root) → HashMap, LinkedHashMap, TreeMap, Hashtable,
                      ConcurrentHashMap, ConcurrentSkipListMap, EnumMap, WeakHashMap
```

**Selection cheat sheet:**

| Need | Use |
|---|---|
| Indexed access, mostly reads | `ArrayList` |
| Heavy insert/remove at ends, queue/deque | `ArrayDeque` (faster than `LinkedList`) |
| Unique, no order | `HashSet` |
| Unique, insertion order | `LinkedHashSet` |
| Unique, sorted | `TreeSet` |
| Key-value, fastest | `HashMap` |
| LRU cache | `LinkedHashMap` with `accessOrder=true` + `removeEldestEntry` |
| Sorted map / range queries | `TreeMap` (`headMap`, `tailMap`, `subMap`) |
| High-concurrency map | `ConcurrentHashMap` |
| Read-mostly listener list | `CopyOnWriteArrayList` |
| Producer-consumer | `LinkedBlockingQueue` / `ArrayBlockingQueue` |
| Enum keys | `EnumMap` (array-backed, fastest) |
| Cache with GC-able keys | `WeakHashMap` |

**Remember:** *ArrayList by default, ArrayDeque over LinkedList, ConcurrentHashMap over Hashtable.*

---

### Q38. `ArrayList` internals — growth, and why `remove` is expensive.

- Backed by `Object[]`; default capacity **10** (lazily allocated on first add).
- Growth: `newCapacity = oldCapacity + (oldCapacity >> 1)` → **1.5×** (10 → 15 → 22 → 33…), then `Arrays.copyOf` (System.arraycopy, O(n)).
- `get(i)` = **O(1)**. `add(last)` = amortized O(1). `add(i, e)` / `remove(i)` = **O(n)** due to shifting.
- `remove(int index)` vs `remove(Object o)` — with `List<Integer>`, `list.remove(1)` removes **index 1**, `list.remove(Integer.valueOf(1))` removes the **value 1**. Classic trick question.
- Pre-size it when you know the size: `new ArrayList<>(10_000)` avoids ~10 array copies.

**Remember:** *ArrayList grows 1.5×; index is O(1), middle surgery is O(n).*

---

### Q39. `HashMap` internals — the most asked question. Explain end to end.

**Structure:** array of buckets (`Node<K,V>[] table`), each bucket a **linked list**, converted to a **red-black tree** when it gets long.

**Constants:** default capacity **16**, load factor **0.75**, `TREEIFY_THRESHOLD = 8`, `UNTREEIFY_THRESHOLD = 6`, `MIN_TREEIFY_CAPACITY = 64`.

**put(k, v) flow:**
1. `hash(key)` = `h = key.hashCode(); h ^ (h >>> 16)` — **spreading**: XORs high bits into low bits so that poor hash functions still vary the low-order bits used for indexing.
2. `index = (n - 1) & hash` — works because n is a power of two, so this is a fast modulo.
3. Bucket empty → insert node.
4. Bucket occupied → compare `hash` first, then `==`, then `equals`. Match → replace value. No match → append to list (Java 8 appends at **tail**; Java 7 used head).
5. If list length ≥ 8 **and** table length ≥ 64 → **treeify** that bucket (O(log n) worst case instead of O(n)).
6. If `size > capacity * loadFactor` → **resize** to 2× and rehash. Java 8 splits each bucket into "lo" and "hi" lists using `(hash & oldCap)` — no rehash computation needed.

**Complexity:** average O(1), worst case O(log n) after treeification (pre-Java 8 it was O(n) — that was the DoS-able hash-collision attack).

**Why capacity is a power of 2:** so `(n-1) & hash` replaces `%`, and so resize can split buckets cheaply.

**null:** `HashMap` allows **one null key** (bucket 0) and many null values. `ConcurrentHashMap`/`Hashtable` allow neither.

**Remember:** **16 / 0.75 / 8 / 64 / 2×** — capacity, load factor, treeify, min table, growth.

---

### Q40. Why do keys need to be immutable? What happens if they mutate?

```java
List<String> key = new ArrayList<>(List.of("a"));
Map<List<String>, String> map = new HashMap<>();
map.put(key, "value");
key.add("b");                 // hashCode changed!
map.get(key);                 // null — looking in the wrong bucket
map.size();                   // 1 — the entry is a memory leak
```
**Why:** `HashMap` chooses the bucket from `hashCode()` **at insertion time** and never re-checks it. If the key's fields change afterwards, `hashCode()` now points to a different bucket, so `get`/`remove` look in the wrong place and the original entry is unreachable but still referenced by the map (a leak). This is why `String`, `Integer`, enums and records of immutables are the safe key types — and why a JPA entity with a mutable/generated id is a risky key.

**Remember:** *A mutated key is a lost entry — always use immutable keys (String, Integer, records, enums).*

---

### Q41. `HashMap` vs `Hashtable` vs `ConcurrentHashMap` vs `SynchronizedMap`.

| | HashMap | Hashtable | Collections.synchronizedMap | ConcurrentHashMap |
|---|---|---|---|---|
| Thread-safe | No | Yes | Yes | Yes |
| Locking | — | Whole map (every method) | Whole map (single mutex) | **Per-bucket CAS + synchronized node** |
| null key/value | 1 null key, null values | Neither | Same as backing map | Neither |
| Iterator | fail-fast | fail-fast (Enumeration not) | fail-fast (manual sync needed) | **fail-safe / weakly consistent** |
| Read performance under load | — | Poor | Poor | **Excellent (reads mostly lock-free)** |
| Status | Default choice | Legacy | Legacy | Modern concurrent choice |

**ConcurrentHashMap in Java 8+:** dropped segments/`ReentrantLock` striping. Now: `volatile` table, CAS for empty-bucket insert, `synchronized` on the **first node of the bucket** for collisions, treeify like HashMap, and `size()` via a striped `CounterCell` array (`LongAdder` style). `size()` is therefore an **estimate**.

**Atomic compound ops (why CHM ≠ synchronized map):**
```java
map.computeIfAbsent(key, k -> expensiveLoad(k));   // atomic
counters.merge(key, 1L, Long::sum);                // atomic increment
map.putIfAbsent(key, value);
```
With `synchronizedMap` you'd need an external lock around check-then-act.

**Remember:** *Hashtable locks the house; ConcurrentHashMap locks one room.*

---

### Q42. Fail-fast vs fail-safe iterators.

- **Fail-fast** (`ArrayList`, `HashMap`, `HashSet`): iterator tracks `modCount`; structural modification during iteration → `ConcurrentModificationException`. It's a **best-effort bug detector**, not a guarantee.
- **Fail-safe** (`ConcurrentHashMap`, `CopyOnWriteArrayList`): iterate a snapshot / weakly-consistent view. No CME, but you may not see concurrent updates.

**Safe removal while iterating:**
```java
Iterator<String> it = list.iterator();
while (it.hasNext()) if (bad(it.next())) it.remove();   // OK
list.removeIf(this::bad);                                // best, Java 8+
for (String s : list) if (bad(s)) list.remove(s);        // ❌ CME
```

**Remember:** *modCount mismatch = CME; use `removeIf` or the iterator's own `remove`.*

---

### Q43. `Comparable` vs `Comparator`.

- `Comparable<T>` — natural ordering, `compareTo`, one per class, in the domain class.
- `Comparator<T>` — external ordering, many per class, composable.

```java
list.sort(Comparator.comparing(Employee::getDept)
                    .thenComparing(Employee::getSalary, Comparator.reverseOrder())
                    .thenComparing(Employee::getName,
                                   Comparator.nullsLast(String::compareTo)));
```
`Collections.sort` / `List.sort` use **TimSort** (stable, adaptive merge sort, O(n log n)); primitives use dual-pivot quicksort (not stable). An inconsistent comparator throws `IllegalArgumentException: Comparison method violates its general contract!` — usually caused by returning a value based on subtraction that overflows, or non-transitive logic.

**Remember:** *Comparable = "I know my order"; Comparator = "you tell me the order".*

---

### Q44. `Arrays.asList()` vs `List.of()` vs `new ArrayList<>()`.

| | `Arrays.asList` | `List.of` | `new ArrayList<>` |
|---|---|---|---|
| Fixed size | Yes | Yes | No |
| `set(i,v)` allowed | **Yes** (writes through to array) | No | Yes |
| `add`/`remove` | `UnsupportedOperationException` | `UnsupportedOperationException` | Yes |
| null elements | Allowed | **NPE** | Allowed |
| Truly immutable | No | Yes | No |

`Collections.unmodifiableList(x)` is an **unmodifiable view** — mutate the backing list and the view changes. `List.copyOf(x)` is an immutable **copy**.

**Remember:** *asList = fixed-size view; List.of = immutable; unmodifiableList = view, copyOf = snapshot.*

---

### Q45. `HashSet` vs `LinkedHashSet` vs `TreeSet`.

| | HashSet | LinkedHashSet | TreeSet |
|---|---|---|---|
| Backing | HashMap | LinkedHashMap | TreeMap (red-black) |
| Order | None | Insertion | Sorted |
| add/contains | O(1) | O(1) | O(log n) |
| null | 1 allowed | 1 allowed | **NPE** (needs comparison) |

`HashSet` literally stores your element as a HashMap key with a dummy `PRESENT` value.

**What this means in practice:** uniqueness in a `HashSet`/`LinkedHashSet` is decided by `hashCode()` + `equals()` (so your class must implement both correctly — see Q2); uniqueness in a `TreeSet` is decided by `compareTo`/`Comparator` returning 0 (so two *different* objects that compare as 0 are treated as duplicates and one is dropped). Pick `LinkedHashSet` when you need de-duplication **and** predictable output order (e.g. unique tags in the order the user typed them); pick `TreeSet` when you need sorted iteration or `first()/floor()/ceiling()` range lookups.

**Remember:** *Hash = fast, Linked = ordered, Tree = sorted.*

---

### Q46. How does `ConcurrentHashMap` work internally? Java 7 segments vs Java 8+, and why no nulls?

**Say this first:** "Java 7 locked a *segment* of the map; Java 8+ locks only a single *bucket* — CAS for an empty bucket, `synchronized` on the bucket's first node for a collision — and reads take no lock at all."

**Answer:**
- **Why it exists:** `Hashtable`/`synchronizedMap` use one lock for the whole map, so threads queue up. `ConcurrentHashMap` (CHM) lets many threads read and write at the same time.
- **Java 7:** an array of `Segment`s (default 16, from `concurrencyLevel`). Each `Segment` was a small hash table that extended `ReentrantLock`. A write locked one segment, so at most 16 writers ran in parallel.
- **Java 8+ `put` step by step:**
  1. Compute the spread hash; find the bucket index.
  2. Table not yet created → create it lazily (CAS on `sizeCtl`).
  3. Bucket empty → insert the node with a **CAS** (no lock).
  4. Bucket is being resized (`hash == MOVED`) → the thread *helps* the resize.
  5. Otherwise → `synchronized (firstNodeOfBucket)`, then walk the list or tree bin and insert or replace.
  6. List ≥ 8 nodes **and** table ≥ 64 → treeify (same as `HashMap`).
- **Reads (`get`)** take no lock: `table`, `Node.val` and `Node.next` are `volatile`.
- **Resize is cooperative:** the table is split into ranges; several threads each migrate a range.
- **`size()`:** kept in `baseCount` plus a `CounterCell[]` (the same idea as `LongAdder`) to avoid contention. Under concurrent updates it is only a snapshot. Prefer `mappingCount()` — it returns a `long` and is the recommended method.
- **Why no null key/value:** with `map.get(k) == null` you cannot tell "absent" from "mapped to null". In a plain `HashMap` you could follow up with `containsKey`. In a concurrent map another thread can change the map between the two calls, so the answer would be unreliable. Doug Lea banned nulls to keep the API unambiguous.
- **Atomic per-key operations:** `compute`, `computeIfAbsent`, `computeIfPresent`, `merge`, `putIfAbsent` run **atomically for that key** because they execute under the bucket lock.

```java
ConcurrentHashMap<String, LongAdder> hits = new ConcurrentHashMap<>();
hits.computeIfAbsent(url, k -> new LongAdder()).increment();   // best counter idiom
Set<String> set = ConcurrentHashMap.newKeySet();               // concurrent Set
```

| | Java 7 | Java 8+ |
|---|---|---|
| Lock granularity | Segment (≈16) | Single bucket head |
| Empty-bucket insert | Needs lock | CAS, lock-free |
| Collision structure | Linked list | List → red-black tree |
| `concurrencyLevel` | Real parameter | Only a sizing hint |

**Pitfalls / follow-up probes:**
- Only the **single operation** is atomic. `if (!map.containsKey(k)) map.put(k, v)` is still a race — use `putIfAbsent`/`computeIfAbsent`.
- Keep the lambda in `compute*`/`merge` **short** and never call blocking I/O: it holds the bucket lock and blocks other writers to that bucket.
- Never modify the same map inside the lambda. It can deadlock/livelock (Java 8) or throw `IllegalStateException: Recursive update` (Java 9+, when detected).
- Iterators are **weakly consistent**: never throw `ConcurrentModificationException`, may or may not show updates made after creation.
- Bulk operations (`forEach(parallelismThreshold, …)`, `reduce`, `search`) exist — they run in parallel on the common pool.

**Remember:** *Java 8 CHM = CAS for empty bucket + `synchronized` on the bucket head + lock-free reads; no nulls because `get()==null` would be ambiguous.*

---

### Q47. `ArrayList` vs `LinkedList` — which would you really use?

**Say this first:** "`ArrayList` — almost always. `LinkedList`'s theoretical O(1) insert rarely wins in practice."

**Answer:**
- **`ArrayList`** is a resizable `Object[]`. Index access is O(1). Appending is amortised O(1) (grows by ~1.5×). Insert/remove in the middle shifts elements with `System.arraycopy`, which is O(n) but very fast.
- **`LinkedList`** is a doubly linked list of `Node` objects (`item`, `prev`, `next`). `get(i)` walks from the nearer end — O(n). Insert/remove is O(1) **only if you already hold the node/iterator position**; finding the position is O(n).
- **Why `ArrayList` wins in practice:**
  1. **Cache locality** — the array is contiguous in memory, so the CPU prefetcher loads neighbours. Linked nodes are scattered on the heap, causing cache misses.
  2. **Memory** — each `LinkedList` node costs an extra object (~24 bytes on a 64-bit JVM with compressed oops) on top of the element. `ArrayList` stores just one reference.
  3. **GC pressure** — one node object per element for the garbage collector to track.
- `LinkedList` also implements `Deque`, but `ArrayDeque` is faster for queue/stack work.

| Operation | ArrayList | LinkedList |
|---|---|---|
| `get(i)` | O(1) | O(n) |
| `add(e)` at end | amortised O(1) | O(1) |
| `add(0, e)` | O(n) (fast shift) | O(1) |
| `add/remove` in middle by index | O(n) | O(n) to find + O(1) to link |
| `remove` via `Iterator` at current position | O(n) | O(1) |
| `contains` | O(n) | O(n) |
| Memory per element | ~4–8 bytes | ~24+ bytes extra |

```java
List<Integer> list = new ArrayList<>(10_000);   // pre-size when you know the size
```

**Pitfalls / follow-up probes:**
- `for (int i...) linkedList.get(i)` is **O(n²)** — a classic trap.
- Queue/stack use → `ArrayDeque`. Frequent removal from the middle of a huge list → consider a different structure, not `LinkedList`.
- `ArrayList.remove(int)` vs `remove(Object)` — `list.remove(1)` removes the index, `list.remove(Integer.valueOf(1))` removes the value.
- `new ArrayList<>(n)` avoids repeated growth/copy; `trimToSize()` releases spare capacity.
- Joshua Bloch (who wrote `LinkedList`) has said he doesn't use it.

**Remember:** *Contiguous memory beats pointer chasing — default to `ArrayList`, `ArrayDeque` for queues.*

---

### Q48. Why are `Vector` and `Stack` discouraged? Explain `Queue`, `Deque` and `ArrayDeque`.

**Say this first:** "`Vector`/`Stack` are legacy (Java 1.0), synchronised on every call, and `Stack` leaks list operations. Use `ArrayDeque`."

**Answer:**
- **`Vector`:** like `ArrayList` but every method is `synchronized`. That is slow when you don't need it, and still not enough for compound actions (check-then-act). It doubles in size by default.
- **`Stack extends Vector`:** a design mistake — inheritance, not composition. A "stack" that also allows `get(i)`, `add(i, e)` and `remove(i)` breaks LIFO (last-in-first-out) guarantees.
- **`Queue`** = FIFO (first-in-first-out). **`Deque`** = double-ended queue — add/remove at both ends, so it works as a queue *and* a stack.

| Intent | Throws on failure | Returns special value |
|---|---|---|
| Add | `add(e)` | `offer(e)` |
| Remove head | `remove()` | `poll()` (null if empty) |
| Look at head | `element()` | `peek()` (null if empty) |

Stack methods on `Deque`: `push` (= `addFirst`), `pop` (= `removeFirst`), `peek` (= `peekFirst`).

- **`ArrayDeque`:** a circular (ring) array with `head` and `tail` indexes; grows when full. Add/remove at both ends is amortised O(1). No per-element node → better memory and cache behaviour than `LinkedList`. **Nulls are not allowed** (null is the "empty" signal of `poll`/`peek`). Not thread-safe.

```java
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1); stack.push(2);
stack.pop();                      // 2  (LIFO)

Queue<String> q = new ArrayDeque<>();
q.offer("a"); q.poll();           // FIFO
```

**Pitfalls / follow-up probes:**
- Iteration order differs: `Stack` iterates **bottom → top**, `Deque` used as a stack iterates **top → bottom** (head first).
- Need a thread-safe stack/queue? Use `ConcurrentLinkedDeque` or a `BlockingQueue` — not `Vector`/`Stack`.
- Do not use `LinkedList` just to get queue behaviour: it allows nulls and costs more memory.
- Need a thread-safe `List` with cheap reads? `CopyOnWriteArrayList`, or `Collections.synchronizedList` — not `Vector`.

**Remember:** *Stack/Vector = legacy; `ArrayDeque` = default stack and queue.*

---

### Q49. How does `PriorityQueue` work? Is its iteration order sorted?

**Say this first:** "It's a binary min-heap stored in an array. Only the head is guaranteed to be the smallest — iteration order is **not** sorted."

**Answer:**
- **What:** an unbounded queue where `poll()` returns the **smallest** element (by natural order or a `Comparator`), not the oldest.
- **How (binary heap):** a complete binary tree stored in an `Object[]`. For index `i`: parent = `(i-1)/2`, children = `2i+1` and `2i+2`. Rule: every parent ≤ its children.
  - `offer(e)`: put at the end, then **sift up** (swap with parent while smaller) — O(log n).
  - `poll()`: take the root, move the last element to the root, then **sift down** — O(log n).
  - `peek()`: read the root — O(1).
  - `remove(Object)` and `contains` — O(n) (linear search).
  - `new PriorityQueue<>(collection)` builds the heap in O(n) (heapify).
- **Why iteration isn't sorted:** the heap only orders parent vs child, not siblings. `toString()`/`for-each` walk the array as is.

```java
PriorityQueue<Integer> pq = new PriorityQueue<>(List.of(5, 1, 4, 2, 3));
System.out.println(pq);          // e.g. [1, 2, 4, 5, 3]  — NOT sorted
while (!pq.isEmpty()) System.out.print(pq.poll() + " ");   // 1 2 3 4 5

PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());

// Top-K largest: keep a min-heap of size K  → O(n log k)
PriorityQueue<Integer> topK = new PriorityQueue<>();
for (int x : data) { topK.offer(x); if (topK.size() > k) topK.poll(); }
```

**Pitfalls / follow-up probes:**
- Elements must be `Comparable` or you must pass a `Comparator`, otherwise `ClassCastException`. `null` is not allowed.
- **Not stable** — equal-priority elements come out in no guaranteed order. Add a sequence number as a tie-breaker if you need FIFO among equals.
- **Mutating an element's priority after insertion** silently breaks the heap. Remove and re-add it.
- Not thread-safe → `PriorityBlockingQueue`.
- Use for: scheduling, Dijkstra, merge K sorted lists, top-K, running median (two heaps).

**Remember:** *Heap = only the root is ordered; offer/poll are O(log n), peek O(1), iteration is not sorted.*

---

### Q50. `TreeMap` internals and the `NavigableMap` methods (`floor`, `ceiling`, `headMap`).

**Say this first:** "`TreeMap` is a red-black tree — sorted keys, O(log n) operations, and powerful range/nearest-key queries."

**Answer:**
- **Structure:** a self-balancing **red-black tree** (a binary search tree that recolours/rotates after changes to keep height ≤ ~2·log n). So `get/put/remove/containsKey` are guaranteed O(log n).
- **Ordering:** keys use natural order (`Comparable`) or the `Comparator` given to the constructor.
- **Equality is decided by the comparator, not `equals`.** Two keys with `compare() == 0` are the *same key*.
- **Nearest-key methods:**

| Method | Meaning |
|---|---|
| `firstKey()/lastKey()` | smallest / largest key |
| `floorKey(k)` | greatest key **≤ k** |
| `ceilingKey(k)` | smallest key **≥ k** |
| `lowerKey(k)` | greatest key **< k** |
| `higherKey(k)` | smallest key **> k** |
| `headMap(to, incl)` | view of keys **< (or ≤) to** |
| `tailMap(from, incl)` | view of keys **≥ (or >) from** |
| `subMap(from, to)` | range view |
| `pollFirstEntry()` | remove and return smallest |
| `descendingMap()` | reversed view |

```java
TreeMap<Integer, String> tiers = new TreeMap<>();
tiers.put(0, "Bronze"); tiers.put(1000, "Silver"); tiers.put(5000, "Gold");
tiers.floorEntry(3200).getValue();      // "Silver"  — which slab does 3200 fall into?
tiers.headMap(1000, true);              // {0=Bronze, 1000=Silver}  (a live view)
```

**Pitfalls / follow-up probes:**
- A comparator **inconsistent with `equals`** makes `TreeMap`/`TreeSet` behave oddly (e.g. a `BigDecimal` `2.0` and `2.00` are `equals`-different but compare equal → one entry is lost).
- Null keys throw `NullPointerException` with natural ordering.
- `headMap/tailMap/subMap` are **views** — writes go through to the original; inserting outside the range throws `IllegalArgumentException`.
- Not thread-safe → `ConcurrentSkipListMap`.
- Prefer `TreeMap` only when you need sorted order or range queries; otherwise `HashMap` (O(1)) is faster.
- Typical uses: price slabs, time-series lookups ("last event before t"), sorted reports, interval scheduling.

**Remember:** *`TreeMap` = red-black tree, O(log n), sorted; `floor` ≤, `ceiling` ≥, `lower` <, `higher` >.*

---

### Q51. How do you build an LRU cache with `LinkedHashMap`?

**Say this first:** "Construct it with `accessOrder=true` and override `removeEldestEntry` to evict the least-recently-used entry."

**Answer:**
- **`LinkedHashMap`** = `HashMap` + a doubly linked list through all entries. That list gives a predictable iteration order.
- Two orders: **insertion order** (default) and **access order** (`new LinkedHashMap<>(cap, 0.75f, true)`). In access order, every `get`/`put` of an existing key moves that entry to the **end** (most recent). The **head** is therefore the least recently used.
- After each insertion, `LinkedHashMap` calls `removeEldestEntry(eldest)`. If it returns `true`, the head entry is removed. That is the whole eviction policy.
- Lookup and update stay O(1) (hash lookup + relink the list node).

```java
class LruCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxSize;
    LruCache(int maxSize) {
        super(16, 0.75f, true);          // true = access order
        this.maxSize = maxSize;
    }
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxSize;
    }
}

var cache = new LruCache<String, String>(2);
cache.put("a", "1"); cache.put("b", "2");
cache.get("a");                 // a becomes most recent
cache.put("c", "3");            // evicts "b"
```

**Pitfalls / follow-up probes:**
- **Not thread-safe.** Wrap with `Collections.synchronizedMap(...)`, or use a lock. In access-order mode even `get()` is a *structural modification* (it relinks), so concurrent reads can corrupt it and iteration while reading throws `ConcurrentModificationException`.
- For production use **Caffeine** (size/time eviction, high concurrency) instead of hand-rolling.
- Interview variant: implement LRU **without** `LinkedHashMap` → `HashMap<K, Node>` + your own doubly linked list with `head`/`tail` sentinels; move-to-front on access, evict from the tail.
- Java 21: `LinkedHashMap` is a `SequencedMap` (`putFirst`, `putLast`, `firstEntry`, `reversed()`).
- Re-inserting an existing key in *insertion-order* mode does **not** change its position.

**Remember:** *LRU = `LinkedHashMap(…, true)` + `removeEldestEntry`; O(1) everything.*

---

### Q52. `EnumMap`, `EnumSet` and `BitSet` — when are they better than `HashMap`/`HashSet`?

**Say this first:** "They trade generality for speed and memory: indexes/bits instead of hashing."

**Answer:**
- **`EnumMap<K extends Enum<K>, V>`:** internally an array indexed by the enum's `ordinal()`. No hashing, no collisions, no node objects. Iteration follows **enum declaration order**. Nulls keys are not allowed; null values are.
- **`EnumSet<E>`:** abstract class with static factories (`noneOf`, `allOf`, `of`, `range`, `complementOf`). Two hidden implementations: `RegularEnumSet` (one `long` bitmask, up to 64 constants) and `JumboEnumSet` (a `long[]`). Operations like `contains`, `add` and `retainAll` become bit operations.
- **`BitSet`:** a growable `long[]` where bit *i* means "int *i* is present". Great for dense non-negative ints: a flag for each of 1 million ids costs ~125 KB (versus tens of MB for a `HashSet<Integer>`).

```java
enum Day { MON, TUE, WED, THU, FRI, SAT, SUN }

EnumSet<Day> weekend = EnumSet.of(Day.SAT, Day.SUN);
EnumSet<Day> weekdays = EnumSet.complementOf(weekend);
EnumSet<Day> range = EnumSet.range(Day.TUE, Day.THU);

Map<Day, List<String>> shifts = new EnumMap<>(Day.class);   // needs the Class token

BitSet seen = new BitSet(1_000_000);
seen.set(42);
boolean has = seen.get(42);
int count = seen.cardinality();                 // number of set bits
for (int i = seen.nextSetBit(0); i >= 0; i = seen.nextSetBit(i + 1)) { /* ... */ }
// set algebra: a.and(b), a.or(b), a.xor(b), a.andNot(b)
```

**Pitfalls / follow-up probes:**
- Neither `EnumMap` nor `EnumSet` nor `BitSet` is thread-safe.
- `EnumMap`/`EnumSet` iterators are weakly consistent (no `ConcurrentModificationException`).
- Don't use `ordinal()` yourself for persistence — reordering constants changes it. Let `EnumMap` use it internally.
- `BitSet` is only for **non-negative ints**; if the ids are sparse and huge (e.g. up to 2³¹) a bitset wastes memory.
- `BitSet` classic uses: Sieve of Eratosthenes, "visited" flags, permission/feature masks, dedup of int ids.

**Remember:** *Enum keys → `EnumMap`/`EnumSet`; dense int flags → `BitSet`. Array/bit tricks beat hashing.*

---

### Q53. `WeakHashMap` and `IdentityHashMap` — what are they and when do you use them?

**Say this first:** "`WeakHashMap` lets the GC drop entries whose key is no longer used elsewhere; `IdentityHashMap` compares keys with `==`, not `equals`."

**Answer:**
- **`WeakHashMap`:** keys are held through `WeakReference`. If a key is no longer *strongly* reachable from anywhere else, the GC may clear it, and the map removes the entry automatically (it polls a `ReferenceQueue` during later calls). Values are held strongly.
  - **Use for:** metadata/caches *attached to objects you don't own* — canonicalising maps, per-object listener data — where the data must not keep the key alive.
- **`IdentityHashMap`:** uses `System.identityHashCode` and `==` for keys. It stores keys and values in one flat array using **linear probing** (open addressing), not buckets.
  - **Use for:** object-graph algorithms where two *equal but distinct* objects must be treated separately — deep copy, serialisation, "visited" sets in graph traversal, proxies. `Collections.newSetFromMap(new IdentityHashMap<>())` gives an identity set.

```java
Map<Object, String> meta = new WeakHashMap<>();
Object key = new Object();
meta.put(key, "info");
key = null;            // now only weakly reachable
System.gc();           // entry may disappear (not guaranteed)

Map<String, Integer> id = new IdentityHashMap<>();
id.put(new String("a"), 1);
id.put(new String("a"), 2);      // size() == 2 : different identities
```

**Pitfalls / follow-up probes:**
- **Value strongly references its key** → the key never becomes weakly reachable → memory leak. The classic mistake.
- Keys like string literals, `Integer` cache values (−128..127), or enum constants are never collected, so entries never vanish.
- `WeakHashMap` is **not a reliable cache** — entries can vanish at any GC. For caching use Caffeine/soft references with size limits.
- `IdentityHashMap` deliberately **violates the `Map` contract** (uses `==` instead of `equals`). Allows null key and values. Not thread-safe.
- `WeakHashMap` is also not thread-safe; wrap with `Collections.synchronizedMap` if shared.
- Related: `ThreadLocalMap` uses weak keys too — see ThreadLocal leaks.

**Remember:** *Weak = "don't keep my key alive"; Identity = "`==` instead of `equals`".*

---

### Q54. `CopyOnWriteArrayList` and `CopyOnWriteArraySet` — how do they work and when to use them?

**Say this first:** "Every write copies the whole array; reads and iteration are lock-free and never throw `ConcurrentModificationException`."

**Answer:**
- **How:** the list holds a `volatile Object[] array`. A mutation (`add`, `set`, `remove`) takes an internal lock, **copies** the array, changes the copy, and publishes it as the new `array`. Readers keep using whatever array reference they already had.
- **Iterators work on a snapshot** of the array taken at creation. They never throw `ConcurrentModificationException`, never see later changes, and don't support `remove()`/`add()` (they throw `UnsupportedOperationException`).
- **Cost:** `get`/`contains` O(1)/O(n) with no lock; every write is **O(n)** plus garbage.
- **`CopyOnWriteArraySet`** is backed by a `CopyOnWriteArrayList`; `add` is O(n) because it checks duplicates (`addIfAbsent`).
- **Use when:** reads/iterations vastly outnumber writes and the collection is small — e.g. **listener/observer lists**, a rarely changed list of routes or feature flags.

```java
List<Listener> listeners = new CopyOnWriteArrayList<>();
listeners.add(l);
for (Listener x : listeners) x.onEvent(e);   // safe even if another thread calls add/remove
```

**Pitfalls / follow-up probes:**
- Frequent writes or large lists → a lot of copying and GC → very slow. Use `ConcurrentHashMap.newKeySet()`, `ConcurrentLinkedQueue` or a `ReentrantReadWriteLock` instead.
- Iterators may be **stale**: a reader can process an element that was already removed.
- `addIfAbsent` and `addAllAbsent` exist for atomic "add if missing".
- Compare to fail-fast (`ArrayList`) and to weakly consistent (`ConcurrentHashMap`): COW iteration is a *true snapshot*.
- Compound actions (`if (!list.contains(x)) list.add(x)`) are still racy — use `addIfAbsent`.

**Remember:** *COW = copy on write; great for read-mostly small lists, terrible for write-heavy ones.*

---

### Q55. The `BlockingQueue` family — differences and the producer–consumer pattern.

**Say this first:** "A `BlockingQueue` makes `put` wait when full and `take` wait when empty, so the queue does the thread coordination for you."

**Answer:**
- **Why:** producer–consumer needs a buffer between threads. Hand-writing `wait/notify` is error-prone; `BlockingQueue` bundles the lock, conditions and signalling.
- **Four method styles:**

| | Throws | Special value | Blocks | Times out |
|---|---|---|---|---|
| Insert | `add` | `offer` | `put` | `offer(e, t, unit)` |
| Remove | `remove` | `poll` | `take` | `poll(t, unit)` |

| Implementation | Bounded? | Notes |
|---|---|---|
| `ArrayBlockingQueue` | Yes (fixed) | Array, **one lock**, optional fairness flag |
| `LinkedBlockingQueue` | Optional (default `Integer.MAX_VALUE`) | **Two locks** (put/take) → higher throughput |
| `PriorityBlockingQueue` | Unbounded | Heap ordering; `put` never blocks |
| `DelayQueue` | Unbounded | Elements implement `Delayed`; `take` only returns expired ones (schedulers, retries) |
| `SynchronousQueue` | Capacity **0** | Direct hand-off; `put` waits for a `take`. Used by `newCachedThreadPool` |
| `LinkedTransferQueue` | Unbounded | `transfer(e)` waits until a consumer receives it |
| `LinkedBlockingDeque` | Optional | Both ends (work-stealing, undo) |

```java
BlockingQueue<String> q = new ArrayBlockingQueue<>(100);

Runnable producer = () -> {
    try { for (String s : source) q.put(s); q.put("POISON"); }
    catch (InterruptedException e) { Thread.currentThread().interrupt(); }
};
Runnable consumer = () -> {
    try {
        for (String s; !(s = q.take()).equals("POISON"); ) process(s);
    } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
};
```

**Pitfalls / follow-up probes:**
- **Unbounded queues hide overload** — a fast producer fills memory until `OutOfMemoryError`. Prefer a **bounded** queue; it gives natural **back-pressure** (producers slow down).
- Shutdown: use a *poison pill* (special sentinel) per consumer, or interrupt.
- `ThreadPoolExecutor` behaviour depends on the queue: `SynchronousQueue` → creates threads up to max; unbounded `LinkedBlockingQueue` → `maxPoolSize` is never reached.
- Always restore the interrupt flag on `InterruptedException`.
- Null elements are not allowed.

**Remember:** *`put` waits when full, `take` waits when empty; prefer bounded queues for back-pressure.*

---

### Q56. `ConcurrentLinkedQueue` and `ConcurrentSkipListMap` — what are they for?

**Say this first:** "Non-blocking, lock-free structures built on CAS: a queue (`ConcurrentLinkedQueue`) and a concurrent **sorted** map (`ConcurrentSkipListMap`)."

**Answer:**
- **`ConcurrentLinkedQueue` / `ConcurrentLinkedDeque`:** unbounded linked queue based on the Michael–Scott non-blocking algorithm — threads retry with **CAS** (compare-and-set) instead of locking. `offer`/`poll` never block; `poll` returns `null` when empty. Use when you want a fast non-blocking hand-off and don't need blocking waits or back-pressure.
- **`ConcurrentSkipListMap` / `ConcurrentSkipListSet`:** a **skip list** = several layers of linked lists; the bottom layer has all keys sorted, upper layers are random "express lanes" (each node is promoted with probability ½). Search/insert/delete are expected **O(log n)**. Because it's linked, updates are done with CAS and no global lock.
- It implements `ConcurrentNavigableMap`, so it has `floorKey`, `ceilingKey`, `headMap`, `descendingMap`, etc. — the concurrent counterpart of `TreeMap`.

| Need | Pick |
|---|---|
| Concurrent, unsorted map | `ConcurrentHashMap` (O(1)) |
| Concurrent, sorted map / range queries | `ConcurrentSkipListMap` (O(log n)) |
| Single-threaded sorted map | `TreeMap` |
| Non-blocking FIFO | `ConcurrentLinkedQueue` |
| Blocking FIFO / back-pressure | `LinkedBlockingQueue`/`ArrayBlockingQueue` |

```java
ConcurrentNavigableMap<Long, Order> byTime = new ConcurrentSkipListMap<>();
byTime.put(ts, order);
Map.Entry<Long, Order> latestBefore = byTime.floorEntry(now);
```

**Pitfalls / follow-up probes:**
- `size()` on both is **O(n)** (traversal) and only approximate under concurrent change. Use `isEmpty()` instead of `size() == 0`.
- Iterators are weakly consistent (no `ConcurrentModificationException`).
- No `null` keys/values/elements.
- A skip list uses more memory than a tree, but is much easier to make concurrent — that's why the JDK has no concurrent red-black tree.
- Don't use `ConcurrentLinkedQueue` as a work queue for a thread pool when you need the workers to wait — they'd spin or need external signalling.

**Remember:** *Lock-free CAS structures: CLQ for FIFO, skip list for concurrent sorted maps.*

---

### Q57. `Iterator` vs `ListIterator` vs `Spliterator`.

**Say this first:** "`Iterator` walks forward once; `ListIterator` walks both ways and can modify a list; `Spliterator` (Java 8) can also **split** the data so parallel streams can process it."

**Answer:**
- **`Iterator<E>`:** `hasNext()`, `next()`, optional `remove()`, plus `forEachRemaining` (Java 8). Works for every `Collection`. One direction, one pass.
- **`ListIterator<E>`:** only for `List`. Adds `hasPrevious()`, `previous()`, `nextIndex()`, `previousIndex()`, **`set(e)`** (replace last returned) and **`add(e)`** (insert at cursor). It can start in the middle: `list.listIterator(n)`.
- **`Spliterator<T>`:** the engine under streams.
  - `tryAdvance(Consumer)` — process one element.
  - `forEachRemaining(Consumer)` — process the rest.
  - `trySplit()` — hand off roughly half the elements to a new spliterator (for fork/join parallelism).
  - `estimateSize()` and `characteristics()` (`ORDERED`, `SIZED`, `SORTED`, `DISTINCT`, `IMMUTABLE`, `CONCURRENT`, `SUBSIZED`, `NONNULL`) let the stream pipeline optimise.
- `ArrayList` and arrays split cheaply (index ranges) → parallelise well. `LinkedList`/`Iterator`-based sources split poorly.

```java
ListIterator<String> it = names.listIterator();
while (it.hasNext()) {
    String n = it.next();
    if (n.isBlank()) it.remove();
    else it.set(n.trim());          // modify in place, safe during iteration
}

// custom Iterable → usable in for-each and streams
class Range implements Iterable<Integer> {
    final int lo, hi;
    Range(int lo, int hi) { this.lo = lo; this.hi = hi; }
    public Iterator<Integer> iterator() {
        return new Iterator<>() {
            int cur = lo;
            public boolean hasNext() { return cur < hi; }
            public Integer next() { return cur++; }
        };
    }
}
Stream<Integer> s = StreamSupport.stream(new Range(0, 5).spliterator(), false);
```

**Pitfalls / follow-up probes:**
- Call `next()` before `remove()`; calling `remove()` twice in a row throws `IllegalStateException`.
- `Iterable.spliterator()` default wraps the iterator with unknown size → poor parallel splitting. Implement a real `Spliterator` if parallel performance matters.
- `for-each` over a `Collection` is just an `Iterator` loop — hence the `ConcurrentModificationException` rules.
- `Enumeration` is the legacy (Hashtable/Vector) cousin; it has no `remove`.

**Remember:** *Iterator = forward; ListIterator = bidirectional + modify; Spliterator = splittable for parallel streams.*

---

### Q58. Why does `ConcurrentModificationException` happen, and what are the correct ways to remove while iterating?

**Say this first:** "The collection's `modCount` changed behind the iterator's back. Remove via the iterator, `removeIf`, a filtered copy, or iterate a copy."

**Answer:**
- Fail-fast collections (`ArrayList`, `HashMap`, …) keep an int `modCount` incremented on each structural change (add/remove/resize). An iterator remembers `expectedModCount` when created. On every `next()` it compares them; a mismatch throws `ConcurrentModificationException` (CME).
- The check happens in **`next()`**, not `hasNext()`. That causes a famous quirk: removing the **second-to-last** element in a for-each *doesn't* throw, because `hasNext()` becomes false (`cursor == size`). CME is "best effort", never a guarantee, and not only about threads — a single thread can trigger it.

```java
for (String s : list) if (s.isEmpty()) list.remove(s);   // CME (usually)
```

**Four correct ways:**
```java
// 1. Iterator.remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) if (it.next().isEmpty()) it.remove();

// 2. removeIf (Java 8) — cleanest; ArrayList does one efficient pass
list.removeIf(String::isEmpty);

// 3. Build a new collection (streams/filter), then replace or reassign
list = list.stream().filter(s -> !s.isEmpty()).collect(Collectors.toList());

// 4. Iterate over a copy (or backwards by index) and modify the original
for (String s : new ArrayList<>(list)) if (s.isEmpty()) list.remove(s);
for (int i = list.size() - 1; i >= 0; i--) if (list.get(i).isEmpty()) list.remove(i);
```

For maps: `map.entrySet().removeIf(e -> e.getValue() == null)`, `map.keySet().removeAll(keys)`, `map.values().remove(x)`, or `map.replaceAll(...)` to change values.

**Pitfalls / follow-up probes:**
- Changing a value of an *existing* key (`map.put(existingKey, v)`) is **not** structural → no CME. Adding/removing keys is.
- `list.remove(int)` vs `list.remove(Object)` confusion when removing from `List<Integer>`.
- With multiple threads, fail-fast is no protection — use `ConcurrentHashMap`, `CopyOnWriteArrayList` (snapshot iterator) or proper locking.
- Option 4 (copy) is O(n) extra memory; `removeIf` avoids it.
- Streams over a collection that is modified inside the pipeline can also throw CME.

**Remember:** *`modCount` mismatch → CME. Remove via `Iterator.remove()`, `removeIf`, filtered copy, or iterate a copy.*

---

### Q59. `HashMap` thresholds (8 / 64 / 6), load factor reasoning, hash spreading and collision handling.

**Say this first:** "Collisions are chained in a bucket; a long chain becomes a red-black tree at 8 (if the table has ≥ 64 slots) and goes back to a list at 6."

**Answer:**
- **Collision handling = separate chaining.** Entries with the same bucket index share a linked list (`Node.next`). (`IdentityHashMap` and `ThreadLocalMap` use *open addressing/linear probing* instead.)
- **Hash spreading:** the index is `(n - 1) & hash` — it uses only the **low bits** of the hash, because `n` is a power of two. So the high bits would be ignored and many hashes would collide. Java 8 mixes them in:
  ```java
  static final int hash(Object key) {
      int h;
      return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);   // fold high 16 bits into low 16
  }
  ```
  It is cheap (one shift + XOR) and protects tables whose `hashCode`s differ only in the upper bits (e.g. floats).
- **Constants:**

| Constant | Value | Meaning |
|---|---|---|
| `TREEIFY_THRESHOLD` | 8 | List in a bucket reaches 8 → candidate for a tree |
| `MIN_TREEIFY_CAPACITY` | 64 | If table < 64, **resize** instead of treeify (small tables collide because they're small) |
| `UNTREEIFY_THRESHOLD` | 6 | On resize/removal, a tree shrinking to ≤ 6 reverts to a list |
| `DEFAULT_LOAD_FACTOR` | 0.75 | Resize when `size > capacity × 0.75` |

- **Why 8 and 6 (not 8 and 8)?** A gap (hysteresis) stops a bucket from flipping between list and tree when size hovers near the limit. The `HashMap` source explains that with good hashes and load factor 0.75, bucket sizes follow a Poisson distribution; the chance of ≥ 8 in one bucket is about 0.00000006 — so trees only appear with bad hashes or attacks.
- **Why 0.75?** A trade-off: lower → fewer collisions but more wasted memory; higher → denser but slower lookups. 0.75 gives a good balance of time vs space.
- **Tree bins** order nodes by hash, then by `compareTo` if the key is `Comparable`, then by a tie-breaker (class name / identity hash). So keys that are `Comparable` give true O(log n).

**Pitfalls / follow-up probes:**
- A bad `hashCode` (constant, or poor spread) → everything in one bucket → O(n) (O(log n) with trees if keys are `Comparable`; still bad for non-comparable).
- Treeification is not about speeding up the average case — it's a **worst-case safeguard** against hash-flooding DoS.
- Resize cost is O(n); sizing the map up front avoids it (next question).
- Java 8 resize splits each bucket into a "low" and "high" list using `hash & oldCap` and preserves relative order (Java 7 reversed order and could loop forever under concurrent resize — a real production bug).

**Remember:** *Chain → tree at 8 (table ≥ 64), back to list at 6; index = `(n-1) & (h ^ h>>>16)`; resize at 0.75.*

---

### Q60. How do you size a `HashMap` correctly? (`new HashMap<>(n)` trap, `HashMap.newHashMap`)

**Say this first:** "`new HashMap<>(1000)` does **not** hold 1000 entries without resizing — you need `expected / 0.75 + 1`, or `HashMap.newHashMap(1000)` on Java 19+."

**Answer:**
- The constructor argument is the **initial capacity (table size)**, rounded up to a power of two (`tableSizeFor`). The resize **threshold** is `capacity × loadFactor`.
- If you plan to put `N` entries, you need `capacity × 0.75 ≥ N`. Passing `N` directly is too small:
  - `new HashMap<>(1000)` → table 1024 → threshold **768** → resizes at the 769th put (1024 → 2048).
- **Formula:** `capacity = (int) (expectedSize / 0.75f) + 1`.
- **Java 19+** adds factory methods that do this for you: `HashMap.newHashMap(n)`, `LinkedHashMap.newLinkedHashMap(n)`, `HashSet.newHashSet(n)`, `LinkedHashSet.newLinkedHashSet(n)`, `WeakHashMap.newWeakHashMap(n)`. (Guava's equivalent: `Maps.newHashMapWithExpectedSize`.)
- **Why bother:** each resize allocates a table twice as big and re-links every entry — O(n) work and garbage. For large known-size maps that is measurable.

```java
// before Java 19
Map<String, User> byId = new HashMap<>((int) (users.size() / 0.75f) + 1);
// Java 19+
Map<String, User> byId2 = HashMap.newHashMap(users.size());
users.forEach(u -> byId2.put(u.id(), u));
```

**Pitfalls / follow-up probes:**
- The table is allocated **lazily** on the first `put`, so `new HashMap<>()` is cheap for empty maps.
- Don't over-size: a huge initial capacity wastes memory and makes iteration (`O(capacity + size)`) slower.
- `new HashMap<>(otherMap)` / `putAll` already pre-sizes from the source size.
- Same idea for `ArrayList`: `new ArrayList<>(n)` or `ensureCapacity(n)` avoids repeated 1.5× growth copies.
- Collectors such as `toMap` don't know the size; for huge streams you may supply your own map factory with a pre-sized map.

**Remember:** *Initial capacity ≠ number of entries. Use `n/0.75 + 1` or `HashMap.newHashMap(n)`.*

---

### Q61. What do the `Collections` and `Arrays` utility classes offer? (what to know for an interview)

**Say this first:** "`Collections` for lists/sets/maps (sort, shuffle, search, wrappers); `Arrays` for arrays (sort, fill, copy, compare, `asList`)."

**Answer:**

| `Collections` method | Notes |
|---|---|
| `sort(list)` | Delegates to `list.sort` → `Arrays.sort` → **TimSort** (stable, O(n log n)) |
| `reverse`, `shuffle(list[, rnd])`, `swap`, `rotate` | In-place; `shuffle(list, new Random(seed))` for repeatable results |
| `binarySearch(list, key)` | List **must be sorted**; returns index, or `-(insertionPoint) - 1` if absent |
| `frequency(c, o)`, `max`, `min`, `disjoint` | O(n) scans |
| `nCopies(n, o)` | Immutable list of n references |
| `emptyList/Set/Map`, `singletonList`, `singleton` | Immutable shared instances — no allocation |
| `unmodifiableList/Set/Map` | Read-only **view** (see next question) |
| `synchronizedList/Map` | Every method locks one mutex; **you must manually `synchronized` while iterating** |
| `reverseOrder()` | Comparator for reverse natural order |

| `Arrays` method | Notes |
|---|---|
| `sort(int[])` | Dual-pivot quicksort (primitives); objects use TimSort; `parallelSort` for big arrays |
| `fill`, `setAll` | Initialise arrays |
| `copyOf`, `copyOfRange` | Copy / resize (shallow for object arrays) |
| `equals` / `deepEquals` | Element-wise / also nested arrays (`int[][]`) |
| `hashCode` / `deepHashCode`, `toString` / `deepToString` | Use `deep*` for multi-dimensional arrays |
| `binarySearch` | Sorted array required |
| `asList` | **Fixed-size view** backed by the array (set OK, add/remove → `UnsupportedOperationException`) |
| `mismatch`, `compare` (Java 9) | First differing index / lexicographic compare |
| `stream(arr)` | Array → stream |

```java
int[][] grid = new int[3][3];
System.out.println(Arrays.toString(grid));        // [[I@1b6d3586, ...]  – useless
System.out.println(Arrays.deepToString(grid));    // [[0, 0, 0], ...]

int[] a = {1, 2, 3}, b = {1, 2, 3};
a.equals(b);               // false  (reference equality)
Arrays.equals(a, b);       // true
```

**Pitfalls / follow-up probes:**
- `Arrays.asList(intArray)` gives `List<int[]>` of size 1 (primitive arrays don't auto-box) — use `Arrays.stream(a).boxed()` or `IntStream`.
- `Arrays.asList(array)` writes through to the original array.
- `Collections.binarySearch` on an unsorted list gives undefined results.
- `Arrays.sort(Integer[], Comparator)` is allowed but `Arrays.sort(int[], Comparator)` is not (needs boxed type).
- `array.length` is a field; `String.length()`, `List.size()` — a favourite warm-up trap.

**Remember:** *`Collections` = list algorithms + wrappers; `Arrays` = array algorithms; `deep*` for nested arrays; `asList` is a fixed-size view.*

---

### Q62. Unmodifiable view vs immutable copy — `Collections.unmodifiableList` vs `List.copyOf` vs `List.of`.

**Say this first:** "`unmodifiableList` is a read-only **window** onto a list that can still change; `List.copyOf` and `List.of` are truly immutable **snapshots**."

**Answer:**
- **`Collections.unmodifiableList(list)`** wraps `list`. You can't modify *through the wrapper*, but whoever holds the original can — and the wrapper reflects it.
- **`List.copyOf(coll)`** (Java 10) makes an independent immutable copy. If the argument is already an immutable list from `List.of`/`copyOf`, it returns **the same instance** (no copy).
- **`List.of(...)`** (Java 9) creates an immutable list directly. Same for `Set.of`, `Map.of`, `Map.ofEntries`, `Set.copyOf`, `Map.copyOf`.
- **`stream.toList()`** (Java 16) → unmodifiable list (allows nulls); `Collectors.toUnmodifiableList()` (Java 10) rejects nulls.

```java
List<String> src = new ArrayList<>(List.of("a", "b"));
List<String> view = Collections.unmodifiableList(src);
List<String> copy = List.copyOf(src);

src.add("c");
view.size();     // 3  — view sees the change
copy.size();     // 2  — snapshot
view.add("x");   // UnsupportedOperationException
```

| | `unmodifiableList` | `List.copyOf` / `List.of` |
|---|---|---|
| Copies data? | No (view) | Yes (copyOf) / n/a |
| Reflects source changes? | **Yes** | No |
| Null elements | Allowed | **Rejected** (`NullPointerException`) |
| Truly immutable? | No | Yes (structure) |

**Pitfalls / follow-up probes:**
- Both are **shallow**: the *elements* can still be mutable objects. Real immutability needs immutable elements (records, `String`).
- Defensive copy rule for immutable classes: copy on the way **in** (`List.copyOf(param)`) and return an immutable list on the way out. Returning `unmodifiableList(this.items)` is fine only if you accept that later internal changes leak out as live updates.
- `List.of(...).contains(null)` throws `NullPointerException`.
- `Set.of`/`Map.of` throw `IllegalArgumentException` on duplicate elements/keys, and their **iteration order is deliberately unspecified and varies between JVM runs** — don't depend on it.
- `Arrays.asList` is neither: it's fixed-size but writable (`set` works).

**Remember:** *View = live window; copy = frozen snapshot. Both shallow.*

---

### Q63. What are `SequencedCollection` and `SequencedMap` in Java 21?

**Say this first:** "Java 21 (JEP 431) added a common API for collections with a defined encounter order: `getFirst()`, `getLast()`, `addFirst`, `removeLast`, and `reversed()`."

**Answer:**
- **The problem before Java 21:** getting the last element was different for every type — `list.get(list.size()-1)`, `deque.getLast()`, `sortedSet.last()`, and `LinkedHashSet` had **no** way except iterating everything. There was no common type for "ordered" collections.
- **New interfaces:**
  - `SequencedCollection<E> extends Collection<E>`: `reversed()`, `addFirst`, `addLast`, `getFirst`, `getLast`, `removeFirst`, `removeLast`.
  - `SequencedSet<E>` (for `LinkedHashSet`, `SortedSet`/`NavigableSet`).
  - `SequencedMap<K,V>`: `firstEntry`, `lastEntry`, `pollFirstEntry`, `pollLastEntry`, `putFirst`, `putLast`, `reversed()`, plus `sequencedKeySet/Values/EntrySet`.
- **Who implements what:** `List` and `Deque` → `SequencedCollection`; `LinkedHashSet`, `SortedSet` → `SequencedSet`; `LinkedHashMap`, `SortedMap` (e.g. `TreeMap`) → `SequencedMap`. **`HashSet`/`HashMap` are not** (no defined order).
- **`reversed()`** returns a **live view**, not a copy — changes show through both ways.

```java
List<Integer> list = new ArrayList<>(List.of(1, 2, 3));
list.getFirst();        // 1   (before 21: list.get(0))
list.getLast();         // 3   (before 21: list.get(list.size() - 1))
list.reversed();        // [3, 2, 1] — view, no copy

LinkedHashMap<String,Integer> m = new LinkedHashMap<>();
m.put("a", 1); m.put("b", 2);
m.putFirst("z", 0);     // z moves to the front
m.lastEntry();          // b=2
```

**Pitfalls / follow-up probes:**
- `getFirst()/getLast()` on an **empty** list throw `NoSuchElementException` (the old `get(0)` threw `IndexOutOfBoundsException`).
- Some operations are unsupported where order is fixed by sorting: `TreeSet.addFirst(x)`/`TreeMap.putFirst` throw `UnsupportedOperationException`; so do mutators on immutable lists from `List.of`.
- Source-compat caveat: adding default methods to `List` can clash with your own classes/interfaces that declare `getFirst`/`reversed` with a different return type.
- Java 21 is LTS — expect this as a "what's new" interview question.
- Streams pre-21 could not reverse cheaply; `list.reversed().stream()` now can.

**Remember:** *Java 21: one API for "first/last/reversed" over lists, deques, linked and sorted sets/maps.*

---

### Q64. Sorting in Java — TimSort vs dual-pivot quicksort, stability and complexity.

**Say this first:** "Objects → **TimSort** (stable merge-sort hybrid). Primitives → **dual-pivot quicksort** (unstable, but stability is meaningless for primitives)."

**Answer:**
- **Stable sort** = elements that compare equal keep their original relative order. This matters when you sort by several keys in turn (sort by city, then stable-sort by name).
- **`Arrays.sort(Object[])`, `Collections.sort`, `List.sort`:** **TimSort** (Python's algorithm, in Java since 7).
  - Finds existing ascending/descending **runs**, extends short runs with binary insertion sort, then merges runs using a stack with invariants.
  - **O(n log n) worst case, O(n) on already-sorted/nearly-sorted data**, needs extra space (up to n/2), **stable**.
  - `List.sort` copies to an array, sorts, and writes back.
- **`Arrays.sort(int[]/long[]/double[]…)`:** **dual-pivot quicksort** (Yaroslavskiy). Average O(n log n), in-place, not stable, very cache-friendly. Older versions could degrade to O(n²) on adversarial input; the rewrite in JDK 14 added a heap-sort fallback so the worst case is also O(n log n).
- **`Arrays.parallelSort`** splits the array, sorts pieces in the common pool, merges; helps for large arrays (thousands+).
- **Stream `sorted()`** is stable for ordered streams.

```java
people.sort(Comparator.comparing(Person::city)
                      .thenComparing(Person::name)
                      .thenComparingInt(Person::age));
```

**Pitfalls / follow-up probes:**
- `IllegalArgumentException: Comparison method violates its general contract!` → your comparator is inconsistent (not symmetric/transitive). Classic cause: `(a, b) -> a.x - b.x` overflowing; use `Integer.compare(a.x, b.x)`.
- Comparator must handle `null`s explicitly (`Comparator.nullsFirst/nullsLast`).
- `Comparator.reversed()` reverses the whole comparator, so place it carefully in a `thenComparing` chain.
- `Collections.sort` on a `LinkedList` still copies to an array — sorting cost isn't improved by the list type.
- Need only top-K or "smallest k"? A `PriorityQueue` (O(n log k)) beats a full sort.
- `TreeMap`/`TreeSet` keep data sorted on insertion; sort once at the end is usually cheaper.

**Remember:** *Objects: TimSort, stable, O(n log n), O(n) if nearly sorted. Primitives: dual-pivot quicksort.*

---

### Q65. Time-complexity cheat sheet and "which collection for which scenario?"

**Say this first:** "Start from the access pattern — by index, by key, in order, as a queue, or concurrent — then pick the structure."

**Answer:**

| Collection | Get / contains | Add | Remove | Notes |
|---|---|---|---|---|
| `ArrayList` | get O(1); contains O(n) | end amortised O(1); middle O(n) | O(n) | Default list |
| `LinkedList` | get O(n) | ends O(1) | O(1) at known node | Rarely best |
| `ArrayDeque` | peek ends O(1) | ends O(1) | ends O(1) | Stack/queue |
| `HashMap` / `HashSet` | O(1) avg (O(log n) worst) | O(1) avg | O(1) avg | No order |
| `LinkedHashMap` / `LinkedHashSet` | O(1) | O(1) | O(1) | Insertion/access order |
| `TreeMap` / `TreeSet` | O(log n) | O(log n) | O(log n) | Sorted, range queries |
| `PriorityQueue` | peek O(1); contains O(n) | O(log n) | poll O(log n); remove(o) O(n) | Heap |
| `EnumMap` / `EnumSet` | O(1) | O(1) | O(1) | Enum keys |
| `ConcurrentHashMap` | O(1) avg, lock-free reads | O(1) | O(1) | Concurrent map |
| `ConcurrentSkipListMap` | O(log n) | O(log n) | O(log n) | Concurrent sorted |
| `CopyOnWriteArrayList` | get O(1) | O(n) | O(n) | Read-mostly |

**Scenario → choice:**
- Ordered list, random access → `ArrayList`.
- Unique items, fast membership → `HashSet`; keep insertion order → `LinkedHashSet`; sorted → `TreeSet`.
- Key → value lookup → `HashMap`; need ordered iteration → `LinkedHashMap`/`TreeMap`.
- LRU cache → `LinkedHashMap` (access order); production → Caffeine.
- Stack or FIFO queue (single thread) → `ArrayDeque`.
- Always want the smallest/most urgent next → `PriorityQueue`.
- Enum keys → `EnumMap`/`EnumSet`. Dense int flags → `BitSet`.
- Shared map, many threads → `ConcurrentHashMap`. Shared sorted map → `ConcurrentSkipListMap`.
- Producer–consumer hand-off → `ArrayBlockingQueue`/`LinkedBlockingQueue`.
- Listeners (rare writes) → `CopyOnWriteArrayList`.
- "Find the floor/ceiling/nearest" → `TreeMap`/`TreeSet`.
- Immutable constants → `List.of`/`Map.of`.

**Pitfalls / follow-up probes:**
- Big-O hides constants — `ArrayList` beats `LinkedList` through cache locality even when the "O" looks worse.
- `HashMap` worst case: O(n) before Java 8, O(log n) after treeification (needs `Comparable` keys to be good).
- Interviewers often follow with: "what if the key class has a bad `hashCode`?" or "what if the data is read-only after startup?" (→ immutable `Map.copyOf`).
- State the trade-off aloud: memory vs speed vs ordering vs thread-safety.

**Remember:** *Index → list; key → map; order → Linked/Tree; queue → Deque/PQ; threads → Concurrent*/Blocking*.*

---

### Q66. Map iteration styles, and pitfalls of `getOrDefault`, `putIfAbsent`, `computeIfAbsent`, `merge`.

**Say this first:** "Iterate `entrySet()` (one lookup per entry), and remember that each of the Java 8 map helpers has a slightly different null/absent rule."

**Answer — iteration:**
```java
for (Map.Entry<String, Integer> e : map.entrySet()) { e.getKey(); e.getValue(); }   // best
map.forEach((k, v) -> ...);                       // Java 8, concise
for (String k : map.keySet()) { map.get(k); }     // extra lookup for every key
for (Integer v : map.values()) { ... }            // values only
map.entrySet().stream()...;                       // stream pipeline
```
- `entrySet()` hands you the key and value together. `keySet()` + `get(k)` does an extra hash lookup per key (O(log n) each on a `TreeMap`).
- To remove while iterating use `entrySet().iterator()` + `remove()` or `entrySet().removeIf(...)`.

**Answer — helper methods:**

| Method | Behaviour | Gotcha |
|---|---|---|
| `getOrDefault(k, d)` | Returns `d` only if the key is **absent** | If the key is mapped to `null`, returns `null`; does **not** store `d` |
| `putIfAbsent(k, v)` | Puts if absent/null-mapped; **returns the old value** (null if it inserted) | `v` is built eagerly even when unused |
| `computeIfAbsent(k, fn)` | Calls `fn` only if absent; stores and returns result | If `fn` returns `null`, nothing is stored |
| `computeIfPresent` / `compute` | Remap existing / any key | Returning `null` **removes** the key |
| `merge(k, v, fn)` | Absent → put `v`; else `fn(old, v)` | `v` must be non-null; `fn` returning `null` removes the key |

```java
Map<String, List<String>> groups = new HashMap<>();
groups.computeIfAbsent(dept, d -> new ArrayList<>()).add(name);   // grouping idiom

counts.merge(word, 1, Integer::sum);                              // counting idiom
```

**Pitfalls / follow-up probes:**
- **Recursive `computeIfAbsent`** — the mapping function modifies the same map (e.g. memoised Fibonacci). `HashMap` (Java 9+) throws `ConcurrentModificationException`; Java 8 could corrupt the map or loop; `ConcurrentHashMap` may livelock/deadlock or throw `IllegalStateException: Recursive update`. Compute outside, or use `get` + `put`.
- `computeIfAbsent` with an expensive function in `ConcurrentHashMap` blocks the bucket; with `HashMap` it isn't thread-safe at all.
- `Map.of`/`ConcurrentHashMap` reject null; `HashMap` accepts it — so `merge` null rules differ per implementation.
- `getOrDefault(k, new ArrayList<>())` doesn't put the list in the map — mutating it loses the data.
- `entry.setValue` works on `HashMap` entries, but entries from `Map.entry(k, v)` are immutable.

**Remember:** *Iterate `entrySet`; `compute*`/`merge` return `null` ⇒ remove; never mutate the map inside `computeIfAbsent`.*

---

### Q67. How do you sort a `Map` by value, count frequencies and find duplicates?

**Say this first:** "Maps aren't sortable by value directly — stream the entries, sort them, and collect into a `LinkedHashMap` to keep the order."

**Answer — sort by value (descending), keep order:**
```java
Map<String, Integer> sorted = map.entrySet().stream()
    .sorted(Map.Entry.<String, Integer>comparingByValue().reversed()
            .thenComparing(Map.Entry.comparingByKey()))      // tie-breaker for equal values
    .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue,
             (a, b) -> a, LinkedHashMap::new));              // LinkedHashMap preserves order
```
A plain `HashMap`/`toMap()` result would lose the order. (Java 21: `LinkedHashMap` is a `SequencedMap`, handy for `firstEntry`.)

**Answer — frequency count:**
```java
// Java 8 loop
Map<String, Integer> freq = new HashMap<>();
for (String w : words) freq.merge(w, 1, Integer::sum);

// Streams
Map<String, Long> freq2 = words.stream()
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
```

**Answer — duplicates:**
```java
// 1. Set.add returns false for an element already present
Set<Integer> seen = new HashSet<>();
Set<Integer> dups = nums.stream().filter(n -> !seen.add(n)).collect(Collectors.toSet());

// 2. From frequency map
List<String> dupWords = freq2.entrySet().stream()
    .filter(e -> e.getValue() > 1).map(Map.Entry::getKey).toList();

// 3. First non-repeating character – keep encounter order
Map<Character, Integer> m = new LinkedHashMap<>();
for (char c : s.toCharArray()) m.merge(c, 1, Integer::sum);
m.entrySet().stream().filter(e -> e.getValue() == 1).findFirst();
```
Top-K most frequent: frequency map + `PriorityQueue` (min-heap of size k) → O(n log k), or sort the entries as above.

**Pitfalls / follow-up probes:**
- `Collectors.toMap` without a merge function throws `IllegalStateException: Duplicate key` — always give a merge function when keys may repeat.
- `Collectors.counting()` returns `Long`, not `Integer`.
- A stateful lambda (`seen.add`) inside a stream is acceptable only for sequential streams; in parallel streams use `groupingBy` instead.
- `Collections.frequency(list, x)` is O(n) — calling it per element is O(n²).
- Sorting by value on a `TreeMap` doesn't work — `TreeMap` sorts by key only.
- `distinct()` removes duplicates but doesn't tell you *which* elements were duplicated.

**Remember:** *Sort a map = stream entries → sort → `LinkedHashMap`; count = `merge` or `groupingBy(counting())`; duplicates = `Set.add` returns false.*

---

# 3. Java 8 → Java 21+ Features

### Q68. What did Java 8 actually give us?

Lambdas, functional interfaces, Streams, `Optional`, `default`/`static` interface methods, method references, new Date-Time API (`java.time`), `CompletableFuture`, `StringJoiner`, `Collectors`, Nashorn (now gone), PermGen → Metaspace, `StampedLock`, `LongAdder`, parallel array sort.

**What each one solved (one line each):**
- **Lambdas / method references** — pass behaviour as a value: `list.sort((a, b) -> a.age() - b.age())` instead of a 5-line anonymous class.
- **Streams** — declarative pipelines over collections (`filter → map → collect`) instead of manual loops; can be run in parallel.
- **`Optional`** — makes "might be absent" explicit in a return type rather than returning `null`.
- **`default`/`static` interface methods** — let libraries add methods to interfaces (e.g. `List.sort`, `Collection.stream`) without breaking every implementer.
- **`java.time`** — immutable, thread-safe replacement for the buggy `Date`/`SimpleDateFormat` (`LocalDate`, `Instant`, `ZonedDateTime`, `Duration`).
- **`CompletableFuture`** — composable async calls (`thenCombine`, `exceptionally`) instead of blocking `Future.get()`.
- **PermGen → Metaspace** — class metadata moved to native memory so it no longer hits the fixed-size `PermGen` `OutOfMemoryError`.
- **`LongAdder` / `StampedLock`** — lower-contention counters and optimistic read locks for high concurrency.

**Remember:** **L-S-O-D-D** = *Lambdas, Streams, Optional, Default methods, Date-time.*

---

### Q69. What is a functional interface? Name the built-in ones.

An interface with **exactly one abstract method** (SAM), marked `@FunctionalInterface` (optional but compiler-checked). `default`/`static`/`Object` methods don't count.

| Interface | Signature | Use |
|---|---|---|
| `Function<T,R>` | `R apply(T)` | map / transform |
| `BiFunction<T,U,R>` | `R apply(T,U)` | two-arg transform |
| `Predicate<T>` | `boolean test(T)` | filter |
| `Consumer<T>` | `void accept(T)` | forEach / side effect |
| `Supplier<T>` | `T get()` | lazy value, `orElseGet` |
| `UnaryOperator<T>` | `T apply(T)` | `replaceAll` |
| `BinaryOperator<T>` | `T apply(T,T)` | `reduce` |
| `Runnable` | `void run()` | task |
| `Callable<V>` | `V call() throws` | task with result |

Composition: `predicate.and(...).or(...).negate()`, `function.andThen(...).compose(...)`.

**Remember:** **F-P-C-S** = *Function transforms, Predicate tests, Consumer consumes, Supplier supplies.*

---

### Q70. Lambda vs anonymous inner class.

| | Lambda | Anonymous class |
|---|---|---|
| `this` | Enclosing instance | The anonymous object itself |
| Bytecode | `invokedynamic` + `LambdaMetafactory` (no extra .class file) | Separate `Outer$1.class` |
| State | Stateless, no fields | Can have fields |
| Applies to | Functional interfaces only | Any interface / abstract class |
| Capture | Effectively final locals | Same |

**Plain-language explanation:** an anonymous class is a full object-in-disguise (its own `.class`, its own `this`). A lambda is just "a block of code with parameters" the compiler turns into a private method, and `invokedynamic` links it to the functional interface **at runtime** (first call), which is smaller and lets the JVM optimise later.

```java
class Demo {
    String name = "outer";
    void run() {
        Runnable anon = new Runnable() { String name = "anon";
            public void run() { System.out.println(this.name); } };   // prints "anon"
        Runnable lam = () -> System.out.println(this.name);           // prints "outer"
    }
}
```
**Capture rule:** a lambda can read a local variable only if it is *effectively final* (never reassigned) — because the lambda gets a **copy** of the value, and two diverging copies would be confusing. Need a mutable counter? Use `AtomicInteger` (or restructure with a stream `reduce`).

**Remember:** *Lambdas are invokedynamic, keep enclosing `this`, and produce no class files.*

---

### Q71. Explain the Stream API pipeline. Intermediate vs terminal operations.

**Pipeline = source → 0..n intermediate (lazy) → 1 terminal (eager).**

- **Intermediate:** `filter, map, flatMap, mapMulti, distinct, sorted, peek, limit, skip, takeWhile, dropWhile, boxed`.
- **Terminal:** `forEach, collect, reduce, count, min, max, findFirst, findAny, anyMatch, allMatch, noneMatch, toList, toArray, iterator`.
- **Short-circuiting:** `limit, findFirst, findAny, anyMatch, allMatch, noneMatch, takeWhile`.
- **Stateful:** `sorted, distinct, limit, skip` — these need to buffer, so they hurt parallel performance.

**Laziness demo:**
```java
Stream<String> s = list.stream().filter(x -> { System.out.println("f " + x); return true; });
// prints NOTHING until a terminal op runs
```
Streams are **single-use** — reuse throws `IllegalStateException: stream has already been operated upon or closed`.

**Remember:** *Nothing happens until the terminal op; a stream is a one-shot pipeline.*

---

### Q72. `map` vs `flatMap`.

- `map`: 1 → 1 (`Stream<List<String>>` stays nested).
- `flatMap`: 1 → many, then flattens one level.

```java
List<Order> orders = ...;
List<Item> allItems = orders.stream()
        .flatMap(o -> o.getItems().stream())   // flatten List<Item> per order
        .toList();

// Optional flatMap: avoid Optional<Optional<T>>
Optional<String> city = Optional.ofNullable(user)
        .flatMap(User::getAddress)   // returns Optional<Address>
        .map(Address::getCity);
```

**Remember:** *map = transform, flatMap = transform + un-nest.*

---

### Q73. Show the `Collectors` you actually use in production.

```java
// group + count
Map<String, Long> byDept = emps.stream()
    .collect(Collectors.groupingBy(Employee::getDept, Collectors.counting()));

// group + downstream map + list
Map<String, List<String>> namesByDept = emps.stream()
    .collect(Collectors.groupingBy(Employee::getDept,
             Collectors.mapping(Employee::getName, Collectors.toList())));

// group + sum / avg / stats
Map<String, Double> avgSalary = emps.stream()
    .collect(Collectors.groupingBy(Employee::getDept,
             Collectors.averagingDouble(Employee::getSalary)));
IntSummaryStatistics stats = emps.stream()
    .collect(Collectors.summarizingInt(Employee::getAge));   // min,max,sum,avg,count

// partition (boolean key, always 2 entries)
Map<Boolean, List<Employee>> split = emps.stream()
    .collect(Collectors.partitioningBy(e -> e.getSalary() > 100000));

// toMap with merge function (avoids IllegalStateException on duplicate keys)
Map<String, Employee> byId = emps.stream()
    .collect(Collectors.toMap(Employee::getId, e -> e, (a, b) -> a, LinkedHashMap::new));

// join
String csv = emps.stream().map(Employee::getName)
    .collect(Collectors.joining(", ", "[", "]"));

// max per group
Map<String, Optional<Employee>> topPaid = emps.stream()
    .collect(Collectors.groupingBy(Employee::getDept,
             Collectors.maxBy(Comparator.comparing(Employee::getSalary))));

// teeing (Java 12) - two collectors, one pass
record Range(int min, int max) {}
Range r = nums.stream().collect(Collectors.teeing(
        Collectors.minBy(Integer::compare), Collectors.maxBy(Integer::compare),
        (a, b) -> new Range(a.orElse(0), b.orElse(0))));
```

**Two traps:**
1. `Collectors.toMap` throws on **duplicate keys** and on **null values** — always supply a merge function.
2. `Collectors.toList()` returns an unspecified mutable list; `stream.toList()` (Java 16+) returns an **immutable** list.

**Remember:** *groupingBy + downstream collector solves 80% of real reporting code.*

---

### Q74. When is `parallelStream()` a bad idea?

It uses the **common ForkJoinPool** (`n-1` threads where n = CPU cores). Avoid when:
- Data set is small (overhead > gain; rule of thumb: < ~10k elements of cheap work).
- Source doesn't split well (`LinkedList`, `Stream.iterate`, IO streams). `ArrayList`, arrays and `IntStream.range` split perfectly.
- The operation is **IO/blocking** — you starve the common pool, which is shared with `CompletableFuture` and the whole JVM.
- Order matters (`forEach` is unordered — use `forEachOrdered`, which kills the benefit).
- You mutate shared state (needs synchronization → serialization).
- Inside a **web request thread** in Spring Boot — you're already parallel across requests; adding parallel streams causes CPU contention and unpredictable latency.

**Fix for blocking work:** custom pool → `new ForkJoinPool(20).submit(() -> list.parallelStream()...).get()`, or better, use `CompletableFuture` with a dedicated executor, or **virtual threads** (Java 21).

**Remember:** *Parallel streams are for CPU-bound work on big splittable data — never for IO.*

---

### Q75. `Optional` — correct and incorrect usage.

**Do:**
```java
return repo.findById(id)
           .map(mapper::toDto)
           .orElseThrow(() -> new NotFoundException("Customer " + id));

String city = Optional.ofNullable(user.getAddress())
                      .map(Address::getCity)
                      .orElse("UNKNOWN");

opt.ifPresentOrElse(this::process, this::logMissing);       // Java 9
Optional<Config> c = primary.or(() -> fallback());          // Java 9
Stream<T> s = opt.stream();                                 // Java 9, flatten in streams
```

**Don't:**
- `optional.get()` without `isPresent()` → `NoSuchElementException`.
- `Optional` as a **field**, **method parameter**, or in an **entity** — it isn't `Serializable` and adds noise. Use it as a **return type** only.
- `Optional<List<T>>` — return an empty list instead.
- `orElse(expensiveCall())` — `orElse` is **eagerly evaluated**; use `orElseGet(() -> expensiveCall())`.
- `if (opt.isPresent()) opt.get()` — that's just a null check with extra steps.

**Remember:** *Optional is a return type; `orElse` is eager, `orElseGet` is lazy.*

---

### Q76. What's new in Java 9–21 that you've used?

| Version | Feature | Why it matters |
|---|---|---|
| 9 | JPMS modules, `List.of`, `Optional.stream`, private interface methods, `CompletableFuture` timeouts | Immutable factories used daily |
| 10 | `var` | Local type inference |
| 11 (LTS) | HTTP Client, `String.strip/isBlank/lines/repeat`, `Files.readString`, single-file run, `var` in lambdas | Dropped Apache HttpClient for the built-in one |
| 14 | Helpful NullPointerExceptions, switch expressions | NPE now tells you *which* variable was null |
| 15 | Text blocks (`"""`) | Readable SQL/JSON in tests |
| 16 | Records, `instanceof` pattern matching, `Stream.toList()` | DTO boilerplate gone |
| 17 (LTS) | Sealed classes, strong encapsulation of internals | Baseline for Spring Boot 3 |
| 19–20 | Virtual threads (preview), structured concurrency (preview) | |
| 21 (LTS) | **Virtual Threads (final)**, pattern matching for switch, record patterns, sequenced collections, generational ZGC | Massive throughput win for IO-bound services |

**Sequenced collections (21):** `SequencedCollection` with `getFirst/getLast/addFirst/addLast/reversed()` — finally uniform for `List`, `LinkedHashSet`, `LinkedHashMap`.

**Remember:** *8 → 11 → 17 → 21 are the LTS steps; 21's headline is virtual threads.*

---

### Q77. Virtual threads (Java 21) — explain them and their impact on server applications.

**What:** Lightweight threads managed by the JVM, not the OS. Millions can exist. A virtual thread runs on a **carrier** (platform) thread; when it blocks on IO, the JVM **unmounts** it and frees the carrier.

**Why it matters for Spring Boot:** the classic model is thread-per-request with a 200-thread Tomcat pool — 200 concurrent requests max, mostly sitting blocked on DB/HTTP calls. With virtual threads, blocking is cheap, so throughput scales with the downstream, not the pool size. You get reactive-level throughput while keeping simple, debuggable, blocking code.

```properties
# Spring Boot 3.2+
spring.threads.virtual.enabled=true
```
```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    var f1 = executor.submit(() -> userClient.get(id));
    var f2 = executor.submit(() -> orderClient.get(id));
    return new Profile(f1.get(), f2.get());
}
```

**Caveats:**
- **Pinning:** `synchronized` blocks around blocking IO pin the carrier thread (fixed in later JDKs; in 21 prefer `ReentrantLock`).
- **`ThreadLocal`** still works but is memory-heavy with millions of threads → prefer scoped values.
- **Don't pool them** — create one per task. Connection pools (Hikari) become the new bottleneck, so size them intentionally.
- CPU-bound work gains nothing.

**Remember:** *Virtual threads make blocking cheap — don't pool them, don't `synchronized` around IO.*

---

### Q78. Method references — the four kinds, and when should you prefer them over lambdas?

**Say this first:** "A method reference is just a lambda that does nothing except call one existing method — so I name the method instead of writing the lambda."

**Answer:** A method reference (`Type::method`) is shorthand for a lambda whose body is a single method call. It exists to remove noise: `s -> s.trim()` becomes `String::trim`. The compiler matches the method's parameters and return type against the target functional interface (the interface with one abstract method, like `Function`).

| Kind | Syntax | Equivalent lambda |
|---|---|---|
| Static method | `Integer::parseInt` | `s -> Integer.parseInt(s)` |
| Bound instance (receiver fixed *now*) | `System.out::println`, `this::validate` | `x -> System.out.println(x)` |
| Unbound instance (receiver is the *first argument*) | `String::toLowerCase` | `s -> s.toLowerCase()` |
| Constructor | `ArrayList::new`, `int[]::new` | `() -> new ArrayList<>()`, `n -> new int[n]` |

```java
List<String> raw = List.of(" a ", "b ", " c");
List<String> clean = raw.stream().map(String::trim).toList();      // unbound
List<Integer> nums = Stream.of("1", "2").map(Integer::valueOf).toList(); // static
Set<String> set = raw.stream().collect(Collectors.toCollection(TreeSet::new)); // constructor
String[] arr = raw.stream().toArray(String[]::new);                // array constructor
```

**Bound vs unbound — the one that confuses people:** in a *bound* reference (`obj::method`) the receiver `obj` is evaluated **once, when the reference is created**. In an *unbound* reference (`Type::method`) the receiver is supplied at **each call** as the first argument.

**When to prefer:** when the lambda is a pure pass-through and the method name explains intent (`map(Order::total)`). **When not to:** when you need to add arguments, negate, or the lambda reads better (`s -> s.length() > 3`). Don't force `Predicate.not(String::isBlank)` if a plain lambda is clearer to your team.

**Pitfalls / follow-up probes:**
- `Function<Integer,String> f = Integer::toString;` is **ambiguous** (static `toString(int)` vs instance `toString()`) → compile error. Use a lambda.
- A bound reference to a `null` receiver (`nullObj::foo`) throws `NullPointerException` **immediately** at creation, not at invocation.
- Overloaded methods (`println`) are resolved from the target type — usually fine, occasionally confusing for `map(String::valueOf)` with `char[]`.
- Method references to `this::method` capture `this` — can keep the object alive (leak risk in long-lived callbacks).

**Remember:** *Static, Bound, Unbound, Constructor — "bound" fixes the receiver now, "unbound" takes it as the first argument.*

---

### Q79. Default, static and private methods in interfaces — and how are inheritance conflicts resolved?

**Say this first:** "Default methods let interfaces evolve without breaking implementers; if two defaults clash, the class must resolve it explicitly."

**Answer:** Before Java 8, adding a method to an interface broke every implementing class. Java 8 added **`default` methods** (with a body, inherited by implementers) so the JDK could add `Collection.stream()` and `List.sort()` safely. **`static` methods** (Java 8) hold utilities that belong with the interface (`Comparator.comparing`). **`private` methods** (Java 9) let default/static methods share helper code without exposing it.

```java
interface Greeter {
    String name();
    default String greet() { return prefix() + name(); }
    private String prefix() { return "Hello, "; }          // Java 9, hidden helper
    static Greeter of(String n) { return () -> n; }        // factory, not inherited
}
```

**Conflict resolution — three rules, applied in order:**
1. **Class wins:** a method from a class (or superclass) beats any interface default.
2. **More specific interface wins:** if `B extends A` and both define the default, `B`'s is used.
3. **Otherwise it's a compile error** — the class must override and may pick one explicitly with `Interface.super.method()`.

```java
interface Flyer   { default String move() { return "fly"; } }
interface Swimmer { default String move() { return "swim"; } }
class Duck implements Flyer, Swimmer {
    @Override public String move() { return Flyer.super.move() + " & " + Swimmer.super.move(); }
}
```

**Pitfalls / follow-up probes:**
- A default method **cannot override** `equals`, `hashCode` or `toString` (the class always wins).
- **Static interface methods are not inherited** — call them as `Greeter.of(...)`, never via an implementing class or instance.
- Interface fields are implicitly `public static final` — constants only, no state. That is the real difference from abstract classes (see the abstract class vs interface question).
- Default methods are not a licence to build "multiple inheritance of state" — keep them small and behavioural.

**Remember:** *Class wins → most specific wins → else override and call `X.super.m()`.*

---

### Q80. Lambda deep dive: variable capture, "effectively final", and how lambdas actually run

**Say this first:** "Lambdas capture *values*, not variables — that's why locals must be effectively final; and they're linked at runtime with `invokedynamic`, not compiled to inner classes."

**Answer (capture):** A lambda can use variables from the enclosing method. The rules:
- **Local variables / parameters:** must be `final` or *effectively final* (assigned once, never changed). The lambda gets a **copy of the value**. If the original could change later, the copy and the original would disagree, and the lambda may even run on another thread after the method returned — so Java simply forbids it.
- **Instance and static fields:** no restriction, because the lambda reads them through `this` / the class. This is *allowed but dangerous* — mutating a field from a lambda in a parallel stream is a data race.

```java
int base = 10;
Function<Integer,Integer> add = x -> x + base;   // OK
// base++;                                       // would make the line above a compile error

int[] counter = {0};                             // the "array trick" compiles but is a code smell
list.forEach(x -> counter[0]++);                 // not thread-safe; prefer AtomicInteger or a reduction
```

**Answer (implementation):** The compiler turns the lambda body into a private synthetic method (e.g. `lambda$main$0`) and emits an **`invokedynamic`** instruction. On first execution the JVM calls a bootstrap method (`LambdaMetafactory`) which generates a small hidden class implementing the functional interface and links the call site. Benefits: no `Outer$1.class` files, smaller jars, and the JVM can optimise later.
- A **non-capturing** lambda is typically created once and reused; a **capturing** one allocates a new object per evaluation (implementation detail, don't rely on it).

**Pitfalls / follow-up probes:**
- A lambda's `this` is the enclosing instance; its parameter names **cannot shadow** locals (an anonymous class can).
- Don't rely on lambda identity: `==` between two lambdas is unspecified.
- Lambdas are **not serializable** unless the target interface extends `Serializable`.
- Stack traces show `lambda$...` frames — name complex logic as a method and use a method reference for readable traces.

**Remember:** *Capture copies values (so effectively final); `invokedynamic` links lambdas at runtime, no inner class files.*

---

### Q81. Stream laziness in practice: element-by-element processing, short-circuiting, ordering and stateful operations

**Say this first:** "A stream processes one element at a time through the whole pipeline — except stateful operations like `sorted`, which must see everything first."

**Answer:** Intermediate operations don't run when you call them; they only build a recipe. The terminal operation pulls elements through, and each element flows through *all* stages before the next one starts (called **vertical** or **loop-fusion** processing). This is why `findFirst` or `limit` can stop early even on an infinite source.

```java
Stream.of("a", "b", "c")
      .filter(s -> { System.out.println("filter " + s); return true; })
      .map(s -> { System.out.println("map " + s); return s.toUpperCase(); })
      .findFirst();
// filter a, map a   <- stops; b and c are never touched
```

**Stateless vs stateful:**
- **Stateless** (`filter`, `map`, `flatMap`, `peek`): decide per element, no memory.
- **Stateful** (`sorted`, `distinct`, `limit`, `skip`): need other elements. `sorted` is a **barrier** — it buffers *every* element before emitting one, so laziness is lost at that point and an infinite stream with `sorted()` never finishes.

**Encounter order:** a `List` or array stream has a defined order; a `HashSet` stream does not. Operations like `findFirst`, `limit`, `skip`, `distinct` respect order (cost extra in parallel); `findAny` and `unordered()` don't care and are cheaper in parallel. `sorted()` is stable for ordered streams; `distinct()` keeps the **first** occurrence.

**Rules of thumb for performance:** put `filter` **before** `map`/`sorted` to shrink the data early; put `limit` early when you can; avoid `sorted().limit(n)` on huge data if you only need the top-n (use a bounded priority queue instead).

**Pitfalls / follow-up probes:**
- `Stream.iterate(1, i -> i + 1).sorted().limit(5)` → hangs / `OutOfMemoryError`. `limit` must come **before** `sorted` here.
- `forEach` on a parallel stream gives no order guarantee; use `forEachOrdered` (kills much of the speed-up).
- The pipeline captures lambdas lazily, so a variable changed after building but before the terminal op can surprise you.

**Remember:** *Lazy, one element at a time, with `sorted` as the barrier; filter early, limit early.*

---

### Q82. `reduce()` vs `collect()` — what is "mutable reduction" and when does each fit?

**Say this first:** "`reduce` combines values into a new immutable result each step; `collect` accumulates into a mutable container."

**Answer:** Both fold a stream into one result. The difference is **how the intermediate result is held**.

**`reduce` — immutable reduction (three forms):**
```java
int sum = nums.stream().reduce(0, Integer::sum);                        // identity + operator
Optional<Integer> max = nums.stream().reduce(Integer::max);             // no identity -> Optional
int totalLen = words.stream().reduce(0, (acc, w) -> acc + w.length(), Integer::sum); // 3-arg
```
The 3-arg form is for when the result type (`Integer`) differs from the element type (`String`): `accumulator` folds an element in, `combiner` merges partial results from parallel threads.

**Rules for `reduce` to be correct (especially in parallel):**
- The operator must be **associative** (`(a op b) op c == a op (b op c)`) and **stateless**.
- The identity must be a true identity (`0` for `+`, `1` for `*`). `reduce(10, Integer::sum)` in a parallel stream adds `10` **once per chunk** — wrong answer.

**`collect` — mutable reduction:** takes a `supplier` (make container), `accumulator` (add element), `combiner` (merge containers).
```java
StringBuilder sb = words.stream()
    .collect(StringBuilder::new, StringBuilder::append, StringBuilder::append);
// Usually you use a ready-made Collector: toList(), joining(), groupingBy() ...
```

**Why not `reduce` for everything?** Using `reduce` to build a list or string copies the whole accumulator each step → **O(n²)**:
```java
words.stream().reduce("", String::concat);          // bad: new String each step
words.stream().collect(Collectors.joining());        // good: one StringBuilder
```

**Pitfalls / follow-up probes:**
- A custom collector: `Collector.of(supplier, accumulator, combiner, finisher, characteristics...)`.
- `reduce` with a non-associative operator (`a - b`) gives nondeterministic results in parallel.
- For simple numbers prefer `IntStream.sum()` / `average()` — clearer and no boxing.

**Remember:** *`reduce` = fold into a value; `collect` = pour into a container. Never build lists/strings with `reduce`.*

---

### Q83. Primitive streams — `IntStream`, `LongStream`, `DoubleStream`, `boxed()`, and why they exist

**Say this first:** "`Stream<Integer>` boxes every number into an object; `IntStream` keeps raw `int`s — faster and with number-specific methods."

**Answer:** `Stream<Integer>` stores `Integer` objects (boxing = wrapping a primitive in an object). That costs memory, GC and cache misses. `IntStream`, `LongStream`, `DoubleStream` avoid boxing and add numeric helpers: `sum()`, `average()`, `min()`, `max()`, `summaryStatistics()`, `range()`.

```java
int total = IntStream.rangeClosed(1, 100).sum();                     // 5050, no boxing
OptionalDouble avg = IntStream.of(3, 5, 8).average();               // OptionalDouble, empty if no data
IntSummaryStatistics st = emps.stream().mapToInt(Employee::age).summaryStatistics();

long vowels = "interview".chars().filter(c -> "aeiou".indexOf(c) >= 0).count(); // chars() is IntStream
List<Integer> boxed = IntStream.range(0, 5).boxed().toList();        // primitive -> object stream
String letters = IntStream.range(0, 3).mapToObj(i -> "x" + i).collect(Collectors.joining(","));
```

**Converting between them:**
| From → To | Method |
|---|---|
| `Stream<T>` → `IntStream` | `mapToInt(...)` (also `mapToLong`, `mapToDouble`) |
| `IntStream` → `Stream<T>` | `mapToObj(...)` or `boxed()` |
| `IntStream` → `LongStream` | `asLongStream()` |
| `int[]` → `IntStream` | `Arrays.stream(arr)` |

**Pitfalls / follow-up probes:**
- `Stream.of(intArray)` gives `Stream<int[]>` (one element: the array!). Use `Arrays.stream(intArray)` or `IntStream.of(...)`.
- `IntStream.sum()` **overflows silently**; for big totals use `mapToLong` first.
- `average()`/`max()` return `OptionalDouble`/`OptionalInt` — call `getAsDouble()`/`orElse(...)`, not `get()`.
- `IntStream` has no `collect(Collectors...)` for objects — `boxed()` first.
- `"é😀".chars()` yields UTF-16 units; use `codePoints()` for real Unicode characters.

**Remember:** *Use primitive streams for numbers; `boxed()` only when you must go back to objects.*

---

### Q84. Advanced `Collectors`: `groupingBy` variants, `filtering`/`flatMapping`, `collectingAndThen`, and `toMap` traps

**Say this first:** "`groupingBy` takes a classifier plus an optional map factory plus a downstream collector — most reporting code is just those three composed."

**Answer:** (Basic `counting`/`mapping`/`partitioningBy` are covered in the earlier Collectors question — this one is the follow-up depth.)

```java
// 1. choose the map type -> sorted keys
Map<String, Long> sorted = emps.stream().collect(
    Collectors.groupingBy(Employee::dept, TreeMap::new, Collectors.counting()));

// 2. multi-level grouping
Map<String, Map<Integer, List<Employee>>> byDeptAge = emps.stream().collect(
    Collectors.groupingBy(Employee::dept, Collectors.groupingBy(Employee::age)));

// 3. filtering as a downstream collector (keeps empty groups!)
Map<String, List<Employee>> rich = emps.stream().collect(Collectors.groupingBy(
    Employee::dept, Collectors.filtering(e -> e.salary() > 100_000, Collectors.toList())));

// 4. flatMapping (Java 9): flatten inside each group
Map<String, Set<String>> skillsByDept = emps.stream().collect(Collectors.groupingBy(
    Employee::dept, Collectors.flatMapping(e -> e.skills().stream(), Collectors.toSet())));

// 5. collectingAndThen: post-process the result
List<String> ro = names.stream()
    .collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList));
Map<String, Employee> top = emps.stream().collect(Collectors.groupingBy(Employee::dept,
    Collectors.collectingAndThen(Collectors.maxBy(Comparator.comparingDouble(Employee::salary)), Optional::get)));
```

**`filter` before `groupingBy` vs `filtering` inside it:** an upfront `filter` drops departments that have *no* matching employee; `Collectors.filtering` keeps the department with an **empty list**. Pick based on whether you need the empty key.

**`toMap` traps:**
- Duplicate key → `IllegalStateException: Duplicate key X (attempted merging values A and B)`. Supply a merge function `(a, b) -> a`.
- A `null` **value** → `NullPointerException` (it uses `HashMap.merge`). A null key from `groupingBy`'s classifier throws "element cannot be mapped to a null key".
- Result is a `HashMap` (unordered) unless you pass `LinkedHashMap::new` as the 4th argument.

**Other useful ones:** `toCollection(TreeSet::new)`, `toUnmodifiableList/Set/Map` (Java 10), `summingInt` vs `summarizingInt` (one number vs five stats), `groupingByConcurrent` (parallel, unordered), `counting()` returns **`Long`** (not `int`).

**Pitfalls / follow-up probes:** `partitioningBy` always returns both `true` and `false` keys; `groupingBy` returns only keys that occur. Prefer `Map.merge`-style thinking for frequency counts.

**Remember:** *classifier + (map factory) + downstream; `filtering` keeps empty groups; always give `toMap` a merge function.*

---

### Q85. Creating and bounding streams: `iterate`, `generate`, `takeWhile`/`dropWhile`, `ofNullable`, `toList`, `mapMulti`

**Say this first:** "Infinite streams are fine as long as something downstream — `limit`, `takeWhile`, or a short-circuiting terminal — stops them."

**Answer:**
```java
Stream.iterate(1, x -> x * 2).limit(5).toList();                 // [1, 2, 4, 8, 16]
Stream.iterate(1, x -> x < 100, x -> x * 2).toList();            // Java 9: seed, hasNext, next (like a for loop)
Stream.generate(UUID::randomUUID).limit(3).toList();             // supplier-based, no seed
IntStream.rangeClosed(1, 10).takeWhile(i -> i < 5).boxed().toList(); // [1,2,3,4]
IntStream.rangeClosed(1, 10).dropWhile(i -> i < 8).boxed().toList(); // [8,9,10]
Stream.ofNullable(maybeNull).count();                            // Java 9: 0 or 1 element
Stream.concat(a.stream(), b.stream());                           // join two streams
```

**`takeWhile` vs `filter`:** `filter` tests **every** element; `takeWhile` **stops at the first failure**. On `[1,2,9,3]` with `x < 5`, `filter` gives `[1,2,3]`, `takeWhile` gives `[1,2]`. They only make sense on **ordered** streams. `dropWhile` is the mirror: discards the leading run, then keeps everything.

**`Stream.toList()` (Java 16) vs `collect(Collectors.toList())`:** `toList()` returns an **unmodifiable** list and allows `null` elements; `Collectors.toUnmodifiableList()` rejects nulls; `Collectors.toList()` gives an unspecified mutable list.

**`mapMulti` (Java 16):** like `flatMap`, but instead of returning a stream you push zero or more results into a consumer. It avoids allocating a small stream per element, so it wins when each input yields **few** outputs.
```java
Stream.of(1, 2, 3).<Integer>mapMulti((n, sink) -> {
    if (n % 2 == 1) { sink.accept(n); sink.accept(n * 10); }
}).toList();   // [1, 10, 3, 30]
```
Use `flatMap` when the mapper already returns a stream or the logic is simple; `mapMulti` for imperative logic or tiny fan-out.

**Pitfalls / follow-up probes:**
- `Stream.generate(...)` or 2-arg `iterate` without a bound = infinite loop at the terminal op.
- `iterate` is inherently sequential — it parallelises poorly.
- `generate` with a stateful supplier is not safe in parallel streams.

**Remember:** *`iterate`/`generate` need a stopper; `takeWhile` stops at the first miss, `filter` never stops.*

---

### Q86. Classic Stream interview programs — Part 1: second highest salary, first non-repeating character, character frequency, find duplicates

**Say this first:** "Think: what is the stream source, what is the key, and which collector gives me the shape I need?"

**Answer:** Assume `record Employee(String name, String dept, int salary) {}`.

**1. Second highest salary (distinct!):**
```java
Optional<Integer> second = emps.stream().map(Employee::salary)
        .distinct().sorted(Comparator.reverseOrder()).skip(1).findFirst();
```
`distinct()` matters — otherwise two people on the top salary make "second" equal to the first.

**2. Employee with the second highest salary:**
```java
Optional<Employee> e = emps.stream()
        .sorted(Comparator.comparingInt(Employee::salary).reversed())
        .skip(1).findFirst();                      // note: ties not handled
```

**3. Character frequency in a string:**
```java
Map<Character, Long> freq = "mississippi".chars().mapToObj(c -> (char) c)
        .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
```

**4. First non-repeating character** — keep **insertion order** with `LinkedHashMap`:
```java
Optional<Character> first = "swiss".chars().mapToObj(c -> (char) c)
        .collect(Collectors.groupingBy(Function.identity(), LinkedHashMap::new, Collectors.counting()))
        .entrySet().stream().filter(en -> en.getValue() == 1L)
        .map(Map.Entry::getKey).findFirst();       // 'w'
```

**5. Find duplicates in a list:**
```java
Set<Integer> dups = nums.stream()
        .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()))
        .entrySet().stream().filter(en -> en.getValue() > 1)
        .map(Map.Entry::getKey).collect(Collectors.toSet());
// alternative: Set<Integer> seen = new HashSet<>();
//              Set<Integer> dups2 = nums.stream().filter(n -> !seen.add(n)).collect(Collectors.toSet());
```
The `seen.add` version is shorter but uses a **stateful predicate** — fine sequentially, **wrong in parallel**.

**6. Count vowels / check anagram / reverse words:**
```java
boolean anagram = Arrays.equals(a.chars().sorted().toArray(), b.chars().sorted().toArray());
String rev = Arrays.stream(sentence.split(" ")).reduce("", (acc, w) -> w + " " + acc).trim();
```

**Pitfalls / follow-up probes:**
- Always state the complexity: grouping is O(n); sort-based is O(n log n).
- Case/Unicode: normalise with `toLowerCase()`; use `codePoints()` for emoji.
- Interviewers often ask for the **non-stream** version too — know a plain `HashMap`/`merge` loop.

**Remember:** *`groupingBy(identity, counting())` is the frequency template; `LinkedHashMap` keeps order.*

---

### Q87. Classic Stream interview programs — Part 2: group by length, flatten, sort a map by value, even/odd, min/max/statistics

**Say this first:** "Most of these are one collector: `groupingBy`, `partitioningBy`, `flatMap`, or `summaryStatistics`."

**Answer:**
```java
// 1. Group words by length
Map<Integer, List<String>> byLen = words.stream().collect(Collectors.groupingBy(String::length));

// 2. Flatten a list of lists
List<Integer> flat = listOfLists.stream().flatMap(List::stream).toList();

// 3. Partition even / odd
Map<Boolean, List<Integer>> parts = nums.stream().collect(Collectors.partitioningBy(n -> n % 2 == 0));

// 4. Sort a Map by value (descending), preserving order in a LinkedHashMap
Map<String, Integer> sorted = map.entrySet().stream()
        .sorted(Map.Entry.comparingByValue(Comparator.reverseOrder()))
        .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue, (a, b) -> a, LinkedHashMap::new));

// 5. Max / min / statistics
int max = nums.stream().mapToInt(Integer::intValue).max().orElseThrow();
Employee oldest = emps.stream().max(Comparator.comparingInt(Employee::age)).orElseThrow();
IntSummaryStatistics s = nums.stream().mapToInt(i -> i).summaryStatistics(); // count,min,max,sum,avg

// 6. Highest-paid per department
Map<String, Optional<Employee>> top = emps.stream().collect(Collectors.groupingBy(
        Employee::dept, Collectors.maxBy(Comparator.comparingInt(Employee::salary))));

// 7. Sum of squares of even numbers, join names
int sq = nums.stream().filter(n -> n % 2 == 0).mapToInt(n -> n * n).sum();
String names = emps.stream().map(Employee::name).sorted().collect(Collectors.joining(", "));

// 8. Multi-field sort
emps.sort(Comparator.comparing(Employee::dept).thenComparing(Employee::salary, Comparator.reverseOrder()));
```

**Why the `LinkedHashMap` in #4?** Plain `toMap` creates a `HashMap`, which throws the sorted order away. This is the classic follow-up trap.

**Pitfalls / follow-up probes:**
- `partitioningBy` always has both keys (even if a list is empty); `groupingBy` does not.
- `max(...)` on an empty stream returns an empty `Optional` — handle it (`orElseThrow`, `orElse`).
- `Comparator.comparing(...).reversed()` after a chain can reverse the **whole** comparator — put `reverseOrder()` inside the specific `thenComparing`.
- Sorting a map by key: use `TreeMap`, or `Map.Entry.comparingByKey()`.

**Remember:** *Shape first: Map of lists → `groupingBy`; two buckets → `partitioningBy`; stats → `summaryStatistics`.*

---

### Q88. Streams vs for-loops: performance, readability, single-use streams and the `peek()` trap

**Say this first:** "Streams buy readability, not speed; a plain loop is often faster for small or simple work — measure with JMH before claiming otherwise."

**Answer (performance):** A stream pipeline allocates stage objects, lambdas and iterators, and `Stream<Integer>` boxes numbers. For small collections or tight numeric loops, an indexed `for` loop is usually faster and the JIT inlines it better. For large data, the cost is dominated by your work, so the difference vanishes. Never benchmark with a naive `System.nanoTime()` — warm-up and JIT make it lie; use **JMH**.

| Prefer streams when | Prefer loops when |
|---|---|
| Filter → map → collect/group transformations | Complex early exits, multiple `break`/`continue` |
| Declarative, side-effect-free logic | You mutate several local variables |
| Easy parallelism on large CPU-bound data | Index-based access, checked exceptions everywhere |
| Collectors (`groupingBy`) save a lot of code | Hot inner loops on primitives |

**Why a stream can't be reused:** a stream is a **single-pass pipeline over a data source**, not a data structure. After a terminal operation it is marked consumed; a second terminal op throws `IllegalStateException: stream has already been operated upon or closed`. To reuse the logic, rebuild it: keep the *source* (list) and a `Supplier<Stream<T>>`.
```java
Supplier<Stream<String>> s = () -> list.stream().filter(x -> x.length() > 2);
long n = s.get().count();
List<String> l = s.get().toList();
```

**`peek()` misuse:** `peek(Consumer)` exists **for debugging** (see elements mid-pipeline). Don't use it for business side effects because:
- It may **not run at all**: since Java 9, `count()` can compute the size from the source without traversing, skipping `peek`/`map`.
- With `limit`/short-circuit ops, fewer elements reach it than you think.
- In parallel streams it runs on many threads in no fixed order.

**Side-effects in general:** lambdas in `map`/`filter` should be **stateless** and **non-interfering** — don't modify the source list while streaming (`ConcurrentModificationException`), and don't add to an outside `ArrayList` with `forEach` (use `collect`; the outside list isn't thread-safe in parallel).

**Pitfalls / follow-up probes:** debugging stack traces through streams are deeper; teams new to streams may overuse long chains — break them into named steps or helper methods.

**Remember:** *Streams = clarity; loops = raw speed; streams are single-use; `peek` is for debugging only.*

---

### Q89. How do you handle checked exceptions inside lambdas and streams?

**Say this first:** "Standard functional interfaces don't declare `throws`, so a checked exception must be caught or wrapped inside the lambda."

**Answer:** `Function`, `Consumer`, `Predicate` etc. have no `throws` clause. So `files.stream().map(Files::readString)` does **not compile** — `readString` throws `IOException` (a checked exception, one the compiler forces you to handle). Options, from simplest:

**1. try/catch inside the lambda (fine for one-offs):**
```java
paths.stream().map(p -> {
    try { return Files.readString(p); }
    catch (IOException e) { throw new UncheckedIOException(e); }
}).toList();
```

**2. A small reusable wrapper (cleanest when it repeats):**
```java
@FunctionalInterface
interface ThrowingFunction<T, R> { R apply(T t) throws Exception; }

static <T, R> Function<T, R> unchecked(ThrowingFunction<T, R> f) {
    return t -> {
        try { return f.apply(t); }
        catch (RuntimeException e) { throw e; }
        catch (Exception e) { throw new RuntimeException(e); }
    };
}
List<String> texts = paths.stream().map(unchecked(Files::readString)).toList();
```

**3. Handle failure per element instead of aborting** — return a result object (or `Optional`) and filter:
```java
record Result<T>(T value, Exception error) {}
```
Map each element to a `Result`, then partition into successes/failures. Use this for batch jobs where one bad row must not kill the run.

**4. Use a plain loop** when the logic is mostly error handling — it's legitimate and clearer.

**Pitfalls / follow-up probes:**
- An unchecked exception thrown from a lambda **aborts the whole pipeline**; in a parallel stream, which exception surfaces is not guaranteed, and others may be lost.
- Wrap with the original exception as the **cause** so stack traces survive.
- "Sneaky throw" tricks (generics to hide checked exceptions) compile but defeat the compiler's safety net — avoid in team code.
- `Callable` allows `throws Exception` but `Runnable` doesn't — a classic executor question.

**Remember:** *Lambdas can't throw checked exceptions — wrap them into unchecked ones (with cause) or return a Result.*

---

### Q90. `Optional` advanced follow-ups: `orElse` vs `orElseGet`, `map` vs `flatMap`, anti-patterns, primitive optionals

**Say this first:** "Use `Optional` as a return type to force callers to think about absence — then chain `map`/`filter`/`orElseGet`, never `get()`."

**Answer:** (Basics are in the earlier `Optional` question; here are the traps interviewers add.)

**`orElse` vs `orElseGet` — the cost difference:**
```java
User u = repo.find(id).orElse(createDefault());        // createDefault() ALWAYS runs
User v = repo.find(id).orElseGet(() -> createDefault()); // runs only when empty
```
If the default is a constant, `orElse` is fine; if it hits a DB/service or has side effects, use `orElseGet`.

**`map` vs `flatMap`:** `map(f)` wraps `f`'s result in an `Optional` (and turns a `null` result into empty). If `f` already returns an `Optional`, you'd get `Optional<Optional<T>>` — use `flatMap`.

**`filter`, `or`, `ifPresentOrElse`, `orElseThrow()`:**
```java
Optional<String> valid = Optional.ofNullable(email).filter(e -> e.contains("@"));
String s = opt.orElseThrow();                  // Java 10: no-arg, clearer than get()
if (opt.isEmpty()) { ... }                     // Java 11
List<String> present = optionals.stream().flatMap(Optional::stream).toList(); // drop empties
```

**Creation:** `Optional.of(x)` throws NPE on `null`; `ofNullable(x)` accepts it; `empty()` for none.

**Primitive optionals:** `OptionalInt`, `OptionalLong`, `OptionalDouble` (from `IntStream.max()`, `average()`) avoid boxing; read with `getAsInt()` / `orElse(0)`.

**Anti-patterns:**
- `Optional` as field, parameter, or inside collections/map values (`Map<K, Optional<V>>`) — use `null`-free design/empty collections.
- `opt.isPresent() ? opt.get() : null` — equals `opt.orElse(null)`, and returning `null` from an `Optional` method defeats the purpose.
- Wrapping a value in `Optional` only to unwrap it a line later.
- Comparing/locking on `Optional` — it's a value-based class; identity operations (`==`, `synchronized`) are discouraged.
- Returning `null` from a method that returns `Optional` — worst of both worlds.

**Pitfalls / follow-up probes:** `Optional` is not `Serializable`; JPA/DTO fields shouldn't use it. For "might be absent" collections return an empty collection, not `Optional<List<T>>`.

**Remember:** *Lazy defaults with `orElseGet`; `flatMap` to avoid nesting; `Optional` is for return types only.*

---

### Q91. The `java.time` API — which class do I use for what?

**Say this first:** "Pick the type by whether you need a zone: `LocalDate`/`LocalDateTime` have none, `Instant` is a UTC timestamp, `ZonedDateTime` is both."

**Answer:** `java.time` (Java 8, by the Joda-Time authors) replaced `Date`/`Calendar`. All types are **immutable** and **thread-safe**; every "change" returns a new object.

| Type | Holds | Use for |
|---|---|---|
| `LocalDate` | date only (2024-05-17) | birthdays, due dates |
| `LocalTime` | time only | opening hours |
| `LocalDateTime` | date + time, **no zone** | local schedule, *not* a point in time |
| `ZonedDateTime` | date-time + `ZoneId` (DST rules) | meetings in a city |
| `OffsetDateTime` | date-time + fixed offset (+05:30) | APIs / DB with offsets |
| `Instant` | point on the UTC timeline (nanos) | timestamps, audit, "now" |
| `Duration` | time-based amount (hours, seconds) | elapsed time, timeouts |
| `Period` | date-based amount (years, months, days) | age, "3 months later" |

```java
LocalDate d = LocalDate.of(2024, 1, 31).plusMonths(1);        // 2024-02-29 (clamped to month end)
long days = ChronoUnit.DAYS.between(start, end);
Period age = Period.between(birth, LocalDate.now());          // years/months/days
Duration took = Duration.between(t1, t2);                     // Instant or LocalDateTime
Instant now = Instant.now();
ZonedDateTime ist = now.atZone(ZoneId.of("Asia/Kolkata"));
Instant back = ist.toInstant();
Date legacy = Date.from(now);  Instant fromLegacy = legacy.toInstant(); // bridge to old APIs
```

**Duration vs Period:** `Period` is calendar-based (a month has varying days); `Duration` is exact seconds. Adding `Period.ofDays(1)` to a `ZonedDateTime` keeps the **wall-clock time** across DST; `Duration.ofHours(24)` adds exactly 24 hours (wall clock may shift).

**Testability tip:** inject a `java.time.Clock` and call `Instant.now(clock)` / `LocalDate.now(clock)`; in tests use `Clock.fixed(...)` instead of mocking statics.

**Pitfalls / follow-up probes:**
- `LocalDateTime` is **not** an instant — never store it as "when it happened" across systems.
- JDBC 4.2+ supports `setObject(i, localDate)` directly — no need for `java.sql.Date`.
- Month arithmetic clamps (Jan 31 + 1 month = Feb 28/29), it doesn't overflow.

**Remember:** *Instant = machine time; LocalX = human wall-clock without zone; ZonedDateTime = both. Period = calendar, Duration = seconds.*

---

### Q92. `DateTimeFormatter` vs `SimpleDateFormat` and time-zone pitfalls — what goes wrong in production?

**Say this first:** "`SimpleDateFormat` is mutable and not thread-safe; `DateTimeFormatter` is immutable and safe to share — and always store timestamps in UTC."

**Answer (thread-safety):** `SimpleDateFormat` keeps an internal `Calendar` that it mutates while parsing/formatting. A shared static instance used by two threads produces **garbled dates or random exceptions** — a classic production bug. Old fixes: `ThreadLocal<SimpleDateFormat>` or synchronization. `DateTimeFormatter` is immutable, so a `static final` constant is fine.

```java
private static final DateTimeFormatter F = DateTimeFormatter.ofPattern("dd-MM-uuuu HH:mm");
String s = LocalDateTime.now().format(F);
LocalDate d = LocalDate.parse("2024-05-17");                    // ISO format needs no formatter
ZonedDateTime z = ZonedDateTime.parse("2024-05-17T10:00:00+05:30[Asia/Kolkata]");
```
Built-in constants: `DateTimeFormatter.ISO_LOCAL_DATE`, `ISO_INSTANT`, `ISO_OFFSET_DATE_TIME`.

**Pattern letter traps:**
- `mm` = minutes, `MM` = month. `HH` = 0–23, `hh` = 1–12 (needs `a` for AM/PM).
- `yyyy` = year-of-era, `uuuu` = proper year; **`YYYY` = week-based year** → `2024-12-30` can print as `2025`. Use `yyyy`/`uuuu`.

**Time-zone pitfalls:**
1. **Store and transmit `Instant`/UTC**; convert to a zone only for display. Avoid storing `LocalDateTime` for events.
2. **Server default zone** (`ZoneId.systemDefault()`) differs across machines/containers — be explicit.
3. **Daylight Saving Time (DST):** a local time may **not exist** (spring-forward gap; `ZonedDateTime` shifts it forward) or occur **twice** (fall-back overlap; it picks the earlier offset by default). Example (US, 2024-03-10): `zdt.plusDays(1)` keeps 10:00 wall clock but only 23 hours elapse; `plus(Duration.ofHours(24))` lands at 11:00.
4. Use region IDs (`Asia/Kolkata`) not abbreviations (`IST` is ambiguous) or fixed offsets, because offsets change with DST rules.
5. "Start of day" is `date.atStartOfDay(zone)` — not always `00:00` (some zones skip midnight on DST days).

**Pitfalls / follow-up probes:** `DateTimeParseException` on invalid input — validate/catch; `ResolverStyle.STRICT` rejects `2023-02-30`; JSON/DB layers should serialise `Instant` as ISO-8601 or epoch.

**Remember:** *Immutable formatter = thread-safe; store UTC `Instant`; `yyyy` not `YYYY`, `MM` not `mm`.*

---

### Q93. `var` (Java 10) — what are the rules, and when is it a bad idea?

**Say this first:** "`var` is compile-time type inference for local variables — the type is still static and fixed; it's not dynamic typing."

**Answer:** With `var`, the compiler works out the variable's type from the initializer on the right. It exists to cut repetition (`Map<String, List<Order>> m = new HashMap<String, List<Order>>()`), not to hide types.

```java
var orders = new HashMap<String, List<Order>>();   // inferred HashMap<String, List<Order>>
for (var e : orders.entrySet()) { ... }
try (var in = Files.newInputStream(p)) { ... }
```

**Where it is allowed:** local variables with an initializer, `for`/enhanced-`for` variables, try-with-resources, and (Java 11) lambda parameters `(var x, var y) -> ...` — useful only to add annotations (`(@NonNull var x) -> ...`).

**Where it is NOT allowed:**
- Fields, method parameters, return types.
- No initializer: `var x;`
- Initialised to `null`: `var x = null;` (no type to infer).
- Array initializer shorthand: `var a = {1, 2};` (but `var a = new int[]{1,2}` is OK).
- A lambda or method reference alone: `var f = () -> 1;` (no target type).
- A variable referencing itself in its initializer.

**Gotchas:**
- Diamond with `var`: `var list = new ArrayList<>();` infers `ArrayList<Object>`. Write `new ArrayList<String>()` or keep the explicit type.
- `var x = 10;` is `int`; `var y = 10L;` is `long`; `var n = cond ? 1 : 2.0;` becomes `double`. Be aware of the inferred type.
- `var` is **not a keyword**, just a reserved type name — `int var = 1;` still compiles, but a class named `var` does not.
- Programming to an interface: `var list = new ArrayList<String>();` makes the variable type `ArrayList`, not `List` — you lose the "program to interface" habit.

**Style guidance:** use `var` when the type is obvious from the right-hand side (`new`, a factory, a well-named method); avoid it when the type is non-obvious (`var result = service.process(x);`). Short methods + good names make `var` safe.

**Pitfalls / follow-up probes:** `var` with anonymous classes keeps the anonymous type (you can access its extra members) — a niche trick.

**Remember:** *`var` = local, initialised, inferred at compile time; not for fields/params/returns or `null`.*

---

### Q94. Switch expressions (Java 14): arrow labels, `yield`, exhaustiveness — how are they better than the old `switch`?

**Say this first:** "A switch *expression* returns a value, has no fall-through, and the compiler checks it covers every case."

**Answer:** The old `switch` statement falls through to the next `case` unless you `break` — a source of bugs — and you need a temporary variable to get a result. Java 14 added **switch expressions** (final in 14):

```java
int days = switch (month) {
    case JAN, MAR, MAY, JUL, AUG, OCT, DEC -> 31;      // multiple labels, arrow = no fall-through
    case FEB -> isLeap ? 29 : 28;
    case APR, JUN, SEP, NOV -> 30;
};                                                      // enum: no default needed if all constants covered

String label = switch (code) {
    case 200 -> "OK";
    case 404 -> "Not found";
    default -> {                                        // block: use yield to produce the value
        log.warn("code {}", code);
        yield "Other";
    }
};
```

**Key points:**
- **Arrow `->` form:** no fall-through, no `break`. Right side is an expression, a block, or a `throw`.
- **`yield`:** returns the value from a block (the colon form `case X: yield 1;` also works).
- **Exhaustive:** an expression must handle all possible values — via `default`, or by covering all enum constants / all permitted subtypes of a sealed type. Otherwise: **compile error**.
- Works with `int`, `char`, `String`, enums (and, from Java 21, patterns).
- `switch` on a `null` selector throws `NullPointerException` (unless `case null` in Java 21).

**Old vs new:**
| | Old statement | New expression |
|---|---|---|
| Fall-through | yes (needs `break`) | no (arrow form) |
| Returns value | no | yes |
| Exhaustive check | no | yes |

**Pitfalls / follow-up probes:**
- Don't mix `->` and `:` labels in one switch.
- Enum exhaustive switch without `default`: if someone later adds an enum constant in a separately compiled jar, you get a **runtime** error for the unhandled value. Adding `default -> throw ...` trades compile-time safety for runtime safety — prefer sealed types where you control all cases.
- `break`/`continue`/`return` cannot jump *out of* a switch expression.

**Remember:** *Arrow, no fall-through, `yield` for blocks, must be exhaustive.*

---

### Q95. Text blocks and the new `String` methods (`isBlank`, `strip`, `lines`, `repeat`, `formatted`, `transform`)

**Say this first:** "Text blocks make multi-line strings readable; the new `String` methods replace common utility-class helpers."

**Answer (text blocks, final in Java 15):** A text block is delimited by `"""`. The content starts on the **next line** after the opening delimiter. Great for JSON, SQL, HTML in code and tests.

```java
String json = """
        {
          "name": "%s",
          "age": %d
        }
        """.formatted(name, age);
```
**How indentation works:** the compiler removes *incidental* whitespace — the smallest common indent across all content lines **and the closing `"""`**. Moving the closing delimiter left keeps more indentation. Trailing spaces are stripped. Special escapes: `\` + newline joins lines (no newline), `\s` keeps a trailing space, `\"""` for three quotes. Normal escapes (`\n`, `\t`) still work. Text blocks are ordinary `String`s — interned in the pool like literals.

**New `String` methods:**
| Method | Since | What |
|---|---|---|
| `isBlank()` | 11 | true if empty **or only whitespace** (`isEmpty()` is only length 0) |
| `strip()`, `stripLeading()`, `stripTrailing()` | 11 | Unicode-aware trim (`trim()` removes only chars ≤ U+0020) |
| `lines()` | 11 | `Stream<String>` split on `\n`, `\r`, `\r\n` |
| `repeat(n)` | 11 | `"ab".repeat(3)` → `ababab` |
| `indent(n)` | 12 | add/remove leading spaces per line |
| `transform(fn)` | 12 | `s.transform(String::strip).transform(Integer::parseInt)` — chain functions |
| `formatted(args)` | 15 | instance version of `String.format` |
| `stripIndent()`, `translateEscapes()` | 15 | helpers behind text blocks |

```java
boolean empty = "  \t ".isBlank();                  // true
long nonBlank = text.lines().filter(Predicate.not(String::isBlank)).count();
String bar = "=".repeat(20);
```

**Pitfalls / follow-up probes:**
- `"%d".formatted(...)` uses the default `Locale` — be explicit (`String.format(Locale.ROOT, ...)`) for machine-readable output.
- Building large strings in loops still needs `StringBuilder`; `repeat` and text blocks don't replace it.
- Never concatenate user input into a SQL text block — still use prepared statements.

**Remember:** *Closing `"""` position sets the indent; `isBlank` ≠ `isEmpty`; `strip` ≠ `trim`.*

---

### Q96. Records in depth: compact constructors, static factories, shallow immutability and serialization

**Say this first:** "A record is a transparent, final data carrier — the compiler writes the constructor, accessors, `equals`, `hashCode` and `toString` from the component list."

**Answer:** `record Point(int x, int y) {}` generates: a `private final` field per component, a **canonical constructor**, accessors named `x()` / `y()` (not `getX`), and value-based `equals`/`hashCode`/`toString`. The class is implicitly `final`, extends `java.lang.Record`, cannot extend another class, but **can implement interfaces**.

**Compact constructor — validation and normalisation without repeating assignments:**
```java
record Range(int lo, int hi) {
    Range {                                    // compact form: no parameter list
        if (lo > hi) throw new IllegalArgumentException("lo > hi");
        // you may reassign parameters (lo = Math.max(lo, 0)); fields are assigned automatically at the end
    }
}
```
**Additional constructors** must delegate with `this(...)`. **Static factories** and static fields/methods are fine; **instance fields** beyond the components are not allowed.
```java
record Email(String value) {
    Email { value = value.strip().toLowerCase(); }
    static Email of(String raw) { return new Email(raw); }
}
```

**Shallow immutability — the trap:** components are `final`, but a mutable component (`List`, array, `Date`) can still be mutated from outside. Defend in the constructor and accessor:
```java
record Team(String name, List<String> members) {
    Team { members = List.copyOf(members); }   // defensive copy, also null-hostile
}
```
Array components use **reference** `equals`; a record holding `int[]` is a poor value type.

**Serialization:** records are serialized differently from normal classes — only the **components** are written, and on deserialization the **canonical constructor** is called, so your validation always runs (no way to bypass it). `writeObject`/`readObject`/`serialVersionUID` customisation is ignored for records. This makes records safer than classes for serialization.

**Other facts:** records can be generic (`record Pair<A,B>(A a, B b)`), nested/local (implicitly static, great as temporary tuples inside a method or stream), and can have annotations on components.

**Pitfalls / follow-up probes:**
- Records are not a fit for JPA entities (need no-arg constructor, mutable state) — use them for DTOs and value objects.
- You can override an accessor, but it must keep the same name, type and be `public`.
- `toString` includes all components — don't put secrets in a record you log.

**Remember:** *Records = final value carriers; compact constructor validates; copy mutable components yourself.*

---

### Q97. Sealed types, `instanceof` patterns, pattern matching for `switch` and record patterns — with exhaustiveness

**Say this first:** "Sealed types close the set of subtypes, so a `switch` over them can be checked by the compiler — no `default` needed, and adding a subtype breaks the build where you forgot to handle it."

**Answer — the four pieces:**

**1. `instanceof` pattern (final in Java 16)** — test and cast in one step; the variable is in scope only where the test is true (*flow scoping*):
```java
if (obj instanceof String s && s.length() > 3) { use(s); }
if (!(obj instanceof String s)) return;      // s is in scope after this line
```
**2. Sealed classes (final in 17):** `sealed` + `permits` lists the only allowed direct subtypes. Each must be `final`, `sealed`, or `non-sealed`, and live in the same module (or package if unnamed module). `permits` can be omitted if all subtypes are in the same file.

**3. Pattern matching for `switch` (final in 21)** with type patterns, guards (`when`) and `case null`.

**4. Record patterns (final in 21)** — deconstruct a record in the pattern, including nested ones.

```java
sealed interface Shape permits Circle, Rect, Square {}
record Circle(double r) implements Shape {}
record Rect(double w, double h) implements Shape {}
record Square(double side) implements Shape {}

static double area(Shape s) {
    return switch (s) {
        case Circle c                      -> Math.PI * c.r() * c.r();
        case Rect(double w, double h)      -> w * h;                 // record pattern
        case Square sq when sq.side() < 0  -> throw new IllegalArgumentException();
        case Square sq                     -> sq.side() * sq.side();
    };   // exhaustive: no default
}
```

**Exhaustiveness and why it matters:** the compiler knows all permitted subtypes, so it errors if you miss one. Add `record Triangle(...) implements Shape` → every such `switch` fails to compile until updated. That is the **safe alternative to the Visitor pattern**. Adding `default` silences this check — avoid it for sealed hierarchies.

**Rules:** cases are checked top to bottom; a case that is *dominated* by an earlier broader one is a compile error (put specific/guarded cases first). `case null` handles `null` (without it, a `null` selector throws `NullPointerException`); `case null, default ->` combines them.

**Pitfalls / follow-up probes:**
- Sealed ≠ `final`: sealed gives a **closed** but still polymorphic hierarchy.
- If a subtype is added later in a separately compiled module, an old exhaustive `switch` throws `MatchException` at runtime.
- Pattern variables are not implicitly `final`, but reassigning them hurts readability — treat them as read-only.
- Unnamed patterns/variables (`_`) came later (see the version cheat sheet).

**Remember:** *Sealed = closed set → exhaustive switch → no `default`; patterns replace cast chains.*

---

### Q98. Helpful NullPointerExceptions (Java 14) — what changed and how do you read them?

**Say this first:** "Since Java 15 the JVM tells you *which* expression was null, so most NPEs can be diagnosed from the message alone."

**Answer:** Before, an NPE said only `java.lang.NullPointerException` plus a line number — on `a.getB().getC().name()` you didn't know which call returned `null`. JEP 358 added a detailed message computed lazily from the bytecode when the exception message is requested. It was introduced in Java 14 as opt-in (`-XX:+ShowCodeDetailsInExceptionMessages`) and became **enabled by default in Java 15**.

```java
String city = user.getAddress().getCity().toUpperCase();
// Cannot invoke "Address.getCity()" because the return value of "User.getAddress()" is null
```
Another example: `Cannot load from int array because "arr" is null`.

**How to read it:** the first part says the **failed action** (`invoke`, `read field`, `load from array`, `store`, `throw`, `synchronize`); the `because` part says **what was null** (a method's return value, a field, a local, an array element).

**Local variable names:** you see real names only if the class was compiled with `-g` (debug info); otherwise you get `"<local4>"`. Most build tools include `-g` by default for Maven/Gradle debug builds, but check release builds.

**Security note:** the message contains **names of code elements, not data values**, so it doesn't leak the contents of variables.

**Still your job:**
- Fail fast with `Objects.requireNonNull(x, "x must not be null")` at API boundaries so the message names the **business** problem.
- Use `Optional` for return values and empty collections instead of `null`.
- Log the exception itself (`log.error("...", e)`), not just a wrapper's `getMessage()`, or the helpful text is lost.

**Pitfalls / follow-up probes:** wrapping an NPE in a new exception without passing it as the **cause** hides the detail. After many repeats the JIT may throw a pre-allocated NPE **without stack trace or message** (`OmitStackTraceInFastThrow`); use `-XX:-OmitStackTraceInFastThrow` when diagnosing.

**Remember:** *Read "Cannot X because Y is null"; compile with `-g` for names; validate early with `requireNonNull`.*

---

### Q99. The built-in `java.net.http.HttpClient` (Java 11) — how do you use it and what does it not do?

**Say this first:** "One shared, immutable client; build a request; `send` blocks, `sendAsync` returns a `CompletableFuture`."

**Answer:** The old `HttpURLConnection` was clumsy; Apache/OkHttp were the usual replacements. Java 11 standardised a modern client (incubated in 9/10) in `java.net.http`: HTTP/1.1 and **HTTP/2** (negotiated automatically), sync and async calls, WebSocket, and builder-style immutable objects.

```java
HttpClient client = HttpClient.newBuilder()
        .version(HttpClient.Version.HTTP_2)
        .connectTimeout(Duration.ofSeconds(3))
        .followRedirects(HttpClient.Redirect.NORMAL)
        .build();                                   // create ONCE, reuse (thread-safe)

HttpRequest req = HttpRequest.newBuilder(URI.create("https://api.example.com/users/1"))
        .timeout(Duration.ofSeconds(5))             // per-request response timeout
        .header("Accept", "application/json")
        .GET().build();

HttpResponse<String> res = client.send(req, HttpResponse.BodyHandlers.ofString());
if (res.statusCode() / 100 != 2) { /* handle error */ }

client.sendAsync(req, HttpResponse.BodyHandlers.ofString())
      .thenApply(HttpResponse::body)
      .thenAccept(System.out::println);

HttpRequest post = HttpRequest.newBuilder(URI.create(url))
        .header("Content-Type", "application/json")
        .POST(HttpRequest.BodyPublishers.ofString(json)).build();
```
`BodyHandlers` choose how the body is consumed: `ofString()`, `ofByteArray()`, `ofFile(path)`, `ofInputStream()`, `ofLines()`, `discarding()`.

**What it does NOT do:**
- It does **not** throw on 4xx/5xx — check `statusCode()` yourself.
- No JSON mapping, no retries, no circuit breaker, no connection-pool tuning like dedicated libraries.
- Default redirect policy is `NEVER` (you must opt in).

**Pitfalls / follow-up probes:**
- Creating a new `HttpClient` per request wastes connections and threads — keep one.
- `send` throws `IOException` and `InterruptedException` — restore the interrupt flag if you catch the latter.
- With virtual threads (Java 21), the simple blocking `send` is a good fit; the client's internal executor can be customised via `.executor(...)`.
- In JDK 21, `HttpClient` also became `AutoCloseable` (`close()`, `shutdown()`) — check JDK docs for the exact lifecycle in your version.

**Remember:** *One client, immutable requests, check the status code yourself; `sendAsync` = `CompletableFuture`.*

---

### Q100. Immutable collection factories (`List.of`, `Set.of`, `Map.of`, `copyOf`) vs `Collections.unmodifiableList` vs `Arrays.asList` — and sequenced collections

**Say this first:** "`List.of` is truly immutable and null-hostile; `unmodifiableList` is just a read-only *view* over a list that can still change underneath."

**Answer:** Java 9 added compact factories for small, fixed data (constants, test data, config).

```java
List<String> l = List.of("a", "b");
Set<Integer> s = Set.of(1, 2, 3);
Map<String, Integer> m = Map.of("a", 1, "b", 2);                  // up to 10 pairs
Map<String, Integer> big = Map.ofEntries(Map.entry("a", 1), Map.entry("b", 2));
List<String> snapshot = List.copyOf(mutableList);                  // Java 10, immutable copy
```

**Comparison:**
| | Mutable? | Nulls | Reflects changes of source? |
|---|---|---|---|
| `List.of`, `Set.of`, `Map.of` | No (throws `UnsupportedOperationException`) | **Rejected** (NPE) | n/a |
| `List.copyOf(c)` | No | Rejected | No (snapshot; returns same instance if already immutable) |
| `Collections.unmodifiableList(l)` | Not via the view | Allowed | **Yes** — a view |
| `Arrays.asList(arr)` | `set` allowed, `add/remove` not (fixed size) | Allowed | Write-through to the array |
| `stream.toList()` (16) | No | Allowed | No |

**Behaviours interviewers like:**
- `Set.of` / `Map.of` throw `IllegalArgumentException` on **duplicate** elements/keys.
- Their **iteration order is unspecified and randomised per JVM run** — never depend on it (use `LinkedHashSet`/`LinkedHashMap` if order matters).
- `List.of(...).contains(null)` throws NPE.
- Immutable ≠ deeply immutable: elements themselves may be mutable.

**Sequenced collections (Java 21):** new interfaces `SequencedCollection`, `SequencedSet`, `SequencedMap` give a uniform **encounter-order API**: `getFirst()`, `getLast()`, `addFirst()`, `addLast()`, `removeFirst()`, `removeLast()`, and `reversed()` (a view). Implemented by `List`, `Deque`, `LinkedHashSet`, `LinkedHashMap`, `TreeSet`/`TreeMap` (sorted ones are sequenced but don't allow `addFirst/addLast`). Replaces hacks like `list.get(list.size() - 1)`.

**Pitfalls / follow-up probes:** returning `Collections.unmodifiableList(this.items)` from a getter leaks later changes — prefer `List.copyOf(items)` (a snapshot) when you want a stable value. `Arrays.asList(int[])` gives `List<int[]>`.

**Remember:** *`of`/`copyOf` = immutable, no nulls; `unmodifiable` = view; `asList` = fixed-size window on the array.*

---

### Q101. Java modules (JPMS, Java 9) — what problem do they solve and how do you read a `module-info.java`?

**Say this first:** "Modules add *strong encapsulation* and *explicit dependencies* above packages — `public` no longer means 'visible to everyone'."

**Answer:** Before Java 9 everything lived on the **classpath**: any `public` class was accessible, internal JDK classes (`sun.misc.Unsafe`) were abused, and missing/duplicate jars failed at runtime ("jar hell"). JPMS (the Java Platform Module System) introduces the **module**: a named group of packages plus a descriptor saying what it needs and what it exposes. The JDK itself is modularised (`java.base`, `java.sql`, `java.net.http`, ...), which enables small custom runtimes via `jlink`.

```java
module com.shop.orders {
    requires java.sql;                       // dependency, checked at compile + startup
    requires transitive com.shop.common;     // my API exposes its types, so my users get it too
    exports com.shop.orders.api;             // only this package is visible outside
    opens com.shop.orders.model to com.fasterxml.jackson.databind; // allow deep reflection
    uses com.shop.spi.PaymentProvider;       // service loader consumer
    provides com.shop.spi.PaymentProvider with com.shop.orders.StripeProvider;
}
```

**Key words:**
| Directive | Meaning |
|---|---|
| `requires` | I depend on this module |
| `requires transitive` | and anyone who requires me also gets it |
| `exports` | package is public API (compile-time and runtime access) |
| `opens` | allows **reflection** on private members (frameworks, serialization) |
| `uses` / `provides ... with` | `ServiceLoader` plug-ins |

**Module path vs classpath:** jars on the **module path** are real modules (or *automatic modules* if they have no descriptor — name derived from the jar). Everything on the classpath goes into the **unnamed module**, which can read all modules — this is how most apps still run.

**Real-world impact (why you hit it):**
- Java 16/17 strongly encapsulate JDK internals: reflective access to `java.*` internals fails with `InaccessibleObjectException`. Quick fix: `--add-opens java.base/java.lang=ALL-UNNAMED` (a stopgap, not a design).
- Most business apps and frameworks **don't** use `module-info`; libraries use it or provide an `Automatic-Module-Name`.
- Java EE modules (JAXB, JAX-WS) were removed in Java 11 — add them as normal dependencies.

**Pitfalls / follow-up probes:** split packages (same package in two modules) are illegal; reflection-heavy frameworks need `opens`; `jdeps` shows module dependencies and `jlink` builds a trimmed runtime image.

**Remember:** *`exports` = visible API, `opens` = reflection allowed, `requires` = dependency; most apps stay on the classpath.*

---

### Q102. Java version cheat sheet: LTS releases (8, 11, 17, 21) and what is notable in 22 and later

**Say this first:** "Know the LTS stepping stones and one headline per version; for anything after 21, say whether it's final or preview — and if unsure, check the JDK docs."

**Answer:** Oracle ships a feature release every **6 months**; **LTS** (Long-Term Support) releases are the ones vendors support for years. Production usually moves LTS → LTS.

| LTS | Headline features | Removed / changed |
|---|---|---|
| **8** (2014) | Lambdas, Streams, `Optional`, default methods, `java.time`, `CompletableFuture` | PermGen → Metaspace |
| **11** (2018) | `HttpClient`, `var` in lambdas, `String` methods, `Files.readString`, single-file source launch, Flight Recorder open-sourced | Java EE & CORBA modules removed; Nashorn deprecated |
| **17** (2021) | **Records**, **sealed classes**, text blocks, switch expressions, `instanceof` patterns, helpful NPE, new `RandomGenerator` | Strong encapsulation of JDK internals; Security Manager deprecated for removal |
| **21** (2023) | **Virtual threads**, pattern matching for `switch`, record patterns, **sequenced collections**, generational ZGC | — |

**In between (non-LTS) that you should know:** 9 modules/`List.of`; 10 `var`; 12 `Collectors.teeing`; 14 switch expressions; 15 text blocks; 16 records and `Stream.toList()`; 18 UTF-8 as default charset (JEP 400).

**After 21 (be careful — status changes between releases):**
| Feature | Status I am confident about |
|---|---|
| Unnamed variables & patterns (`_`) | Final in **22**: `catch (Exception _)`, `case Circle(_) ->` for unused names |
| Foreign Function & Memory API | Final in **22** (replaces JNI for many uses) |
| Stream Gatherers (`Stream.gather(...)`, custom intermediate ops) | Preview in 22/23, **final in 24** |
| Synchronized pinning of virtual threads | Addressed in **24** |
| Scoped Values | Preview from 21; **final in 25** (check docs for your JDK) |
| Structured Concurrency | Still **preview** as of 25 — verify in current docs |
| String Templates | Preview in 21/22, **removed in 23** — don't rely on it |
| Flexible constructor bodies (statements before `super(...)`/`this(...)`) | Preview earlier; **final in 25** |
| **Java 25** | The latest LTS at the time of writing (Sept 2025); also includes compact source files / instance main methods and module import declarations as final features |

**How to answer "which version are you on and why?":** state your runtime (e.g. 17 or 21), why (LTS, framework baseline, vendor support), and what you use from it (records, virtual threads, pattern switch). Mention the upgrade path: build on the new JDK, fix removed APIs/illegal reflective access, run with `--release`, update agents/libraries.

**Pitfalls / follow-up probes:**
- **Preview features** need `--enable-preview` and can change or vanish; never ship them in a library without saying so.
- Don't quote "final" from memory for post-21 items in an interview — say "as far as I know; I'd verify in the JDK release notes."
- Vendors (Temurin, Corretto, Zulu, Oracle) all ship OpenJDK builds; LTS support length varies by vendor.

**Remember:** *LTS = 8, 11, 17, 21, 25; records/sealed = 17, patterns/virtual threads = 21; verify anything newer.*

---

# 4. Exception Handling

### Q103. Checked vs unchecked exceptions — and what do you use in a service?

- **Checked** (`extends Exception`) — compiler-enforced, for recoverable conditions (`IOException`, `SQLException`).
- **Unchecked** (`extends RuntimeException`) — programming errors and unrecoverable domain failures.
- **Error** (`OutOfMemoryError`, `StackOverflowError`) — never catch (except to log and die).

**My stance at 9 YOE:** In a Spring Boot service I use **unchecked custom exceptions** almost exclusively, because:
1. Checked exceptions don't compose with lambdas/streams.
2. `@Transactional` rolls back on `RuntimeException` by default (checked ones require `rollbackFor`).
3. They don't pollute every intermediate method signature.

I define a small hierarchy and map it once in a `@RestControllerAdvice`:
```java
public abstract class AppException extends RuntimeException {
    private final String code; private final HttpStatus status;
    ...
}
public class NotFoundException extends AppException { ... }   // 404
public class ValidationException extends AppException { ... }  // 400
public class DownstreamException extends AppException { ... }  // 502
```

**Remember:** *Checked for the caller's recovery, unchecked for bugs and domain failures; Spring services prefer unchecked.*

---

### Q104. try-with-resources, suppressed exceptions, and multi-catch.

```java
try (var conn = ds.getConnection();
     var ps = conn.prepareStatement(SQL)) {          // closed in REVERSE order
    ...
} catch (SQLException | IOException e) {             // multi-catch: e is effectively final
    throw new DataAccessException("query failed", e); // ALWAYS chain the cause
}
```
- Requires `AutoCloseable`.
- If the body throws **and** `close()` throws, the close exception is **suppressed** and attached: `e.getSuppressed()`. With plain finally you'd lose the original.
- Java 9+: you can use an existing effectively-final variable: `try (existingResource) { }`.

**Remember:** *try-with-resources closes in reverse and suppresses close-time failures instead of hiding the real one.*

---

### Q105. Exception handling anti-patterns you review for in PRs.

1. `catch (Exception e) {}` — swallowing. Silent failures.
2. `e.printStackTrace()` — bypasses the logging framework; use `log.error("context", e)`.
3. Losing the cause: `throw new MyException(e.getMessage())` — stack trace gone.
4. Using exceptions for flow control (they're expensive — stack trace capture).
5. Catching `Throwable`/`Error`.
6. `return` inside `finally` — discards the exception.
7. Catching then rethrowing the same exception with no added value.
8. Logging **and** rethrowing → duplicate log entries (`log-and-throw` anti-pattern). Do one.
9. Leaking internal messages/stack traces to API consumers (security issue) — sanitize in the advice.

**Why each one is bad (and what to do instead):**
```java
// ❌ swallows + loses cause + prints to stderr
try { repo.save(o); } catch (Exception e) { e.printStackTrace(); throw new MyException(e.getMessage()); }

// ✅ wrap with cause, don't log here — let the @RestControllerAdvice log once
try { repo.save(o); } catch (DataAccessException e) { throw new OrderSaveException("save failed for " + o.getId(), e); }
```
- An empty catch hides production failures — the system keeps going in a corrupt state and nobody knows why.
- Passing only `e.getMessage()` discards the stack trace and the original type, so debugging at 2 a.m. becomes guesswork. Always pass the exception as the **cause** argument.
- Log-and-throw means one failure produces 3–5 log lines across layers, making dashboards and alerts misleading. Rule: *handle it* **or** *propagate it*, and log at the boundary (controller advice, message listener, scheduled job).
- Catching `Throwable`/`Error` can swallow `OutOfMemoryError`/`StackOverflowError`, leaving the JVM in an undefined state.

**Perf note:** `fillInStackTrace()` is the costly part. For high-frequency expected control-flow exceptions, override it or use a cached singleton exception with `super(msg, cause, false, false)` (no suppression, no stack trace).

**Remember:** *Never swallow, never lose the cause, log once at the boundary.*

---

### Q106. Explain the `Throwable` → `Error` / `Exception` hierarchy. What should you never catch?

**Say this first:** "Everything throwable extends `Throwable`. Below it there are two branches: `Error` (the JVM is in trouble, don't handle it) and `Exception` (the app can react). Under `Exception`, `RuntimeException` is unchecked and everything else is checked."

**Answer:**
```
Throwable
├── Error                         (unchecked – JVM/system level problems)
│    ├── OutOfMemoryError, StackOverflowError
│    ├── LinkageError → NoClassDefFoundError, ExceptionInInitializerError
│    └── AssertionError, InternalError, VirtualMachineError ...
└── Exception
     ├── IOException, SQLException, TimeoutException ...   (CHECKED)
     └── RuntimeException                                   (UNCHECKED)
          ├── NullPointerException, IllegalArgumentException
          └── IllegalStateException, ClassCastException ...
```
- **Checked** = the compiler forces you to either `catch` or declare it with `throws`. The idea: the caller *can recover* (retry, fallback, ask the user).
- **Unchecked** = `RuntimeException` and `Error` subclasses. No forced handling. Mostly programming bugs or unrecoverable states.
- `Throwable` itself is the only class `throw` accepts; it carries message, cause, stack trace and suppressed exceptions.

**What to catch, from safest to riskiest:**

| Catch | Verdict |
|---|---|
| A specific exception (`catch (SQLException e)`) | Best — you know what went wrong and what to do |
| `catch (Exception e)` | Acceptable only at a **boundary** (top of a job, request, message listener) to log/translate. Note it also catches `RuntimeException`, which hides bugs inside business logic |
| `catch (Throwable t)` / `catch (Error e)` | Almost never. It also traps `OutOfMemoryError`, `StackOverflowError`, `ThreadDeath`-style failures after which the JVM state may be unreliable |
| `catch (InterruptedException e) {}` | Never swallow. Re-set the flag: `Thread.currentThread().interrupt();` |

**Why not catch `Error`?** After `OutOfMemoryError` there may be no memory even to log. After `LinkageError` a class is half-loaded. Best response: log if you can, then let the process die and let the orchestrator restart it (`-XX:+ExitOnOutOfMemoryError`).

**Legit exceptions to the rule:** a top-level thread/worker loop that must log a `Throwable` and then rethrow or exit; test frameworks catching `AssertionError`.

**Pitfalls / follow-up probes:**
- *Is `Error` always fatal?* No — `StackOverflowError` in one request thread is often survivable; `OutOfMemoryError` is a gamble. The point is you can't *fix* it by catching.
- *Can you throw `Throwable` from a method?* Yes, but you must declare it — a bad API (callers must catch everything).
- *Is `Exception` checked?* Yes, except the `RuntimeException` branch.
- *What's `ExceptionInInitializerError`?* A static initializer threw; the class is then unusable (later uses give `NoClassDefFoundError`).

**Remember:** *Throwable → Error (don't touch) / Exception (checked) → RuntimeException (bugs). Catch specific; broad catches only at boundaries.*

---

### Q107. Exception chaining and wrapping — how do you do it correctly?

**Say this first:** "Wrap to change the abstraction level, always pass the original as the *cause*, and don't log it at every layer."

**Answer:** *Chaining* means one exception stores another as its **cause**. The reader of the stack trace then sees "Caused by: ..." all the way to the root problem.

**Why wrap at all?** The service layer shouldn't leak `SQLException` to its callers — that ties them to JDBC. You translate it into a domain exception (`OrderSaveException`) that makes sense at that layer, but keep the original for debugging.

```java
try {
    repo.save(order);
} catch (SQLException e) {
    throw new OrderSaveException("Could not save order " + order.id(), e);   // cause kept
}
```
Custom exceptions must expose the cause constructor:
```java
public class OrderSaveException extends RuntimeException {
    public OrderSaveException(String msg, Throwable cause) { super(msg, cause); }
}
```
If the type has no such constructor, use `initCause(e)` once (it throws if the cause was already set).

**Rules of thumb:**
1. Add **context** (ids, operation) in the message — the stack trace says *where*, the message says *what*.
2. Wrap only if you add value (new abstraction or context). Otherwise let it propagate.
3. Don't wrap an exception your caller is expected to handle by type (e.g. keep `TimeoutException` visible if retry logic needs it).
4. Find the root cause with a loop (or Apache/Guava helpers):
```java
Throwable root = e;
while (root.getCause() != null && root.getCause() != root) root = root.getCause();
```
5. **Log once** — either handle (log and recover) or throw. Logging and rethrowing at every layer gives 3–5 duplicate stack traces for one failure and misleads alerts.

**Logging correctly:**
```java
log.error("Order {} failed", id, e);          // ✅ last arg = Throwable → full stack trace
log.error("Order failed: " + e);              // ❌ only toString(), no stack
log.error("Order failed: {}", e.getMessage()); // ❌ loses trace and type
```
- Log at the **boundary** (controller/listener/job runner), with correlation id (MDC).
- Don't log sensitive data in messages (PII, tokens).
- Expected business failures (validation) deserve `warn`/`info` without a big stack trace; unexpected ones are `error` with the trace.

**Pitfalls / follow-up probes:**
- *Difference between cause and suppressed?* **Cause** = "this failed *because of* that". **Suppressed** = secondary failures (usually from `close()`) that happened while the primary was already propagating.
- *`getMessage()` vs `toString()`?* `toString()` = class name + message — more informative.
- *Cause cycles?* `initCause` prevents self-causation; a loop A→B→A is possible only by hand — guard with a depth limit when walking.
- *Does Java print the full chain?* Yes, `printStackTrace` prints "Caused by:" with "... N more" to avoid repeating common frames.

**Remember:** *Wrap with cause, add context, log once at the boundary.*

---

### Q108. How do you design a custom exception? (checked vs unchecked, `serialVersionUID`, error codes)

**Say this first:** "Small hierarchy, one base class, unchecked by default, carries a stable error code, and never carries non-serializable state."

**Answer — design decisions, in order:**

1. **Checked or unchecked?** Use **checked** only if the *direct caller can realistically recover* and you want the compiler to remind them (e.g. a library's `InsufficientFundsException`). For application/service code, use **unchecked**: they don't clutter every signature, work with lambdas, and avoid the "catch and wrap" noise.
2. **Extend the right class** — `RuntimeException` (or `Exception` for checked). Never extend `Throwable` or `Error` directly.
3. **Always provide the four standard constructors** (message, cause, both, and optionally the 4‑arg protected one for stack-trace control).
4. **Carry a machine-readable code**, not just text, so clients and dashboards don't parse messages.
5. **Keep a hierarchy shallow**: one `AppException` base, a few subclasses by *meaning* (not by every error).

```java
public class AppException extends RuntimeException {
    private static final long serialVersionUID = 1L;

    private final String code;                       // e.g. "ORDER_NOT_FOUND"

    public AppException(String code, String message) { super(message); this.code = code; }
    public AppException(String code, String message, Throwable cause) {
        super(message, cause); this.code = code;
    }
    public String code() { return code; }
}

public class NotFoundException extends AppException {
    private static final long serialVersionUID = 1L;
    public NotFoundException(String entity, Object id) {
        super(entity.toUpperCase() + "_NOT_FOUND", entity + " not found: " + id);
    }
}
```

**What is `serialVersionUID`?** `Throwable` implements `Serializable` (exceptions can cross JVM boundaries — RMI, remote calls, some caches/queues). `serialVersionUID` is a version number: if the class changes shape incompatibly, a different ID makes deserialization fail clearly instead of silently misreading. If you omit it, the compiler derives one from the class structure, so a harmless change can break compatibility. IDEs/linters (`-Xlint:serial`) warn when it's missing. Declare it `private static final long`.

**Serialization gotchas:** fields in exceptions should be serializable (strings, ints, enums) — don't store entities, `HttpRequest`, or connections. Mark non-serializable fields `transient` or avoid them.

**Pitfalls / follow-up probes:**
- *Why not one exception per error?* Class explosion. Use one class + code, or a few categories.
- *Immutable fields?* Make them `final`; exceptions may be shared across threads when logged/rethrown.
- *Don't build the message from heavy `toString()`* of big objects — messages get logged.
- *Overriding `fillInStackTrace`* for hot expected exceptions — see the performance question.
- *Name conventions:* end with `Exception`; unchecked domain exceptions are usually named by condition (`OrderNotFoundException`).

**Remember:** *Unchecked by default, one base class, stable code, serialVersionUID, cause constructor.*

---

### Q109. `finally` edge cases — `return` in `finally`, exceptions in `finally`, `System.exit`, and try-finally vs try-with-resources.

**Say this first:** "`finally` runs almost always — and whatever it *returns or throws* wins over what `try` was doing."

**Answer:** `finally` is meant for cleanup that must run whether the `try` completed normally, returned, or threw.

**Edge case 1 — `return` in `finally` overrides everything (and swallows exceptions):**
```java
static int f() {
    try { throw new RuntimeException("boom"); }
    finally { return 42; }          // exception is silently DISCARDED; f() returns 42
}
```
**Edge case 2 — the return value is already computed before `finally` runs:**
```java
static int g() {
    int x = 1;
    try { return x; }               // value 1 is saved
    finally { x = 99; }             // changing the local does NOT change the result
}                                    // returns 1 (but if x were a mutable object, its state changes ARE visible)
```
**Edge case 3 — an exception thrown in `finally` replaces the original:** the first exception is lost (no suppression with plain try/finally).
```java
try { throw new IOException("original"); }
finally { throw new IllegalStateException("from finally"); }   // caller sees only this one
```
**Edge case 4 — `finally` does NOT run when:**
- `System.exit()` / `Runtime.halt()` is called inside `try` (`exit` runs shutdown hooks but not your `finally`);
- the JVM crashes or the process is killed (`kill -9`);
- the thread loops forever or is blocked forever in `try`;
- the thread is a **daemon** and the JVM exits while it is mid-`try`.

**try-finally vs try-with-resources (TWR):**
```java
// old style — verbose, and a close() failure hides the real exception
InputStream in = open();
try { use(in); } finally { in.close(); }

// TWR — closes automatically, in REVERSE order of declaration
try (var in = open(); var out = create()) { copy(in, out); }
```
**Order of execution in TWR:**
1. resources are initialised top to bottom;
2. body runs;
3. resources are **closed** bottom to top;
4. *then* the `catch` and `finally` blocks of the same statement run.

So inside `catch`/`finally` the resources are **already closed** — don't use them there. If the body and `close()` both throw, the body's exception is primary and the close failure is attached via `getSuppressed()`.

**Pitfalls / follow-up probes:**
- *If resource initialisation itself throws?* Resources already opened are still closed; the failing one isn't.
- *Can `finally` change a thrown exception?* Only by throwing/returning itself — which is why you don't.
- *Nested try-catch inside `finally`* for cleanup that can fail: catch and log there so you don't mask the original.
- *Why was `finalize()` different?* Unrelated to `finally`; it's deprecated GC-time cleanup.

**Remember:** *Never `return`/`throw` from `finally`; prefer TWR; `System.exit` skips `finally`.*

---

### Q110. Exception rules for overriding methods, `catch` ordering and multi-catch.

**Say this first:** "An override can't promise *more* checked exceptions than the parent. In `catch`, order goes narrow → wide, and multi-catch alternatives can't be related."

**Answer — overriding (why: *substitutability*):** code written against the parent type only handles what the parent declared. If a child could suddenly throw a new checked exception, the caller's code would break at runtime.

| Parent declares | Child override may declare |
|---|---|
| nothing | no checked exceptions (any unchecked is fine) |
| `IOException` | `IOException`, a subclass (`FileNotFoundException`), or nothing |
| `IOException` | ❌ `Exception` or `SQLException` (broader/unrelated) |
| `throws Exception` | anything narrower |

```java
class Reader { void read() throws IOException {} }
class FileReader2 extends Reader {
    @Override void read() throws FileNotFoundException {}   // ✅ narrower
    // @Override void read() throws Exception {}            // ❌ compile error
}
```
- **Constructors:** a subclass constructor must declare (or handle) the checked exceptions of the super constructor it calls.
- **Static/private methods** are hidden, not overridden — these rules don't apply.
- **Interfaces:** an implementing class can only narrow what the interface method declares.

**`catch` rules:**
1. Catch blocks are checked top to bottom; a **subclass after its superclass** is a compile error ("already been caught"):
```java
try { ... }
catch (FileNotFoundException e) { }     // narrow first
catch (IOException e) { }               // then wider
```
2. Catching a **checked** exception that the `try` body can't throw is a compile error (except `Exception`/`Throwable`).
3. **Multi-catch** (Java 7): `catch (IOException | SQLException e)`
   - alternatives must **not** be subclass-related (`IOException | FileNotFoundException` → compile error);
   - `e` is implicitly `final` — you can't reassign it;
   - its static type is the closest common supertype, so you can only call members they share.
4. **Precise rethrow** (Java 7): if you catch a broad type and rethrow an effectively-final parameter, the compiler knows the *actual* exceptions the body can throw:
```java
void m() throws IOException {            // not "throws Exception"
    try { doIo(); }
    catch (Exception e) { log.warn("x"); throw e; }   // compiler infers IOException only
}
```

**Pitfalls / follow-up probes:**
- *Can an override throw a new unchecked exception?* Yes, always.
- *Does `@Override` check exceptions?* The compiler checks them with or without the annotation; the annotation just verifies you're overriding.
- *Is order significant for unrelated exceptions?* No, only for related ones.

**Remember:** *Child can narrow, never widen, checked exceptions; catch narrow first; multi-catch types must be unrelated.*

---

### Q111. `StackOverflowError` vs `OutOfMemoryError` — causes and how you react.

**Say this first:** "SOE = one thread's call stack is full (too deep). OOM = a memory *region* is exhausted (heap, metaspace, native)."

| | `StackOverflowError` | `OutOfMemoryError` |
|---|---|---|
| Where | Thread's own **stack** (limited by `-Xss`, commonly 512 KB–1 MB) | **Heap** (`-Xmx`), Metaspace, direct memory, native threads |
| Typical cause | Infinite/very deep recursion | Leak, too much data loaded, too small heap/limits |
| Scope | Affects that thread/request | Can hit any thread; JVM may be unstable |
| Fix | Fix recursion; make iterative | Heap dump, fix leak, resize (see the OOM-types question in this guide) |

**How SOE happens (step by step):** every method call pushes a *frame* on the thread's stack (locals + operand stack). Frames are only popped when the method returns. Unbounded recursion keeps pushing until the fixed-size stack is full → JVM throws `StackOverflowError`.

**Common real-world SOE causes:**
```java
// 1. Missing base case
int f(int n) { return f(n - 1); }

// 2. Accidental self-call (getter/toString)
public String getName() { return getName(); }

// 3. Cyclic object graphs in toString/equals/hashCode/JSON
class Order    { List<Item> items; public String toString() { return "Order" + items; } }
class Item     { Order order;      public String toString() { return "Item" + order; } }  // Order↔Item loop
```
- Bidirectional JPA relations serialised to JSON, Lombok `@ToString`/`@EqualsAndHashCode` on both sides.
- Very deep recursive parsing (nested JSON/regex with backtracking) — attacker-controlled input is a DoS risk.
- Deep call chains from frameworks (proxies, aspects) plus your recursion.

**Handling SOE:** fix the cause first. Increasing `-Xss` (or creating `new Thread(null, r, "n", stackSize)`) only postpones it and multiplies memory by thread count. Convert deep recursion to a loop with an explicit `Deque`. Catching SOE is a last-resort guard (e.g. in a parser returning "input too deep") — keep the catch small, since the stack is nearly full when you land in the handler.

**How OOM differs:** the message tells you the region — `Java heap space`, `Metaspace`, `unable to create native thread`, `Direct buffer memory`. React with heap dump + GC log analysis, not with `try/catch`. Set `-XX:+HeapDumpOnOutOfMemoryError` and `-XX:+ExitOnOutOfMemoryError` so containers restart cleanly.

**Pitfalls / follow-up probes:**
- *Does a larger heap fix SOE?* No — stacks are separate from heap.
- *Can you catch OOM and continue?* Technically yes (e.g. a huge allocation attempt failing cleanly), but other threads may already be starved. Don't rely on it.
- *Tail-call optimisation?* Java doesn't do it, so tail recursion still grows the stack.

**Remember:** *SOE = too deep (stack); OOM = too much (heap/native). Fix the algorithm or leak, not the flag.*

---

### Q112. What do exceptions cost? Why not use them for flow control?

**Say this first:** "Throwing is cheap *only if no stack trace is built*. Creating an exception captures the whole stack — that's the expensive part."

**Answer — where the cost comes from:**
1. `new Exception()` calls `fillInStackTrace()`, a native method that walks the current thread's stack and records every frame. Cost grows with **stack depth** (deep Spring/servlet stacks have hundreds of frames).
2. Unwinding the stack to the matching `catch` handler is slower than a normal branch and can block some JIT optimisations.
3. Printing/logging the trace allocates lots of strings and I/O.
4. Stack traces are materialised into `StackTraceElement[]` lazily on first access (`getStackTrace()`/print).

Order of magnitude: a normal return is nanoseconds; throw + catch with a trace is microseconds — thousands of times slower in a hot loop. Measure with JMH rather than guess.

**Flow-control anti-pattern:**
```java
// ❌ exception as "not found" signal in a hot path
try { return Integer.parseInt(s); } catch (NumberFormatException e) { return -1; }   // in a loop of millions

// ✅ validate first / use an Optional-like result
Optional<User> findById(long id);          // empty = not found, no exception
boolean looksNumeric = s.chars().allMatch(Character::isDigit);
```
Also bad for readability: it hides the normal path inside `catch`, and `goto`-like control flow breaks reasoning.

**If you really need cheap exceptions** (rare — e.g. a parser backtracking, internal control signals):
```java
public class FastException extends RuntimeException {
    public FastException(String msg) {
        super(msg, null, false, false);      // enableSuppression=false, writableStackTrace=false
    }
}
// or: @Override public synchronized Throwable fillInStackTrace() { return this; }
```
Trade-off: no stack trace to debug with — acceptable only for expected, well-understood conditions.

**JVM-level trick to know (surprising in production):** with C2 JIT, hot *implicitly thrown* exceptions (`NullPointerException`, `ArrayIndexOutOfBoundsException`, `ArithmeticException`, `ClassCastException`, …) can be *re-thrown without a stack trace* (`-XX:+OmitStackTraceInFastThrow`, on by default). If logs suddenly show an NPE with an empty trace after many occurrences, look at earlier log entries or run with `-XX:-OmitStackTraceInFastThrow`.

**Pitfalls / follow-up probes:**
- *Is a `try` block itself costly?* No. Entering a `try` with no throw has essentially zero cost (exception tables are compile-time data).
- *Are checked exceptions slower than unchecked?* No, same mechanism.
- *When is it OK?* Truly exceptional conditions (I/O failure, invalid state). Failures that happen in 10% of normal inputs should be modelled as return values.
- *`Throwable` constructor with `writableStackTrace=false`* also suppresses `setStackTrace` use.

**Remember:** *Capturing the stack is the cost; use return values for expected outcomes, exceptions for exceptional ones.*

---

### Q113. Uncaught exceptions in threads and thread pools — `execute` vs `submit`, and the uncaught exception handler.

**Say this first:** "An exception in a background thread doesn't reach the caller. With `execute` it kills the worker (and prints to stderr); with `submit` it is silently stored in the `Future` — if nobody calls `get()`, you never see it."

**Answer — plain threads:** an exception that escapes `run()` terminates *that thread only*. Before dying, the JVM calls the thread's **`UncaughtExceptionHandler`**. Order of lookup:
1. handler set on the thread: `t.setUncaughtExceptionHandler(...)`;
2. handler of its `ThreadGroup`;
3. the global default: `Thread.setDefaultUncaughtExceptionHandler(...)`;
4. otherwise the default prints the stack trace to `System.err` — which many deployments don't collect.

```java
Thread.setDefaultUncaughtExceptionHandler((t, e) ->
    log.error("Uncaught in thread {}", t.getName(), e));   // set once at startup
```
The handler is for *logging/alerting*, not recovery — the thread is already gone.

**Thread pools (`ExecutorService`):**

| Call | If the task throws |
|---|---|
| `executor.execute(r)` | Exception propagates out of the worker → handler invoked → **worker thread dies** and the pool creates a replacement |
| `executor.submit(r)` / `submit(callable)` | Exception is **captured in the `Future`**. Nothing is logged, thread survives. `future.get()` throws `ExecutionException` wrapping it |

```java
Future<?> f = pool.submit(() -> { throw new IllegalStateException("lost!"); });
// nothing printed ... until:
try { f.get(); } catch (ExecutionException e) { log.error("task failed", e.getCause()); }
```
**Scheduled tasks — a classic trap:** with `scheduleAtFixedRate`/`scheduleWithFixedDelay`, if one run throws, **all future runs are cancelled silently**. Always wrap the body:
```java
scheduler.scheduleAtFixedRate(() -> {
    try { doWork(); } catch (Throwable t) { log.error("job failed", t); }   // keep the schedule alive
}, 0, 1, TimeUnit.MINUTES);
```
**Robust patterns:**
- Wrap tasks in a decorator that catches, logs, and (if needed) rethrows.
- Custom `ThreadFactory` that sets the uncaught handler and meaningful thread names.
- Subclass `ThreadPoolExecutor` and override `afterExecute(Runnable r, Throwable t)` — for `submit`, unwrap the `Future` to see the real exception.
- Always consume `Future`s you care about (or use `CompletableFuture` with `exceptionally`/`handle`).
- Virtual threads (Java 21) behave the same: uncaught exceptions go to the handler; they are not magically propagated to the parent.

**Pitfalls / follow-up probes:**
- *Does `Thread.join()` rethrow the exception?* No.
- *Does an `Error` in a pool thread stop the pool?* The worker dies and is replaced; the pool lives.
- *`ForkJoinPool` / parallel streams?* Exceptions are rethrown to the caller of `join`/terminal operation (possibly wrapped).
- *`InterruptedException` inside tasks:* restore the interrupt flag.

**Remember:** *`execute` → worker dies with a handler call; `submit` → swallowed into the Future. Always catch/log in scheduled jobs.*

---

### Q114. Exception handling in streams, lambdas and `CompletableFuture`.

**Say this first:** "Lambdas passed to streams can't throw checked exceptions; and in `CompletableFuture` the exception is carried inside the future — you handle it with `handle`/`exceptionally`, usually wrapped in `CompletionException`."

**Streams and lambdas — the problem:** functional interfaces like `Function<T,R>` don't declare `throws`, so this doesn't compile:
```java
paths.stream().map(p -> Files.readString(p));   // ❌ IOException is checked
```
**Options:**
```java
// 1) wrap in unchecked inside the lambda
paths.stream().map(p -> {
    try { return Files.readString(p); }
    catch (IOException e) { throw new UncheckedIOException(e); }   // JDK helper type
}).toList();

// 2) your own throwing functional interface + adapter
@FunctionalInterface interface ThrowingFn<T,R> { R apply(T t) throws Exception; }
static <T,R> Function<T,R> unchecked(ThrowingFn<T,R> f) {
    return t -> { try { return f.apply(t); } catch (Exception e) { throw new RuntimeException(e); } };
}

// 3) return a result type — keep the stream going
.map(p -> { try { return Optional.of(Files.readString(p)); } catch (IOException e) { return Optional.<String>empty(); } })
```
- Streams are **lazy** — the exception appears at the *terminal* operation (`collect`, `forEach`), not where you wrote `map`.
- One failing element aborts the whole pipeline unless you handle it per-element.
- Don't mutate shared state or swallow errors silently inside parallel streams; the first exception propagates (others may be lost).

**`CompletableFuture`:** an exception completes the future *exceptionally*, and downstream stages are skipped until something handles it.
```java
CompletableFuture.supplyAsync(() -> callRemote())
    .thenApply(this::parse)                       // skipped if callRemote failed
    .exceptionally(ex -> fallback())              // only on failure; returns a recovery value
    .whenComplete((r, ex) -> log(r, ex));         // observe; does NOT change the result

cf.handle((result, ex) -> ex == null ? result : fallback());   // success OR failure → new value
```
| Method | Runs on | Can change result? |
|---|---|---|
| `exceptionally(fn)` | failure only | yes (recovery value) |
| `handle(bifn)` | success and failure | yes |
| `whenComplete(biconsumer)` | success and failure | no (side effect; original outcome kept) |
| `exceptionallyCompose` (Java 12) | failure | yes, via another future |

**Wrapping gotcha:** an exception thrown inside a stage reaches *later* stages' `ex` as a **`CompletionException`** with your exception as `getCause()`. `join()` throws `CompletionException`; `get()` throws `ExecutionException`. Unwrap before checking types:
```java
Throwable real = (ex instanceof CompletionException && ex.getCause() != null) ? ex.getCause() : ex;
```
Add timeouts too (Java 9+): `cf.orTimeout(2, SECONDS)` / `completeOnTimeout(...)`; otherwise a stuck call never fails.

**Pitfalls / follow-up probes:**
- *Fire-and-forget `CompletableFuture`* with no handler: the exception vanishes. Always attach `exceptionally`/`whenComplete` with logging.
- *Which thread runs the handler?* The completing thread or the caller, unless you use the `*Async` variants.
- *`allOf` fails if any fails;* inspect individual futures for partial success.

**Remember:** *Wrap checked in lambdas; `handle` = success+failure, `exceptionally` = failure only; expect `CompletionException`.*

---

### Q115. The common runtime exceptions — when each occurs — plus fail-fast, `Objects.requireNonNull` and defensive programming.

**Say this first:** "Each runtime exception is a precise signal: wrong argument, wrong state, wrong type, wrong index, or wrong operation. Choose the one that tells the caller who made the mistake."

| Exception | When it happens |
|---|---|
| `NullPointerException` | Calling a method/accessing a field on `null`, unboxing null, `switch` on a null String/enum, `synchronized(null)`, throwing null. Since Java 14/15 *helpful NPE messages* name the null variable |
| `ClassCastException` | Invalid downcast `(String) obj` where obj isn't a `String`; heap pollution with raw/unchecked generics |
| `ArrayIndexOutOfBoundsException` / `StringIndexOutOfBoundsException` / `IndexOutOfBoundsException` | Index < 0 or ≥ length/size (`list.get(5)` on 3 elements) |
| `NumberFormatException` | `Integer.parseInt("12a")`; it **extends** `IllegalArgumentException` |
| `ArithmeticException` | Integer `/ 0` or `% 0` (double division gives `Infinity`/`NaN`, no exception) |
| `IllegalArgumentException` | **Caller passed a bad argument** — negative size, invalid enum name (`Enum.valueOf`) |
| `IllegalStateException` | **Object is in the wrong state** for this call — iterator `remove()` twice, using a closed resource, calling `start()` twice |
| `UnsupportedOperationException` | Operation not supported by design: `add()` on `List.of(...)`, `Collections.unmodifiableList`, or `Arrays.asList` (fixed-size) |
| `ConcurrentModificationException` | Structurally modifying a collection while iterating it (see below) |
| `ArrayStoreException`, `NegativeArraySizeException` | Storing wrong type into an array; `new int[-1]` |

**IllegalArgument vs IllegalState — the rule:** if the *same call would work* with a different argument, it's `IllegalArgumentException`. If it would fail *for any argument* until the object's state changes (not opened, already closed, not initialised), it's `IllegalStateException`.

**`ConcurrentModificationException` — how and why:** `ArrayList`/`HashMap` keep a `modCount` that increments on every structural change (add/remove). An iterator remembers the value when created and checks it on each `next()`. A mismatch → throws. This is **fail-fast, best-effort** — meant to expose bugs, not to guarantee thread safety, and it can happen in a single thread:
```java
for (String s : list) if (s.isEmpty()) list.remove(s);   // ❌ CME (usually)
list.removeIf(String::isEmpty);                          // ✅
it.remove();                                             // ✅ via Iterator
// concurrent use: CopyOnWriteArrayList / ConcurrentHashMap (weakly consistent iterators, no CME)
```

**Fail-fast vs defensive programming:**
- **Fail-fast:** detect a problem at the earliest point and stop loudly (validate inputs in constructors/public methods). The error appears next to its cause, not 20 frames later as a mystery NPE.
- **Defensive:** protect against bad *external* input and misuse (copy mutable inputs, validate, sanitise). Don't over-do it for *internal* invariants — hiding bugs with silent defaults (`if (x == null) x = ""`) is the opposite of fail-fast.

**`Objects.requireNonNull`** — the standard fail-fast null check:
```java
public Order(Customer customer, List<Item> items) {
    this.customer = Objects.requireNonNull(customer, "customer must not be null");
    this.items    = List.copyOf(Objects.requireNonNull(items, () -> "items required for " + customer));  // supplier: message built lazily
}
```
Other helpers: `Objects.requireNonNullElse(v, def)`, `Objects.checkIndex(i, len)` (throws `IndexOutOfBoundsException`), `Optional` for *return types* (not fields/parameters). `assert` is disabled by default (`-ea` needed) — never use it for argument validation.

**Pitfalls / follow-up probes:**
- *Throwing `NullPointerException` yourself?* `requireNonNull` does; it's the convention for null arguments (`Objects` docs).
- *Returning `null` collections:* return empty collections instead.
- *Don't catch NPE/CCE to "handle" them* — fix the bug.
- *`UnsupportedOperationException` from `List.of` in tests?* Immutable collection; copy to `new ArrayList<>(...)`.

**Remember:** *Pick the exception that names the culprit: bad arg → IAE, bad state → ISE, null → NPE (check early with `requireNonNull`).*

---

# 5. JVM, Memory & Garbage Collection

### Q116. Explain the JVM memory model / runtime data areas.

```
┌──────────────── JVM ────────────────────────────────┐
│ HEAP (shared)                                       │
│   Young Gen: Eden | Survivor S0 | Survivor S1        │
│   Old Gen (Tenured)                                  │
│                                                      │
│ METASPACE (native, off-heap since Java 8)            │
│   class metadata, static fields ref, constant pool   │
│                                                      │
│ Per-thread: JVM Stack (frames, locals, operand stack)│
│             PC Register, Native Method Stack          │
│                                                      │
│ Shared: Code Cache (JIT-compiled native code)         │
│         Direct/off-heap buffers (NIO, Netty)          │
└──────────────────────────────────────────────────────┘
```
- **Heap** — all objects and arrays. `-Xms` / `-Xmx`.
- **Stack** — one per thread, holds frames with local primitives and object *references*. `-Xss`. Overflow → `StackOverflowError`.
- **Metaspace** — replaced PermGen in Java 8; grows automatically unless capped with `-XX:MaxMetaspaceSize`. Leaks here usually mean **classloader leaks** (hot redeploys, dynamic proxies, Groovy).
- **String pool** — in the heap since Java 7.

**How the areas relate (plain language):** when a method is called, a **frame** is pushed on that thread's stack holding its local variables; primitives live right in the frame, but objects (`new Order()`) live on the **heap** and the frame holds only a reference. When the method returns, the frame is popped instantly (no GC needed) while the heap object stays until nothing references it. **Metaspace** holds the *description* of each loaded class (fields, methods, bytecode) — it is per-class, not per-object. Heap is shared by all threads (so it needs GC + thread-safety care); stacks are private (so local variables are inherently thread-safe).

```java
void handle() {
    int count = 3;                 // stack (primitive)
    Order o = new Order();         // 'o' reference → stack, Order object → heap
}                                  // frame popped; Order becomes garbage once unreferenced
```
**Typical flags:** `-Xms512m -Xmx2g` (heap start/max — set equal in prod to avoid resize pauses), `-Xss512k` (per-thread stack). Deep recursion → `StackOverflowError`; too many objects → `OutOfMemoryError: Java heap space`.

**Remember:** *Heap = objects, Stack = frames, Metaspace = classes, CodeCache = JIT output.*

---

### Q117. How does garbage collection work? Explain generational GC.

**Reachability:** objects reachable from **GC roots** (thread stacks, static fields, JNI refs, active class loaders) survive; the rest are garbage. Java uses **tracing (mark-sweep-compact)**, *not* reference counting — so cyclic references are collected fine.

**Weak generational hypothesis:** most objects die young.
1. New objects → **Eden**.
2. Eden full → **Minor GC** (stop-the-world, but short): live objects copied to a Survivor space, age++.
3. Survivors ping-pong S0 ↔ S1 each minor GC.
4. Age > `MaxTenuringThreshold` (default 15) → **promoted to Old Gen**. Huge objects may go straight to Old Gen.
5. Old Gen full → **Major/Full GC** — expensive.

**Reference types:** Strong > **Soft** (cleared only when memory is low — cache-friendly) > **Weak** (`WeakHashMap`, cleared at next GC) > **Phantom** (post-mortem cleanup).

**Why generational?** Scanning the whole heap every time would be slow. Since ~90%+ of objects (request DTOs, temp strings) die right after the request, the JVM collects only the small Young Gen frequently — cheap because it copies the few survivors and wipes Eden in one go. Long-lived objects (caches, singletons) migrate to Old Gen, which is collected rarely. "Stop-the-world" (STW) means all application threads pause while GC runs; the whole tuning game is keeping those pauses short.

**Example:** a `List<Order>` field in a singleton bean is reachable from a GC root (static/bean → list), so it is never collected; a local `List<Order>` created in a request method is unreachable once the method returns, so it dies in the next minor GC. Memory leaks in Java = objects that are *still reachable* but no longer needed.

**Remember:** *Eden → Survivor → Old; minor GC is cheap, full GC is the enemy.*

---

### Q118. Compare the garbage collectors and say which one you'd pick.

| GC | Flag | Best for | Notes |
|---|---|---|---|
| Serial | `-XX:+UseSerialGC` | Tiny heaps, single CPU, containers < 1GB | Single-threaded |
| Parallel (Throughput) | `-XX:+UseParallelGC` | Batch jobs, max throughput, latency doesn't matter | Java 8 default |
| **G1** | `-XX:+UseG1GC` | General purpose services, heaps 4–32 GB | **Default since Java 9.** Region-based, concurrent marking, predictable pause target via `-XX:MaxGCPauseMillis=200` |
| ZGC | `-XX:+UseZGC` | Low latency, huge heaps (TB) | Sub-millisecond pauses, colored pointers; generational in 21 |
| Shenandoah | `-XX:+UseShenandoahGC` | Low latency, mid heaps | Concurrent compaction |
| Epsilon | — | Benchmarks/tests only | No-op collector |

**My default answer:** "For a typical Spring Boot microservice in Kubernetes with a 2–4 GB heap I stay on **G1** with `-XX:MaxGCPauseMillis` tuned, `-XX:+UseContainerSupport` and `-XX:MaxRAMPercentage=75`. I only move to ZGC when p99 latency SLOs are tight and heap is large."

**Container flags that matter:**
```
-XX:MaxRAMPercentage=75.0        # instead of hardcoded -Xmx
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps
-XX:+ExitOnOutOfMemoryError      # let K8s restart the pod
-Xlog:gc*:file=/logs/gc.log:time,uptime:filecount=5,filesize=10M
```

**Remember:** *Parallel = throughput, G1 = balanced default, ZGC = latency.*

---

### Q119. Types of `OutOfMemoryError` and how you diagnose each.

| Message | Cause | Fix |
|---|---|---|
| `Java heap space` | Real leak or heap too small | Heap dump → MAT dominator tree |
| `GC overhead limit exceeded` | >98% time in GC, <2% reclaimed | Same as above; usually a leak |
| `Metaspace` | Classloader leak, too many proxies/dynamic classes | Check for redeploy leaks, cap MaxMetaspaceSize |
| `unable to create new native thread` | Thread leak / OS limit | Bound your executors; check `ulimit -u` |
| `Direct buffer memory` | Netty/NIO off-heap leak | `-XX:MaxDirectMemorySize`, check buffer release |
| `Requested array size exceeds VM limit` | Array > 2^31-1 | Paginate |

**Diagnosis workflow:**
1. `jcmd <pid> GC.heap_info` and GC logs → is it growing monotonically after full GCs?
2. `jcmd <pid> GC.class_histogram` → what class dominates?
3. `jcmd <pid> GC.heap_dump /tmp/heap.hprof` (or the `-XX:+HeapDumpOnOutOfMemoryError` file).
4. Open in **Eclipse MAT** → *Leak Suspects* → **Dominator Tree** → find the GC-root path holding the objects.
5. Fix, then verify with a soak test.

**Remember:** *Histogram tells you WHAT, dominator tree tells you WHO is holding it.*

---

### Q120. Common memory leak causes in a long-running Java application.

1. **Unbounded caches** — a `HashMap` used as a cache with no eviction. Fix: Caffeine/Redis with TTL + max size.
2. **`static` collections** that only ever grow.
3. **`ThreadLocal` not cleaned** — with a pooled thread the value survives the request forever. Always `remove()` in a `finally` or use a filter.
4. **Unclosed resources** — streams, `Connection`, `HttpClient` responses, `EntityManager`.
5. **Listener/callback registration without deregistration** (observers, JMS listeners, `@EventListener` on prototype beans).
6. **`ClassLoader` leaks** on hot redeploy (Tomcat) — usually a driver/TL/shutdown hook holding the webapp loader.
7. **Non-static inner classes / lambdas** capturing `this` and outliving the outer object.
8. **Session bloat** — dumping large objects into `HttpSession`.
9. **Hibernate first-level cache** in a long transaction reading millions of rows — no `clear()`/`flush()`. Fix: batch with `entityManager.flush(); entityManager.clear();` or use a stateless session.
10. **`String.substring` in old JDKs** / interning user input into the string pool.

**Three worth being able to show in code:**
```java
// 1. Unbounded cache — grows forever
private static final Map<String, Report> CACHE = new HashMap<>();   // ❌ no eviction
// fix: Caffeine.newBuilder().maximumSize(10_000).expireAfterWrite(10, MINUTES).build();

// 3. ThreadLocal on a pooled Tomcat thread — value outlives the request
CONTEXT.set(user);
try { chain.doFilter(req, res); } finally { CONTEXT.remove(); }     // ✅ always clean up

// 9. Hibernate persistence context holds every loaded entity until the tx ends
for (int i = 0; i < ids.size(); i++) {
    process(em.find(Order.class, ids.get(i)));
    if (i % 500 == 0) { em.flush(); em.clear(); }                    // release managed entities
}
```
**How you'd spot it in production:** heap usage after each full GC trends upward like a staircase (a healthy app returns to a stable baseline) → take a heap dump and look at the dominator tree (see Q119).

**Remember:** *Leaks = "something static, pooled, or cached is still holding a reference."*

---

### Q121. What is JIT? Explain C1/C2 and tiered compilation.

The JVM starts **interpreting** bytecode, profiles execution, and hot methods get compiled to native code by the **JIT**:
- **C1 (client)** — compiles fast, optimizes lightly. Good for startup.
- **C2 (server)** — compiles slower, optimizes aggressively (inlining, escape analysis, loop unrolling, dead code elimination, lock elision).
- **Tiered compilation** (default) uses C1 first, then promotes hot methods to C2.

**Consequences you should mention:**
- **Warm-up** — the first N thousand requests are slower. That's why you warm up before load tests, and why microbenchmarks need JMH.
- **Escape analysis** can allocate on the stack and remove locks — a reason not to hand-optimize.
- `-XX:TieredStopAtLevel=1` speeds up startup for short-lived CLI/batch apps.
- **AOT / CRaC / GraalVM Native Image** solve cold start for serverless (Spring Boot 3 supports `native-image` via AOT processing).

**In plain words:** bytecode is portable but slow to interpret line by line. The JVM counts how often each method/loop runs; once it is "hot" (thousands of invocations) the JIT (*Just-In-Time* compiler) translates it to native machine code at runtime and caches it in the Code Cache, using real profile data (e.g. "this branch is always taken", "this interface only has one implementation") to optimise more than an ahead-of-time compiler could. If an assumption later proves wrong the JVM **deoptimises** back to the interpreter.

**Practical example:** after a deploy, the first minute of traffic shows higher p99 latency, then it drops — that's JIT warm-up. In Kubernetes this is why a freshly started pod behind a load balancer can look "slow"; mitigate with readiness probes + a warm-up routine, or slow-start ramping at the load balancer, rather than assuming a code bug. `-XX:+PrintCompilation` shows what is being compiled.

**Remember:** *Interpret → C1 → C2; warm-up is real, so benchmark with JMH.*

---

### Q122. Explain the class loading phases — load, link (verify, prepare, resolve) and initialize.

**Say this first:** "A class goes through Loading → Linking (Verification, Preparation, Resolution) → Initialization. It is done lazily, the first time the class is actually needed."

**Answer — why lazy?** A big app has thousands of classes; loading them all at start would make startup slow and waste memory. The JVM loads a class only when it's first used.

```
 .class bytes ──► LOADING ──► LINKING ─────────────────────────► INITIALIZATION
                  (find bytes,  ├ Verification (is bytecode safe?)   (run static
                   create the   ├ Preparation  (allocate statics,     initializers
                   Class object)│               set DEFAULT values)   <clinit>)
                                └ Resolution   (symbolic → direct refs)
```
1. **Loading** — a class loader finds the binary (`.class` from a jar/dir/network), parses it, and creates the in-memory class representation (metadata in Metaspace plus a `java.lang.Class` object on the heap). The superclass and interfaces are loaded first.
2. **Verification** — checks the bytecode is structurally valid and type-safe (no stack underflow, no illegal casts, correct method signatures). This protects the JVM from corrupt or malicious classes. (Turning verification off via `-Xverify:none` is deprecated; don't.)
3. **Preparation** — allocates memory for `static` fields and sets them to **default values** (`0`, `false`, `null`). Not your assigned values yet!
4. **Resolution** — replaces symbolic references in the constant pool ("method `Foo.bar()`") with direct references. Implementations may do this lazily, when the instruction is first executed.
5. **Initialization** — runs the class initializer `<clinit>`: static field assignments and `static {}` blocks in textual order. Superclass is initialized first. It's executed **once**, thread-safely (the JVM holds an init lock).

```java
class A {
    static int x = 5;                      // preparation: x = 0;  initialization: x = 5
    static { System.out.println("A init"); }
}
```
**What triggers initialization:** `new A()`, calling a static method, reading/writing a static field (except compile-time constants like `static final int K = 5`, which are inlined), `Class.forName("A")`, initializing a subclass, and the main class at startup. `Class.forName(name, false, loader)` and `A.class` **don't** initialize.

**Pitfalls / follow-up probes:**
- *What if `<clinit>` throws?* `ExceptionInInitializerError` the first time; later uses fail with `NoClassDefFoundError: Could not initialize class A`.
- *Deadlock?* Two classes whose static initializers depend on each other, initialised by two threads, can deadlock.
- *Singleton holder idiom* works because class initialization is lazy and thread-safe.
- *Initialization order of statics vs instance:* see the guide's static vs instance initialization question.

**Remember:** *Load → Verify → Prepare (defaults) → Resolve → Initialize (real values, once).*

---

### Q123. The class loader hierarchy, parent delegation, and custom class loaders.

**Say this first:** "Loaders form a parent chain. A loader first asks its parent; only if the parent can't find the class does it try itself. That keeps core classes safe and consistent."

**Answer — the built-in loaders (Java 9+):**
```
Bootstrap  (native; loads java.base etc. from the runtime image)   getClassLoader() == null
    ▲
Platform   (other JDK modules: java.sql, java.xml ...)            ClassLoader.getPlatformClassLoader()
    ▲
Application / System  (your classpath / module path)              ClassLoader.getSystemClassLoader()
```
In Java 8 the middle one was the **Extension** loader (`jre/lib/ext`), removed in 9 and replaced by the Platform loader.

**Delegation algorithm** (`ClassLoader.loadClass(name)`):
1. `findLoadedClass(name)` — already loaded by me?
2. Else ask `parent.loadClass(name)` (the bootstrap at the top).
3. Else `findClass(name)` — try to load it myself.
4. Not found anywhere → `ClassNotFoundException`.

**Why parent-first?**
- **Security:** you can't ship your own `java.lang.String` — the bootstrap loader always wins.
- **Uniqueness:** a class is identified by its **name + defining loader**. The same `.class` loaded by two loaders gives two *different* types (`ClassCastException: Foo cannot be cast to Foo`).

**Custom class loaders — why and how:** plugin systems, hot reload, isolating different versions of a library, loading from DB/network, app servers (each web app gets its own loader). Override `findClass`, not `loadClass`, to keep delegation:
```java
class PluginLoader extends ClassLoader {
    private final Path dir;
    PluginLoader(Path dir, ClassLoader parent) { super(parent); this.dir = dir; }
    @Override protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            byte[] b = Files.readAllBytes(dir.resolve(name.replace('.', '/') + ".class"));
            return defineClass(name, b, 0, b.length);          // bytes → Class
        } catch (IOException e) { throw new ClassNotFoundException(name, e); }
    }
}
```
Prefer `URLClassLoader` when loading jars.

**Child-first** (web containers like Tomcat): for application classes the loader tries itself *before* the parent, so a webapp can use its own library version; it still delegates JDK classes to the parent.

**Thread context class loader (TCCL):** `Thread.currentThread().getContextClassLoader()` lets *core* code (JNDI, JDBC drivers via `ServiceLoader`) load classes from the *application* loader, which is a child they couldn't otherwise see.

**Pitfalls / follow-up probes:**
- *A class loader leak* keeps all its classes alive in Metaspace (a static or thread holding any class/instance from the loader) — typical in hot redeploys.
- *Unloading:* a class can be unloaded only when its loader is unreachable.
- *Can you break delegation?* Yes by overriding `loadClass`; do it carefully.
- *Does the `Class` object live on the heap?* Yes; its metadata is in Metaspace.

**Remember:** *Ask the parent first; class identity = name + loader; custom loaders override `findClass`.*

---

### Q124. `ClassNotFoundException` vs `NoClassDefFoundError` — and how to debug them.

**Say this first:** "`ClassNotFoundException` is a checked exception when you load a class *by name* and it isn't there. `NoClassDefFoundError` is an error: the class was there at compile time (or loaded before) but the JVM can't *define* it now."

| | `ClassNotFoundException` | `NoClassDefFoundError` |
|---|---|---|
| Type | Checked `Exception` | `Error` (`LinkageError`) |
| Triggered by | `Class.forName("x.Y")`, `ClassLoader.loadClass`, `ServiceLoader`, reflection by name | Normal code (`new Y()`, static call) that implicitly needs `Y` |
| Meaning | "Not on the classpath of this loader (right now)" | "Was available when compiled, missing/broken at runtime" |
| Recovery | Handle it (plugin missing → skip) | Usually fatal; fix deployment |

**Common causes of `NoClassDefFoundError`:**
1. A dependency jar is present at build time but missing at runtime (`provided`/`optional` scope, shaded jar issue).
2. **Static initializer failed earlier**: the first use gave `ExceptionInInitializerError`; later uses give `NoClassDefFoundError: Could not initialize class X`. Search logs for the *first* error.
3. Class visible to one loader but not another (parent/child loader visibility; TCCL mismatch).
4. Version conflicts — wrong jar version leads to `NoSuchMethodError`/`NoSuchFieldError`/`IncompatibleClassChangeError` (other `LinkageError`s).
5. Case-sensitivity differences between Windows (dev) and Linux (prod).

**Related errors:**
- `UnsupportedClassVersionError` — class compiled for a newer Java than the running JVM (class file major version: Java 8=52, 11=55, 17=61, 21=65).
- `ClassCastException: X cannot be cast to X` — same class name from two loaders.

**Debugging steps:**
```bash
java -verbose:class -cp app.jar Main | grep Foo          # or -Xlog:class+load=info
mvn dependency:tree  /  gradle dependencies                # duplicate/missing versions
jar tf lib.jar | grep Foo                                  # is it inside?
jcmd <pid> VM.classloader_stats                            # loaders and their classes
```
In code, print where a class came from: `Foo.class.getProtectionDomain().getCodeSource().getLocation()`.

**Pitfalls / follow-up probes:**
- *Why is one checked and the other an Error?* The first is an explicit lookup you can plan for; the second is a linkage inconsistency in the program itself.
- *Why "Could not initialize class"?* The class is marked erroneous after init failure.
- *Fat-jar/shading* can hide or duplicate classes — check `META-INF` and relocation settings.

**Remember:** *CNFE = looked up by name and not found; NCDFE = was there at compile time, unusable at runtime (check init failure first).*

---

### Q125. Bytecode basics, `javap`, and the structure of a stack frame.

**Say this first:** "`javac` compiles to bytecode for a *stack-based* virtual machine. `javap -c` shows it. Each method call gets a frame with local variables and an operand stack."

**Answer — why bytecode?** It's a portable instruction set (≈200 opcodes), so the same `.class` runs on any OS/CPU with a JVM — the JVM then interprets it or JIT-compiles it to native code.

```java
int add(int a, int b) { int c = a + b; return c; }
```
```bash
javac Calc.java && javap -c -p Calc          # -p includes private; -v = constant pool, flags, stack/locals sizes
```
```
int add(int, int);
  0: iload_1      // push local[1] (a)
  1: iload_2      // push local[2] (b)
  2: iadd         // pop two, push sum
  3: istore_3     // pop → local[3] (c)
  4: iload_3
  5: ireturn
```
Local slot 0 is `this` for instance methods (absent in static methods). Prefix tells the type: `i`=int, `l`=long, `a`=reference, `d`=double.

**What's in a frame (one per method call, on that thread's stack):**
```
┌──────────────────────────────┐
│ Local variable array         │  this, params, locals (long/double take 2 slots)
│ Operand stack                │  scratch area for computations
│ Reference to runtime         │  to resolve fields/methods of this class
│   constant pool              │
│ (return address / exception  │
│   handling info)             │
└──────────────────────────────┘
```
The sizes (`max_locals`, `max_stack`) are computed by `javac`, so a frame has a fixed size. Calls push a new frame; `return` or an exception pops it.

**Invocation instructions you should recognise:**
| Opcode | Used for |
|---|---|
| `invokevirtual` | Normal instance methods (dynamic dispatch) |
| `invokeinterface` | Interface methods |
| `invokespecial` | Constructors, `super.m()`, private methods |
| `invokestatic` | Static methods |
| `invokedynamic` | Lambdas, (since Java 9) string concatenation, records' `toString/equals`, pattern switch — linked at runtime via bootstrap methods |

**Useful things `javap` explains:**
- why `i++` in `Integer` fields is not atomic (`getfield`, `iadd`, `putfield`);
- what a lambda compiles to (`invokedynamic` → hidden class, not an inner class file);
- that `synchronized` blocks use `monitorenter/monitorexit`;
- try/finally code is *duplicated* on each exit path by `javac` (see the finally question).

**Pitfalls / follow-up probes:**
- *Does bytecode execution equal what's executed on CPU?* No — the JIT re-optimises hot code, so bytecode isn't a performance predictor.
- *Where's the stack frame for a native method?* On the native method stack.
- *Why `StackOverflowError`?* Too many frames for the thread's stack size (`-Xss`).

**Remember:** *Bytecode = stack machine; `javap -c -p` to read it; a frame = locals + operand stack + constant-pool ref.*

---

### Q126. What lives where — heap, stack, Metaspace, code cache — and what happened to PermGen?

**Say this first:** "Objects live on the heap, locals live in stack frames, class metadata lives in Metaspace (native memory), JIT'd code lives in the code cache."

*(The diagram of runtime data areas is covered in the earlier JVM memory-model question; this one answers "where does X live?")*

| Thing | Where | Notes |
|---|---|---|
| Objects, arrays (`new ...`) | **Heap** | Eden/Survivor/Old (see GC questions) |
| Local primitives, object *references* in a method | **Thread stack** (frame) | Gone when the method returns |
| Instance fields | Heap (inside the object) | Primitive fields inline, reference fields hold addresses |
| **Static fields** | In the `java.lang.Class` mirror object on the **heap** (since Java 7/8) | Their *descriptions* are in Metaspace |
| Class metadata: methods, bytecode, field layout, vtable | **Metaspace** | Per-class, not per-object |
| Runtime constant pool | **Metaspace** | Symbolic references per class |
| String literals / interned strings | **Heap** (string table, since Java 7) | Table entries are weak; strings are GC-able |
| JIT compiled native code | **Code cache** | `-XX:ReservedCodeCacheSize` |
| Thread stacks, direct buffers, JNI, GC data structures | **Native (off-heap)** memory | Not counted in `-Xmx` |
| Lambdas' captured variables | Captured into a lambda object on the heap (when capturing) | Non-capturing lambdas can be cached singletons |

**PermGen → Metaspace (Java 8):**
- **PermGen** was a fixed-size part of the *heap* (`-XX:MaxPermSize`) holding class metadata (and, before 7, interned strings and statics). It caused frequent `OutOfMemoryError: PermGen space`, especially on redeploys, because sizing was guesswork.
- **Metaspace** lives in **native memory**, grows on demand, and is freed when the owning class loader dies. Default is *unlimited* (bounded by the OS/container), so cap it: `-XX:MaxMetaspaceSize=256m`; `-XX:MetaspaceSize` is only the initial high-water mark that triggers the first GC.
- There's also a **compressed class space** (part of metaspace area) for class pointers when compressed class pointers are on (`-XX:CompressedClassSpaceSize`).

**String pool and deduplication:** the String pool (string table) is in the **heap** (moved from PermGen in Java 7), sized with `-XX:StringTableSize`. Interning huge numbers of unique strings bloats it. **String deduplication** (`-XX:+UseStringDeduplication`, originally G1-only; newer JDKs added more collectors — check JDK docs) lets the GC point equal `char/byte[]` values to one shared array; it merges the *backing array*, not the `String` objects, and costs a little CPU. It's useful for apps holding millions of duplicate strings.

**Pitfalls / follow-up probes:**
- *Does an object ever live on the stack?* Not by design; the JIT may eliminate the allocation via scalar replacement (see the escape analysis question).
- *Can Metaspace still OOM?* Yes: `OutOfMemoryError: Metaspace` from class loader leaks or too many generated classes (proxies, scripting).
- *Is the stack part of `-Xmx`?* No — `-Xss` × thread count is extra.

**Remember:** *Heap = objects (and string pool, static values via Class), stack = frames, Metaspace = class metadata in native memory.*

---

### Q127. Object memory layout, the object header and compressed oops.

**Say this first:** "A Java object is header + fields + padding. On a 64-bit JVM with defaults the header is 12 bytes, objects are 8-byte aligned, and references are 4 bytes thanks to compressed oops."

**Answer — layout of a normal object:**
```
┌───────────────┬─────────────────┬──────────────┬─────────┐
│ mark word (8) │ class pointer(4)│ fields ...   │ padding │
└───────────────┴─────────────────┴──────────────┴─────────┘
  hash code, GC age,  points to class
  lock state          metadata (Metaspace)
Arrays: header + 4-byte length, then elements.
```
- **Mark word** — stores identity hash code (once computed), GC age, and lock state (thin/fat lock, etc.). This is why `synchronized` needs no extra memory per object.
- **Class pointer** — locates the class metadata. It's 4 bytes with *compressed class pointers*, 8 bytes without.
- **Alignment** — object size is rounded up to a multiple of 8 bytes. An empty `new Object()` is 16 bytes (12 header + 4 padding); an `Integer` is 16 bytes (12 + 4); a `Long`/`Double` is 24 bytes (12 + 8 = 20, rounded up) — verify with JOL.

**Compressed oops ("ordinary object pointers"):** a 64-bit reference would be 8 bytes, doubling memory for reference-heavy graphs and hurting CPU cache. Because objects are 8-byte aligned, the JVM stores a 32-bit value and shifts it left 3 bits to get the real address — so 32 bits can address **~32 GB** of heap. Enabled by default when `-Xmx` is below about 32 GB.
- **The 32 GB cliff:** a heap of 33 GB can hold *less* live data than 31 GB because all references suddenly double. Choose ≤ ~31 GB, or go well beyond (e.g. 40+ GB).
- Check: `java -XX:+PrintFlagsFinal -version | grep UseCompressedOops`.

**Why it matters in practice:**
- A `List<Integer>` of 1M entries costs ~(4 ref + 16 object) ≈ 20 MB, vs ~4 MB for `int[]` — boxed primitives and many small objects waste memory. Prefer primitive arrays/specialised collections for huge data.
- Field order is rearranged by the JVM to reduce padding. `@Contended` can pad fields apart to avoid false sharing (JDK-internal, needs a flag for user code).
- Tools: **JOL** (Java Object Layout, OpenJDK project) to print layout:
```java
System.out.println(ClassLayout.parseClass(Order.class).toPrintable());
```
**Newer JDKs:** work on *compact object headers* (8-byte header) is available in recent JDKs (introduced as experimental, later a product feature in newer versions) — check the JDK docs for your version.

**Pitfalls / follow-up probes:**
- *Shallow vs retained size* in heap dumps: shallow = the object's own layout; retained = what would be freed if it died.
- *Identity hash code* isn't stored until first requested (`System.identityHashCode`/default `hashCode`).
- *Why do 8 booleans not cost 8 bytes?* Each `boolean` field takes 1 byte, padded in the object.

**Remember:** *Header (12) + fields + padding to 8; compressed oops keep refs 4 bytes up to ~32 GB.*

---

### Q128. Escape analysis, inlining, deoptimization and warm-up — what the JIT really does.

**Say this first:** "The JIT optimises using what it *observes at runtime*; if its assumption breaks, it deoptimises. That's why Java code gets faster after warm-up and why microbenchmarks lie."

*(The basic C1/C2/tiered explanation is in the earlier JIT question; this goes deeper.)*

**Answer — key optimisations (C2):**
1. **Inlining** — copy a small callee's body into the caller. This removes call overhead and, more importantly, unlocks the other optimisations. Limits: `-XX:MaxInlineSize` (35 bytecodes, cold) and `-XX:FreqInlineSize` (325, hot). Tiny getters are always inlined.
2. **Escape analysis (EA)** — checks whether an object can be seen outside the method/thread:
   - *NoEscape* → **scalar replacement**: the object is never created; its fields live in registers/stack slots. (So "stack allocation" in Java is really *no allocation*.)
   - Locks on non-escaping objects → **lock elision** (the `synchronized` disappears). Related: *lock coarsening* merges adjacent locks.
```java
double len(double x, double y) {
    Point p = new Point(x, y);          // doesn't escape → likely no allocation after JIT
    return Math.sqrt(p.x * p.x + p.y * p.y);
}
```
3. **Speculative optimisations using profiles:** "this `if` is never taken", "this interface call has only one implementation" (monomorphic, via class hierarchy analysis) → direct, inlined call. Plus loop unrolling, range-check elimination, dead-code elimination, intrinsics (e.g. `Math`, `String`, `Arrays.copyOf` use CPU instructions).

**Deoptimization:** each speculation is guarded by an **uncommon trap**. If the assumption fails (e.g. a second implementation is loaded and the call site becomes polymorphic), the JVM discards the compiled code ("made not entrant"), falls back to the interpreter, re-profiles, and recompiles. A few deopts are normal; frequent deopt loops hurt latency. See them with `-XX:+PrintCompilation` (look for `made not entrant`) or JFR events.

**On-Stack Replacement (OSR):** a long-running loop in a method still executing gets compiled and swapped in mid-loop.

**Warm-up consequences:**
- First requests run interpreted or C1 code — slow; hot paths reach C2 after thousands of invocations/loop iterations (tiered: levels 0–4).
- After a deploy, latency spikes until warmed → use warm-up traffic, gradual rollout, or CDS/AOT features (see the startup optimisation question).
- Benchmarks: use **JMH** (it handles warm-up, dead-code elimination, forks). Naïve `System.nanoTime()` loops mostly measure the interpreter or an optimiser that deleted your code.
- **Code cache** full → JIT stops (`CodeCache is full` warning) → performance cliff; `-XX:ReservedCodeCacheSize`.

**Java Memory Model link (brief):** the JIT and CPU may *reorder or cache* memory operations as long as single-thread behaviour is unchanged. The JMM (see "Explain the Java Memory Model and happens-before") defines what multithreaded code may rely on; `volatile`/`synchronized` insert the barriers that stop the JIT from, e.g., hoisting a field read out of a loop (the classic `while (!stop)` hang).

**Pitfalls / follow-up probes:**
- *"Don't pre-optimise allocations"* — small, short-lived, non-escaping objects are cheap or free.
- *Megamorphic call sites* (3+ implementations) can't be inlined → slower virtual calls.
- *Huge methods (> 8000 bytecodes) aren't JIT-compiled* by default (`-XX:-DontCompileHugeMethods`).

**Remember:** *Profile → speculate → inline/scalar-replace → deopt if wrong; always warm up before measuring.*

---

### Q129. GC roots, reachability, and the four reference types (strong, soft, weak, phantom).

**Say this first:** "An object is garbage when no GC root can reach it through strong references. Soft, weak and phantom references let you *point at* an object without keeping it alive."

**Answer — GC roots** (where tracing starts):
- local variables and parameters in active stack frames of **live threads**;
- **static fields** of loaded classes (and the classes/class loaders themselves);
- **JNI references** (native code) and objects used as monitors/locks;
- internal JVM structures (system classes, interned strings table references, etc.).

GC marks everything reachable from roots; everything else is garbage — so **cycles are fine** (no reference counting). A *memory leak* in Java = unwanted object still reachable from a root.

**Reference strengths:**
| Type | Cleared when | Typical use |
|---|---|---|
| **Strong** (`Foo f = new Foo()`) | never while reachable | normal code |
| **Soft** (`SoftReference<T>`) | JVM is low on memory (before throwing OOM; policy-dependent) | memory-sensitive caches (prefer Caffeine with a size bound in real code) |
| **Weak** (`WeakReference<T>`, `WeakHashMap`) | next GC once only weakly reachable | canonicalising maps, listener registries, `ThreadLocal` keys, metadata keyed by objects you don't own |
| **Phantom** (`PhantomReference<T>`) | `get()` always returns `null`; enqueued after the object is finalizable/cleared | post-mortem cleanup (this is what `Cleaner` uses) |

```java
WeakReference<Image> ref = new WeakReference<>(bigImage);
bigImage = null;                // now only weakly reachable
Image i = ref.get();            // may be null after GC → always null-check
```
**ReferenceQueue:** pass one to the reference; after the referent is cleared the JVM enqueues the reference so you can run cleanup:
```java
ReferenceQueue<Resource> q = new ReferenceQueue<>();
PhantomReference<Resource> p = new PhantomReference<>(res, q);
// a daemon thread: Reference<?> r = q.remove(); → release native handle
```
**Notes:**
- `WeakHashMap` weakly references **keys** only; if the *value* strongly references the key, it's never cleared (classic leak).
- Soft references make GC behaviour hard to predict (they're retained longer under big heaps); many teams avoid them for caches.
- Reachability levels (strongly → softly → weakly → finalizer → phantom → unreachable) are evaluated by the GC; `Reference.reachabilityFence(obj)` prevents an object being collected too early in rare native-interop cases.

**Pitfalls / follow-up probes:**
- *`System.gc()` guaranteed to clear weak refs?* It's a hint (often honoured), not a contract. Don't use it as logic.
- *Why do leaks happen if Java has GC?* Because "reachable" ≠ "needed".
- *Phantom vs weak:* weak is cleared *before* finalization; phantom *after*, and you can't resurrect.
- *Class loaders* are roots too: a loaded class stays alive as long as its loader is reachable.

**Remember:** *Roots = stacks + statics + JNI; strong keeps, soft = on memory pressure, weak = next GC, phantom = cleanup hook.*

---

### Q130. Why is `finalize()` deprecated, and what replaces it (`Cleaner`, try-with-resources)?

**Say this first:** "Finalizers are unpredictable, slow and unsafe. Use try-with-resources for deterministic cleanup and `Cleaner` only as a safety net."

**Answer — what finalization was:** `Object.finalize()` was called by the GC before reclaiming an object, supposed to release native resources (files, sockets).

**Why it's a bad idea:**
1. **No timing guarantee** — it may run seconds later or never (JVM exit). You can't release scarce resources like file descriptors reliably.
2. **Slows GC and delays freeing** — a finalizable object needs a special queue, a finalizer thread, and survives at least one extra GC cycle (memory held longer); heavy finalizer use can trigger OOM.
3. **Resurrection** — `finalize()` could make `this` reachable again.
4. **Exceptions are ignored**, ordering between objects is undefined, finalizer thread can be a bottleneck.
5. **Security** — "finalizer attacks" on partially constructed objects.

**Status:** `finalize()` deprecated in Java 9, **deprecated for removal in Java 18** (JEP 421). Finalization can be turned off for testing with `--finalization=disabled`. Check JDK docs for the current removal state in your version.

**Replacements:**
```java
// 1) Deterministic: AutoCloseable + try-with-resources  (preferred)
try (var conn = pool.acquire()) { ... }

// 2) Safety net: java.lang.ref.Cleaner (Java 9)
public class NativeHandle implements AutoCloseable {
    private static final Cleaner CLEANER = Cleaner.create();
    private final Cleaner.Cleanable cleanable;
    private final State state;                       // MUST NOT reference NativeHandle itself

    NativeHandle(long address) {
        this.state = new State(address);
        this.cleanable = CLEANER.register(this, state);   // run state when 'this' becomes phantom reachable
    }
    @Override public void close() { cleanable.clean(); }  // explicit + idempotent

    private static class State implements Runnable {      // static nested: no hidden ref to outer
        final long address; State(long a) { address = a; }
        @Override public void run() { /* free native memory at address */ }
    }
}
```
- `Cleaner` is built on **phantom references**; the cleaning action runs on a daemon thread and runs at most once.
- The action must **not** capture the object (including through lambdas or inner classes), or it'll never become unreachable.
- Still no timing guarantee, so it's a *backstop*; always provide `close()`.

**Pitfalls / follow-up probes:**
- *Does `Closeable` replace everything?* It fixes cleanup when users call it; Cleaner covers forgetting.
- *`finalize()` vs `finally`:* completely unrelated; `finally` is deterministic control flow.
- *Detecting leaks of unclosed resources:* log from the Cleaner action ("resource leaked") in test builds.
- *Library authors:* JDK itself migrated `FileInputStream`, etc., away from finalizers.

**Remember:** *Don't finalize; close explicitly (try-with-resources); Cleaner is a late safety net.*

---

### Q131. GC algorithms in depth — mark-sweep, mark-compact, copying — and Serial, Parallel, CMS history and Epsilon.

**Say this first:** "Collectors combine three basic techniques: *mark-sweep* (cheap but fragments), *mark-compact* (slides survivors together), and *copying* (evacuate live objects to a fresh area)."

*(Comparison table of collectors is in the earlier GC question; this is the underlying mechanics.)*

**Basic algorithms:**
| Technique | How | Pros | Cons |
|---|---|---|---|
| **Mark-Sweep** | Mark reachable; free the unmarked | Simple, no object moves | **Fragmentation** → can't fit a big object even with free total space; slower allocation (free lists) |
| **Mark-Compact** | Mark; then slide live objects to one end; update references | No fragmentation, fast bump-pointer allocation | Extra pass; whole-heap pause unless done cleverly |
| **Copying** (semi-space) | Copy live objects to an empty space; wipe the old one | Cost ∝ *live* data (great when most is dead); compacts for free | Needs spare space; bad when most data survives |

Generational GC uses **copying** for Young (few survivors) and **mark-compact** or **mark-sweep** variants for Old.

**Marking** uses three colours (tri-colour): white = unseen, grey = seen but children not scanned, black = fully scanned. Concurrent collectors need **barriers** so app threads that change references during marking don't hide live objects (G1: *SATB* snapshot-at-the-beginning write barrier; ZGC/Shenandoah: load barriers).

**Collectors from this angle:**
- **Serial** (`-XX:+UseSerialGC`): single GC thread, STW. Young = copying; Old = mark-sweep-compact. Smallest footprint; JVM ergonomics choose it on "small" machines (< 2 CPUs or < ~1.8 GB RAM) *even if you expected G1* — a common surprise in tiny containers.
- **Parallel** (`-XX:+UseParallelGC`): same algorithms, but multi-threaded STW; maximises **throughput**, pauses grow with heap.
- **CMS** (concurrent mark-sweep): old-gen mostly concurrent but fragmenting; deprecated in 9, **removed in Java 14**. Seen in old Java 8 tuning guides.
- **G1 / ZGC / Shenandoah:** region-based, mostly concurrent — see the next questions.
- **Epsilon** (`-XX:+UnlockExperimentalVMOptions -XX:+UseEpsilonGC`): allocates but **never reclaims**; process dies at heap exhaustion. Use for GC-free benchmarks, short-lived jobs, or measuring allocation behaviour.

**Choosing in one line:** tiny/single CPU → Serial; batch throughput → Parallel; general services → G1; strict latency & big heaps → ZGC (or Shenandoah); experiments → Epsilon.

**Pitfalls / follow-up probes:**
- *Why not only mark-sweep?* Fragmentation causes allocation failures and slow allocation.
- *Why is allocation in Java so fast?* After compaction/copying the free space is contiguous: **bump-pointer** allocation, plus thread-local allocation buffers (TLABs) avoid locking.
- *Throughput vs latency vs footprint:* you can optimise only two — the "GC trilemma".
- *Is "Full GC" always the same thing?* Collector-dependent; for G1 it's the single fallback "Pause Full" that compacts everything.

**Remember:** *Sweep fragments, compact moves, copying evacuates; Serial/Parallel are STW, concurrent collectors need barriers.*

---

### Q132. Stop-the-world pauses and safepoints — what are they and why do pauses exceed GC time?

**Say this first:** "A safepoint is a moment when every Java thread is parked in a well-defined state, so the VM can do work that needs a consistent view — GC being the main one."

**Answer:** The JVM can't move objects or walk thread stacks while Java code is mutating them. To do so, it requests a **safepoint**:
1. VM thread sets a "stop" flag/poll page.
2. Each Java thread checks it at **safepoint polls** (method returns, loop back-edges, certain transitions) and blocks. Threads running native code (JNI) or blocked I/O are already "safe".
3. When *all* are stopped → VM operation runs (GC, deoptimisation, class redefinition, thread dump, heap dump...).
4. Threads resume.

**Total pause ≈ time-to-safepoint (TTSP) + operation time (+ resume).** A short GC may still show a long application pause if one thread takes long to *reach* the safepoint:
- a long counted loop (`for (int i...)`) with no poll, e.g. huge array copy/ intrinsic — mitigated in newer JDKs by *loop strip mining*;
- a thread descheduled by the OS or the container's CPU throttling;
- slow GC log writes, swapping, or page faults (e.g. transparent huge pages compaction).

**Diagnose:**
```bash
-Xlog:safepoint*:file=safepoint.log:time,uptime       # shows reason, TTSP ("Safepoint ... Total: ..ns, Reaching ...")
-Xlog:gc*,safepoint                                   # compare app-stopped time vs GC time
```
GC log lines like `Total time for which application threads were stopped` (older style) show real stop time.

**Not everything needs a global stop:** since JDK 10 **thread-local handshakes** let the VM run an operation on *one* thread (or each thread in turn) without stopping all. Modern low-latency collectors (ZGC, Shenandoah) keep global safepoints very short (root scanning only) and do the heavy work concurrently.

**Types of pauses to separate in conversation:**
- **GC pauses** — copying/compaction/marking-remark.
- **Non-GC safepoints** — deoptimisation, biased-lock revocation (feature removed in newer JDKs), class redefinition, `jstack`/`jcmd Thread.print` itself, heap dumps.
- **Non-JVM stalls** — OS scheduling, CPU throttling (cgroup quotas), noisy neighbours, GC thread starvation.

**Pitfalls / follow-up probes:**
- *Can `jstack` on a production JVM cause a pause?* Yes, a short safepoint; heap dumps are much longer.
- *Why did p99 spike with no GC?* Check safepoint log, CPU throttling (`container_cpu_cfs_throttled`), swap.
- *Is "concurrent" the same as no pause?* No — concurrent collectors still have short STW phases.

**Remember:** *Pause = time to reach safepoint + work; "concurrent" still has tiny pauses.*

---

### Q133. How does G1 work internally — regions, remembered sets, mixed collections, humongous objects?

**Say this first:** "G1 splits the heap into equal-sized regions, collects the ones with the most garbage first, and works to a pause-time goal."

**Answer — layout:**
```
Heap = many equal regions (1–32 MB, power of 2; picked from heap size, -XX:G1HeapRegionSize)
 [E][E][S][O][O][E][H][H][O][ ][E][O] ...
  E=Eden  S=Survivor  O=Old  H=Humongous  [ ]=free
```
Roles aren't fixed — a region can be Eden now, free later, Old afterwards. This makes it easy to pick *which* regions to collect.

**Cycle of work:**
1. **Young collection** (STW): evacuate live objects from all Eden+Survivor regions (copying) to new Survivor/Old regions. Parallel threads. G1 sizes the young generation dynamically to meet `-XX:MaxGCPauseMillis` (default 200 ms — a *goal*, not a guarantee).
2. **Concurrent marking cycle:** starts when old occupancy crosses the *initiating heap occupancy* (`IHOP`, 45% default, adaptive since JDK 9). Phases: *Initial Mark* (piggybacked on a young pause; log shows "Concurrent Start") → concurrent mark → *Remark* (short STW) → *Cleanup*. It computes how much live data each old region holds. Uses **SATB** barriers for correctness.
3. **Mixed collections:** after marking, the next several young GCs also include a selection of **old regions with the most garbage** ("garbage first") until reclaimable space falls below a threshold (`G1HeapWastePercent`, default 5).
4. **Full GC (fallback):** if G1 can't keep up (allocation outpaces concurrent work, "to-space exhausted/evacuation failure"), it falls back to a stop-the-world compacting Full GC (parallel since JDK 10). Seeing these = tuning problem.

**Remembered sets & card tables:** to collect a region without scanning the whole heap, each region tracks incoming references from *other* regions (its **remembered set**), maintained by a *post-write barrier* that marks "dirty cards" processed by concurrent refinement threads. This has a CPU/memory cost (a few % to 20% of the heap in bad cases).

**Humongous objects:** objects ≥ **half a region** are "humongous": allocated in contiguous region(s) directly in the old area, and only reclaimed at marking/cleanup (newer JDKs can reclaim some eagerly). Many large short-lived arrays (e.g. 8 MB buffers, big `StringBuilder`s, big JSON) can cause fragmentation and frequent concurrent cycles. Fix: bigger regions, smaller buffers, streaming.

**Useful flags (change sparingly):** `-XX:MaxGCPauseMillis`, `-XX:G1HeapRegionSize`, `-XX:InitiatingHeapOccupancyPercent`, `-XX:G1ReservePercent`, `-XX:ParallelGCThreads/ConcGCThreads`. Setting young-gen sizing flags (`-Xmn`, `NewRatio`) **disables** G1's adaptive sizing — usually a mistake.

**Pitfalls / follow-up probes:**
- *Why does a lower pause goal reduce throughput?* Smaller young sizes → more frequent GCs and more overhead.
- *"Pause Young (Mixed)" repeating for a long time?* Old garbage is high — ok if pauses are short.
- *"Pause Full (G1 Evacuation Pause)/Allocation Failure"* → raise heap, lower IHOP, reduce allocation rate or humongous objects.
- *G1 vs ZGC?* G1 trades some pause time for lower CPU/memory overhead.

**Remember:** *G1 = regions + pause goal; young → concurrent mark → mixed; Full GC means something's wrong.*

---

### Q134. ZGC and Shenandoah — how do ultra-low-pause collectors work? (colored pointers, load barriers, generational ZGC)

**Say this first:** "ZGC and Shenandoah do almost everything — including moving objects — *while the app runs*, so pauses stay in the low-millisecond or sub-millisecond range and don't grow with heap size."

**Answer — the hard problem:** compaction (moving objects) normally needs a stop-the-world, because every reference to a moved object must be fixed. Concurrent collectors need a trick so app threads never see a stale address.

**ZGC (production since JDK 15):**
- **Colored pointers:** metadata bits (e.g. "marked", "remapped") are stored *inside* the 64-bit reference itself, so GC state is tracked without touching object headers. (Because of this, ZGC doesn't use compressed oops.)
- **Load barriers:** tiny code the JIT inserts when the app *reads a reference from the heap*. If the pointer is stale (object was relocated), the barrier fixes it ("self-healing") before use.
- **Phases:** short STW (root scanning/marks start & end) + concurrent mark, concurrent relocate, concurrent remap. Pauses are typically **sub-millisecond** (often well below 1 ms), independent of heap size; heaps up to **16 TB**.
- **Generational ZGC:** JDK 21 added a generational mode (`-XX:+UseZGC -XX:+ZGenerational`) that collects young objects more often, lowering CPU and memory overhead (non-generational originally treated the heap as one). In later JDKs it became the default/only ZGC mode — check the JDK docs for your version.
- Needs spare headroom and more CPU than G1; give it enough memory so allocation doesn't outrun concurrent collection (**allocation stall** is its "Full GC" — look for it in logs).

**Shenandoah (OpenJDK builds since JDK 12; not in all vendors' builds):**
- Concurrent **evacuation (compaction)** done with **barriers** (load-reference barriers in recent versions) and forwarding pointers, so it works with compressed oops.
- Pause times short and mostly independent of heap size; region-based like G1. Generational mode was added in newer JDKs — verify status in your JDK docs.
- Flags: `-XX:+UseShenandoahGC`, heuristics (`adaptive`, `compact`, `static`).

**Trade-offs (be honest in an interview):**
| | G1 | ZGC / Shenandoah |
|---|---|---|
| Pauses | ~10–200 ms goal | ~1 ms or less |
| Throughput | Higher | Lower (barrier + concurrent threads' CPU) |
| Memory overhead | Lower | Higher (needs headroom) |
| Tuning | Pause goal, heap size | Mostly just heap size |

**Pitfalls / follow-up probes:**
- *When ZGC?* p99/p999 latency SLOs, large heaps (tens of GB+), tolerance for ~5–15% more CPU.
- *Are they "pauseless"?* No — very short pauses exist.
- *On a 1-CPU container?* Concurrent collectors need spare cores — use G1/Serial instead.
- *Barriers and JIT:* the cost is paid on every reference load, which is the throughput price.

**Remember:** *Colored pointers + load barriers = relocate objects while running; pick for latency, pay with CPU and headroom.*

---

### Q135. How do you read GC logs? What do you look for?

**Say this first:** "Turn on unified logging, then look at four things: pause time, frequency, heap-after-GC trend, and the GC *cause*."

**Answer — enable (Java 9+ unified logging):**
```bash
-Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags:filecount=5,filesize=20m
# Java 8:  -XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:gc.log
```
Overhead is tiny; run it in production always, with rotation.

**Sample G1 lines:**
```
[12.345s][info][gc] GC(41) Pause Young (Normal) (G1 Evacuation Pause) 612M->130M(1024M) 8.431ms
[40.100s][info][gc] GC(55) Pause Young (Concurrent Start) (G1 Humongous Allocation) 700M->420M(1024M) 15.2ms
[44.800s][info][gc] GC(60) Pause Young (Mixed) (G1 Evacuation Pause) 900M->350M(1024M) 21.7ms
[90.500s][info][gc] GC(90) Pause Full (Allocation Failure) 1022M->700M(1024M) 1830ms
```
Reading `612M->130M(1024M)`: used heap **before** → **after** GC (committed heap total). Last value = pause duration.

**What to check:**
1. **Pause durations** (p99/max) vs your SLO. Many 200 ms+ pauses → consider heap size, pause goal, or ZGC.
2. **Frequency:** young GC every 100 ms means a huge **allocation rate**. Allocation rate ≈ (before − previous after) / time between GCs.
3. **Heap *after* GC trend:** after-GC usage is the **live set**. A rising staircase = leak (see the leak diagnosis question). Flat = healthy.
4. **Any "Pause Full"** on G1/others, "to-space exhausted/evacuation failure", or ZGC "Allocation Stall" → capacity or tuning problem.
5. **GC cause** in parentheses: `Allocation Failure`/`G1 Evacuation Pause` (normal), `G1 Humongous Allocation` (big objects), `Metadata GC Threshold` (Metaspace growth → set `MetaspaceSize`/look for class generation), `System.gc()` (someone called it; `-XX:+DisableExplicitGC` or `ExplicitGCInvokesConcurrent`), `Ergonomics`.
6. **Promotion rate:** Young→Old growth per GC; high promotion fills Old and forces mixed/full GCs (objects living too long — caches, large per-request data).
7. **Time fields:** `User` vs `Real` and `Sys` in detailed logs: `Sys` high → OS (swap/THP); `Real > User/threads` → CPU starvation.
8. **GC overhead:** % of wall time spent in GC; over ~5–10% is concerning for latency-sensitive services.

**Tools:** read raw logs; quick graphs with GCViewer or similar visualisers (be careful uploading production logs to third-party sites); `jstat -gcutil <pid> 1000` for live counters; JFR for GC events.

**Pitfalls / follow-up probes:**
- *Why does the log show lower "after" than real usage?* It's post-collection; concurrent collectors have floating garbage.
- *High GC time but small heap?* Raise the heap or reduce allocation (find hot allocators with JFR/async-profiler alloc mode).
- *`-Xlog` decorators:* `uptime`, `time`, `tags` make correlation with app logs possible.

**Remember:** *Before→after(total) pause; watch frequency, after-GC floor, Full GCs and the cause.*

---

### Q136. Essential JVM flags — heap, containers, Metaspace, GC selection, diagnostics.

**Say this first:** "Know a short, deliberate set: size memory relative to the container, pick a collector, cap native areas, and make failures observable."

**Memory sizing:**
| Flag | Meaning |
|---|---|
| `-Xms` / `-Xmx` | Initial / max heap. Equal in prod to avoid resize pauses (but watch RSS in containers) |
| `-XX:MaxRAMPercentage=75` | Max heap as % of container/host RAM (default 25%). **Use this in containers** instead of hard-coded `-Xmx` |
| `-XX:InitialRAMPercentage` / `MinRAMPercentage` | Same idea for initial heap / very small memory (≤ ~256 MB) cases |
| `-Xss` | Per-thread stack size (default typically 1 MB on 64-bit Linux) |
| `-XX:MaxMetaspaceSize` | Cap class metadata (unlimited by default) |
| `-XX:MaxDirectMemorySize` | Cap NIO direct buffers (defaults to roughly the max heap size) |
| `-XX:ReservedCodeCacheSize` | JIT code cache |

**Container awareness:** `-XX:+UseContainerSupport` is on by default (Java 10+, backported to 8u191+). The JVM reads **cgroup** memory and CPU limits, so `MaxRAMPercentage` and CPU-based sizing (GC threads, `availableProcessors()`, ForkJoin parallelism) follow the limits. Pitfalls:
- Heap 75% + Metaspace + threads + direct memory + code cache must fit within the limit or the kernel kills the container (**OOMKilled**, exit 137) — *not* a Java `OutOfMemoryError`.
- CPU limit < 2 or memory < ~1.8 GB → JVM picks **Serial GC** unless you set one explicitly.
- Override CPU view if needed: `-XX:ActiveProcessorCount=N`.

**GC selection:** `-XX:+UseG1GC` (default), `-XX:+UseParallelGC`, `-XX:+UseZGC`, `-XX:+UseShenandoahGC`, `-XX:+UseSerialGC`; `-XX:MaxGCPauseMillis=…` for G1.

**Observability / failure handling:**
```
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps/
-XX:+ExitOnOutOfMemoryError            # or +CrashOnOutOfMemoryError: die so the orchestrator restarts
-Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=20m
-XX:NativeMemoryTracking=summary       # then jcmd <pid> VM.native_memory
-XX:StartFlightRecording=duration=60s,filename=rec.jfr   # JFR; see the tools question
-XX:+AlwaysPreTouch                    # commit heap at start: predictable latency, slower startup
-XX:+UseStringDeduplication            # G1 (and some others in newer JDKs): dedupe equal strings
```
**Inspect what's really in effect:**
```bash
java -XX:+PrintFlagsFinal -version | grep -E 'MaxHeapSize|UseG1GC|MaxRAMPercentage'
jcmd <pid> VM.flags            # flags of a running JVM
jcmd <pid> VM.command_line
```
**Where to put them:** `JAVA_TOOL_OPTIONS` env var (picked up by every JVM; logs a notice) or `JDK_JAVA_OPTIONS` (only for the `java` launcher) or the command line.

**Pitfalls / follow-up probes:**
- *Removed flags:* many Java 8 flags (PermSize, CMS flags, `-XX:+UseParNewGC`) are gone — JVM may refuse to start with unrecognised options (`-XX:+IgnoreUnrecognizedVMOptions` hides this, don't).
- *Cargo-cult tuning:* don't copy giant flag lists; defaults are good, each flag needs evidence from GC logs.
- *Experimental/diagnostic flags* need `-XX:+UnlockExperimentalVMOptions` / `-XX:+UnlockDiagnosticVMOptions`.

**Remember:** *Percentages for containers, cap native areas, pick a GC, dump on OOM, log GC.*

---

### Q137. How do you size the heap and tune for latency vs throughput?

**Say this first:** "Measure first: define the SLO, observe the live set and allocation rate from GC logs, then size the heap and pick the collector — don't guess flags."

**Answer — a repeatable method:**
1. **Set goals:** *latency* (p99 response, max pause), *throughput* (requests/s, batch duration), *footprint* (container cost). You can't maximise all three.
2. **Gather data under realistic load** (load test ≥ 30–60 min to include warm-up and mixed GCs): GC logs, JFR, response time percentiles.
3. **Find the live set** = heap *after* full/mixed GCs at steady state.
4. **Size the heap:** commonly **3–4× the live set** (rule of thumb, validate) so GC isn't constantly under pressure; more headroom → fewer collections and less promotion. A heap too big costs money and (for STW collectors) longer pauses; too small → frequent GCs or OOM.
5. **Set `-Xms = -Xmx`** (or use `MaxRAMPercentage` plus equal `InitialRAMPercentage`), leave 25–35% of container memory for non-heap.
6. **Check the allocation rate** and reduce it at source (cheapest "tuning"): avoid boxing, repeated string concatenation in loops, large temporary collections, per-request `ObjectMapper`/regex compile, unnecessary copies. Find hotspots with JFR/async-profiler allocation profiling.
7. **Choose the collector** by goal:
   - *Throughput/batch:* Parallel.
   - *Balanced service:* G1 (default). Tune only `MaxGCPauseMillis`, heap size, maybe region size/IHOP.
   - *Strict latency / big heap:* ZGC (or Shenandoah) with enough headroom and CPU.
   - *Tiny containers/short jobs:* Serial (lower overhead).
8. **Change one thing at a time**, re-measure, keep the config in version control with the reason.

**Latency vs throughput table:**
| Knob | Effect |
|---|---|
| Lower `MaxGCPauseMillis` (G1) | Shorter pauses, more GCs, less throughput |
| Bigger young gen / heap | Fewer GCs, less promotion (but longer individual pauses in STW collectors) |
| Concurrent collectors | Short pauses but more CPU and barrier cost |
| `AlwaysPreTouch`, equal Xms/Xmx | No resize/page-fault latency, slower start |

**Rules of thumb for what to avoid:**
- Setting young generation sizes manually with G1 (turns off adaptivity).
- Growing the heap to hide a leak.
- Ignoring non-heap memory in containers.
- Tuning without warm-up (tiered compilation changes allocation behaviour).

**Pitfalls / follow-up probes:**
- *Throughput metric:* % of time *not* in GC. Parallel targets 95%+ ("GC time ratio").
- *Why do latency spikes happen after deploy?* Cold JIT, caches, class loading — not GC.
- *Scale horizontally vs bigger heap?* Many small JVMs + G1 often beat one giant JVM unless you need a huge shared in-memory data set.

**Remember:** *Measure live set + allocation rate → heap ≈ 3–4× live set → pick GC by SLO → change one thing at a time.*

---

### Q138. Step-by-step: diagnosing a memory leak with heap dumps (jcmd/jmap, VisualVM, MAT).

**Say this first:** "Confirm it's a leak with GC logs, take heap dumps, then in MAT follow the dominator tree to the GC-root path that's holding the objects."

*(The OOM-type table is in the earlier OutOfMemoryError question; this is the hands-on procedure.)*

**Steps:**
1. **Confirm:** heap *after* GC keeps climbing (staircase) in GC logs/metrics until OOM or restart. Plateau = no leak, just a small heap.
2. **Quick histogram (cheap, no full dump):**
```bash
jcmd <pid> GC.class_histogram | head -30        # instance counts & bytes per class
```
   Do it twice, minutes apart; the class whose count keeps growing is your suspect (`byte[]`, `String`, `HashMap$Node`, your own class).
3. **Capture a heap dump** (do it when the leak is visible, ideally on one instance taken out of rotation):
```bash
jcmd <pid> GC.heap_dump /dumps/heap.hprof       # path is on the target JVM's filesystem; default = live objects (forces a GC)
jmap -dump:live,format=b,file=heap.hprof <pid>  # alternative
# automatic:  -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps
```
   Warning: it **pauses the JVM** (seconds to minutes for big heaps), needs disk about the heap size, and the file contains **sensitive data** (passwords, tokens, PII) — protect and delete.
4. **Open in Eclipse MAT** (needs RAM ≈ dump size; use `ParseHeapDump.sh` for headless parsing) or VisualVM/JProfiler/YourKit:
   - **Leak Suspects report** → a quick first hint.
   - **Histogram** → classes by count/*shallow heap*.
   - **Dominator tree** → objects ranked by **retained heap** (memory freed if that object disappeared). The top entries with huge retained size are the holders (a static map, a cache, a thread's `ThreadLocalMap`, a class loader).
   - On a suspect: **Path to GC Roots → exclude all phantom/weak/soft references** — shows *who* strongly keeps it alive. The path ends at a GC root such as `static field`, `Thread`, or `class loader`.
   - **List objects → with outgoing references** to inspect contents (e.g. which keys are in the map), and **OQL** queries for ad-hoc analysis.
5. **Compare two dumps** (MAT *Compare Basket*/histogram diff) taken some time apart to see what grew.
6. **Fix** the cause: bound the cache (size + TTL), clear `ThreadLocal`, deregister listeners, close resources, fix classloader refs.
7. **Verify** with a soak test: heap-after-GC should be flat.

**Shallow vs retained heap:** shallow = the object's own bytes; retained = shallow + everything only reachable through it. Dominator tree uses retained size, which is why a tiny `HashMap` object can "dominate" 1 GB.

**Other leak types:** `Metaspace` → `jcmd <pid> VM.classloader_stats`, `GC.class_stats` (diagnostic) for loader leaks; native/direct memory → NMT (see the native memory question).

**Pitfalls / follow-up probes:**
- *Dump in production without downtime?* Use one instance behind a load balancer, or rely on OOM dump; consider JFR *old object sample* events (`OldObjectSample`) which record leak candidates with low overhead.
- *Dump looks fine but leak exists?* Dumped only live objects after a GC — right; ensure you dumped at a high-water time.
- *Common culprits:* static maps, unbounded caches, listeners, `ThreadLocal`, unclosed streams, class loaders.

**Remember:** *Histogram → heap dump → MAT dominator tree → path to GC roots → fix → soak test.*

---

### Q139. Reading a thread dump — states, locks, deadlocks and high CPU.

**Say this first:** "A thread dump is a snapshot of every thread's stack and state. Take three, 5–10 seconds apart, and compare what's stuck, what's spinning, and who holds which lock."

**Take one:**
```bash
jcmd <pid> Thread.print -l        # preferred; -l includes java.util.concurrent lock info
jstack -l <pid>                    # same idea
kill -3 <pid>                      # Unix: SIGQUIT writes the dump to the process's stdout
jcmd <pid> Thread.dump_to_file -format=json out.json   # JDK 21: includes virtual threads
```
**Anatomy of one thread:**
```
"http-nio-8080-exec-7" #52 prio=5 os_prio=0 tid=0x...  nid=0x1f4c waiting for monitor entry
   java.lang.Thread.State: BLOCKED (on object monitor)
	at com.acme.Cache.get(Cache.java:42)
	- waiting to lock <0x000000076ab2c0d8> (a java.util.HashMap)
	at com.acme.OrderService.find(OrderService.java:88)
```
**Thread states and what they usually mean:**
| State | Meaning | Look for |
|---|---|---|
| `RUNNABLE` | Running **or** ready, or inside native code (e.g. socket read) | Many threads in the same hot method = CPU burn. A `RUNNABLE` thread in `SocketInputStream.read` is actually waiting on I/O |
| `BLOCKED` | Waiting to enter a `synchronized` block/method (monitor held by someone else) | Find `- locked <0x…>` in the owner thread — lock contention |
| `WAITING` | Waiting indefinitely: `Object.wait()`, `Thread.join()`, `LockSupport.park` (j.u.c locks, queues, pool idle workers) | Idle pool workers are normal; "parking to wait for <…>" shows the lock/queue |
| `TIMED_WAITING` | Same with timeout: `sleep`, `wait(ms)`, `parkNanos` | Normal for schedulers/pollers |
| `NEW` / `TERMINATED` | Not started / finished | Rare in dumps |

**Deadlock detection:** jstack/jcmd print at the bottom:
```
Found one Java-level deadlock:
"T1": waiting to lock monitor 0x...(object 0x..., a Foo), which is held by "T2"
"T2": waiting to lock monitor 0x...(object 0x..., a Bar), which is held by "T1"
```
`-l` also detects deadlocks involving `ReentrantLock`. Programmatically: `ThreadMXBean.findDeadlockedThreads()`. Fix by consistent lock ordering/timeouts (see the guide's deadlock question).

**Patterns to recognise:**
- **All request threads BLOCKED on one lock** → contention on a single monitor (a cache, logger, `Collections.synchronizedMap`).
- **All pool threads WAITING on the same queue/future, nobody RUNNABLE** → pool starvation (tasks waiting on tasks in the same pool).
- **Many threads RUNNABLE in `HashMap.get`/regex** → infinite loop/CPU hog (old non-thread-safe `HashMap` bugs) or catastrophic regex backtracking.
- **Waiting on connection pool** (`HikariPool.getConnection`) → pool exhausted or DB slow.
- Thread count growing across dumps → thread leak.

**High CPU recipe (Linux):**
```bash
top -H -p <pid>            # find hot thread id (decimal)
printf '%x\n' <tid>         # convert to hex → match "nid=0x…" in the dump
```
Then read that thread's stack in 3 dumps. For a sampled view over time use JFR or async-profiler.

**Pitfalls / follow-up probes:**
- *A single dump can mislead* — always compare several.
- *Does `jstack` stop the JVM?* It forces a safepoint, short but not free.
- *Virtual threads:* not in plain `Thread.print`; use the JSON dump format.
- *Daemon vs user threads* and thread names: name your pools, it makes dumps readable.

**Remember:** *BLOCKED = waiting for a monitor, WAITING = parked/waiting, RUNNABLE ≠ always CPU; take 3 dumps, find the lock owner.*

---

### Q140. Which diagnostic tool when — JFR, async-profiler, jcmd, jstat, jconsole/VisualVM?

**Say this first:** "`jcmd` is the Swiss-army entry point, JFR is the always-on flight recorder, async-profiler gives accurate flame graphs, and the rest are quick lookups."

| Tool | What it's for | Overhead / notes |
|---|---|---|
| **`jcmd <pid> <cmd>`** | One CLI for most tasks: `help`, `VM.flags`, `Thread.print`, `GC.heap_info`, `GC.class_histogram`, `GC.heap_dump`, `JFR.start`, `VM.native_memory`, `Compiler.codecache` | Preferred over the older jmap/jstack/jinfo |
| **`jstat -gcutil <pid> 1000`** | Live GC counters (Eden/Old %, GC count/time) every second | Very low; quick sanity check |
| **`jstack`**, **`jmap`**, **`jinfo`** | Thread dump / heap dump & histogram / view-set flags | Older; `jcmd` covers them |
| **JFR (Java Flight Recorder)** | Low-overhead event recorder built in the JVM: GC, JIT, locks, I/O, allocation samples, exceptions, CPU samples, thread states | Typically ~1–2% with default profile; safe for production |
| **JDK Mission Control (JMC)** | GUI to analyse `.jfr` files | Separate download |
| **async-profiler** | Sampling profiler (CPU, wall-clock, allocation, lock) giving **flame graphs**; avoids *safepoint bias*, shows native + Java frames | Open source, external agent; good for "what is burning CPU?" |
| **jconsole / VisualVM** | GUI for MBeans, heap/thread charts, sampling, heap dump, simple profiling | Dev/staging; VisualVM is a separate download; JMX remote needs care (security) |
| **Eclipse MAT** | Heap dump analysis | Offline; see the leak question |
| **JMH** | Microbenchmarks | Not a production tool |

**JFR in practice:**
```bash
java -XX:StartFlightRecording=duration=120s,filename=rec.jfr,settings=profile -jar app.jar
jcmd <pid> JFR.start name=r1 settings=profile duration=60s filename=/tmp/r1.jfr   # on a running JVM
jcmd <pid> JFR.dump name=r1 filename=/dumps/r1.jfr
jfr print --events jdk.GCPhasePause rec.jfr            # CLI summary/print/ flame
```
JFR was open-sourced in JDK 11 (and backported to later Java 8 updates). Always-on, rolling recording (`maxage=1h`) is a great production practice: when an incident happens, dump the last hour.

**Picking the tool:**
- *High CPU?* async-profiler `cpu`/JFR execution samples + thread dump (hot thread).
- *High latency but low CPU?* wall-clock profiling, thread dumps (blocked/waiting), JFR lock/IO events.
- *GC pauses?* GC logs, `jstat`, JFR GC events.
- *Heap growth?* histogram → heap dump → MAT.
- *Allocation churn?* JFR allocation events or async-profiler `alloc`.
- *Off-heap/RSS growth?* NMT, `pmap`.
- *Deadlock/hang?* thread dumps.

**Pitfalls / follow-up probes:**
- *Safepoint bias:* profilers that sample only at safepoints (some JVMTI-based ones) mis-attribute hot spots; JFR and async-profiler avoid this.
- *Security:* JMX/attach exposes control of the JVM; restrict access, never open JMX without auth/TLS.
- *Same user:* `jcmd` must run as the same OS user as the JVM; in containers use `kubectl exec`/ephemeral containers.
- *Overhead check:* any profiler on production: start short, watch latency.

**Remember:** *jcmd to poke, JFR to record, async-profiler to see CPU, jstat to watch GC, MAT to dissect heap.*

---

### Q141. "My JVM uses more memory than `-Xmx`" — native memory, direct buffers, off-heap leaks and NMT.

**Say this first:** "`-Xmx` limits only the *Java heap*. The process footprint (RSS) also includes Metaspace, thread stacks, code cache, direct buffers, GC structures and native allocations."

**Answer — what adds up to the process memory:**
```
RSS ≈ Heap (Xmx) + Metaspace & class space + Thread stacks (N × Xss)
      + Code cache (JIT) + Direct/mapped buffers + GC data structures
      + JIT compiler arenas + symbol/string tables + native libs (malloc/JNI)
      + glibc malloc fragmentation / arenas + CDS archive
```
| Area | What grows it | Control |
|---|---|---|
| Metaspace | Many classes / loader leaks / proxies | `-XX:MaxMetaspaceSize` |
| Thread stacks | Thread count × `-Xss` (default ~1 MB reserved; touched pages count) | Bound pools; use virtual threads |
| Code cache | JIT output | `-XX:ReservedCodeCacheSize` |
| Direct buffers | `ByteBuffer.allocateDirect`, Netty pooled arenas, NIO channels | `-XX:MaxDirectMemorySize`; release buffers |
| GC overhead | Remembered sets (G1), mark bitmaps, ZGC multi-mapping | Choose collector wisely, headroom |
| Native/JNI libs | Compression (zlib), SSL, DB drivers, image libs, RocksDB | App-level |
| malloc arenas | Many threads → fragmented per-thread arenas in glibc | `MALLOC_ARENA_MAX=2` (commonly used), or jemalloc |

**Direct buffers in detail:** `allocateDirect` memory is off-heap but freed only when the small `DirectByteBuffer` heap object is garbage-collected (via a `Cleaner`). If the heap is quiet, GC runs rarely, and off-heap usage balloons; the JVM triggers `System.gc()` internally when the limit is hit (so `-XX:+DisableExplicitGC` can backfire → `OutOfMemoryError: Direct buffer memory`). Netty leaks come from unreleased `ByteBuf` reference counts — enable leak detection (`-Dio.netty.leakDetection.level=advanced` in testing).

**Native Memory Tracking (NMT):**
```bash
java -XX:NativeMemoryTracking=summary -jar app.jar        # small overhead (detail mode = more)
jcmd <pid> VM.native_memory baseline                       # take a baseline
# ... run for a while ...
jcmd <pid> VM.native_memory summary.diff                   # what grew: Java Heap, Class, Thread, Code, GC, Internal...
```
NMT tracks **JVM-internal** allocations only — *not* memory allocated by third-party native libraries via `malloc`. If NMT's total is far below RSS, suspect native libs or malloc fragmentation — use `pmap -x <pid>`, `/proc/<pid>/smaps`, `jemalloc`/`tcmalloc` profiling, or `perf`.

**Triage checklist (container OOMKilled but no Java OOM):**
1. Compare `container_memory_working_set` with the heap used → the gap = non-heap.
2. Check `jcmd <pid> VM.native_memory summary` for what dominates.
3. Check thread count, direct buffer usage (`BufferPoolMXBean`, JMX `java.nio:type=BufferPool`), Metaspace.
4. Lower heap % (`MaxRAMPercentage=60–70`) to leave headroom; cap Metaspace/direct memory.
5. Fix the actual leak (unclosed buffers/streams, thread leak, native lib).

**Pitfalls / follow-up probes:**
- *Does heap shrink and return memory to the OS?* G1 and others can uncommit, but slowly/conservatively; `Xms=Xmx` never shrinks below.
- *Is RSS growth always a leak?* No — warm-up (JIT, class loading, TLABs, arenas) raises it to a plateau.
- *`-Xmx` equals container limit?* Guaranteed OOMKill eventually. Never.
- *Virtual memory vs RSS:* reserved ≠ resident; compare RSS.

**Remember:** *Xmx covers only the heap; account for Metaspace, stacks, code cache, direct buffers and native libs — NMT to see JVM's share.*

---

### Q142. Startup optimisation — Class Data Sharing/AppCDS and GraalVM Native Image vs the JVM.

**Say this first:** "The JVM is optimised for peak throughput after warm-up. For fast startup and low footprint you can *pre-process* work: share class data (CDS/AppCDS), or compile ahead-of-time to a native binary (GraalVM Native Image)."

**Why startup is slow:** loading and verifying thousands of classes, interpreting cold code, JIT warm-up, and (in frameworks) reflection/classpath scanning at boot.

**Class Data Sharing (CDS) and AppCDS:**
- CDS maps a pre-parsed, pre-verified **archive of classes** into memory instead of loading from jars; shared read-only across JVMs on the host (saves startup *and* memory).
- A default archive for JDK classes is enabled by default since JDK 12.
- **AppCDS** adds *your* classes (and libraries):
```bash
# JDK 13+: record at exit, then reuse
java -XX:ArchiveClassesAtExit=app.jsa -jar app.jar        # run a typical workload, then exit
java -XX:SharedArchiveFile=app.jsa -jar app.jar           # start faster
# JDK 19+: -XX:+AutoCreateSharedArchive -XX:SharedArchiveFile=app.jsa  (creates/updates automatically)
```
- Typical gain: tens of percent faster startup for class-heavy apps. Bake the archive into the container image. The archive must match the exact JDK build and classpath.
- Other startup knobs: `-XX:TieredStopAtLevel=1` (C1 only, fast short jobs), `-Xshare:auto`, trim classpath, avoid heavy static initialisation, `jlink` for a smaller runtime.
- **Leyden** (OpenJDK project) aims to extend this with ahead-of-time caching of class loading/linking and profiles; parts have appeared in recent JDKs — check the JDK docs for status in your version.
- **CRaC** (Coordinated Restore at Checkpoint) restores a pre-warmed JVM snapshot — available in some distributions; check vendor docs.

**GraalVM Native Image (AOT):** compiles your app and the needed JDK parts into a standalone executable using **closed-world analysis** (only code reachable at build time is included).

| | JVM (HotSpot) | Native Image |
|---|---|---|
| Startup | Seconds | Milliseconds |
| Memory (RSS) | Larger (JIT, metaspace, etc.) | Much smaller |
| Peak throughput | Usually higher (profile-guided JIT) | Often lower unless PGO is used |
| Warm-up | Needed | None |
| Build time | Fast | Slow (minutes), high build RAM |
| Dynamic features (reflection, proxies, resources, JNI, serialization) | Work freely | Need **configuration/reachability metadata**; some unsupported |
| Dynamic class loading / agents | Yes | Largely no |
| Debug/observability | Mature (JFR, jcmd, agents) | More limited (check current docs) |
| GC options | All | Fewer (Serial default; G1 in some editions) |
| Portability | Same `.class` everywhere | One binary per OS/arch |

**When to pick which:** serverless/CLI tools/scale-to-zero microservices → native; long-running, throughput-sensitive services needing full JVM tooling → JVM (+ AppCDS). Test both; don't assume native is faster overall.

**Pitfalls / follow-up probes:**
- *Why does reflection break in native image?* Closed-world: classes accessed only reflectively aren't seen unless declared.
- *"Write once run anywhere"* is lost for native binaries.
- *CDS archive mismatch:* JVM silently ignores or warns, then loads normally.
- *Startup vs steady-state:* AppCDS doesn't change peak performance.

**Remember:** *CDS shares pre-processed classes; Native Image trades peak speed and dynamism for instant start and tiny memory.*

---

### Q143. Project Loom, Panama, Valhalla, Leyden — what are they? (awareness level)

**Say this first:** "These are OpenJDK projects shaping future Java: Loom = lightweight threads, Panama = native interop, Valhalla = value objects, Leyden = startup/footprint. Feature status changes per release — always check the JDK docs/JEPs for your version."

| Project | Goal | Status (verify for your JDK) |
|---|---|---|
| **Loom** | Cheap **virtual threads**, plus structured concurrency and scoped values | **Virtual threads are final in JDK 21** (JEP 444). Structured concurrency/scoped values were preview in 21 — check later JDKs |
| **Panama** | Safe, fast access to native code and off-heap memory: **Foreign Function & Memory API** (replacing JNI, `Unsafe` off-heap usage) and the **Vector API** | FFM API became final in JDK 22 (JEP 454); Vector API has remained incubating — check docs |
| **Valhalla** | **Value classes** (objects without identity, flattened in memory) and primitive-like generics to cut memory and pointer-chasing | In progress with preview work — check docs; not something to rely on in production yet |
| **Leyden** | Faster startup, warm-up and smaller footprint via ahead-of-time caching and condensed runtime state | Pieces arriving incrementally — check docs |
| **Amber** (related) | Language features: records, sealed classes, pattern matching, text blocks | Many final since Java 16–21 |

**Loom in plain words (the one most asked):**
- A **virtual thread** is a lightweight thread managed by the JVM, scheduled onto a small pool of OS **carrier threads**. When it blocks on I/O or a lock, it *unmounts* from the carrier, which then runs another virtual thread. You can have millions of them.
- Why: classic thread-per-request code (simple, blocking style) stops being limited by the ~thousands of OS threads, without switching to reactive code.
```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 10_000; i++) executor.submit(() -> callRemoteService());
}
Thread.ofVirtual().start(() -> work());
```
- Good for **I/O-bound** workloads, not CPU-bound (no extra cores).
- **Pitfalls:** *pinning* — a virtual thread inside a `synchronized` block that blocks could pin its carrier (relaxed in recent JDKs — check docs); don't pool virtual threads; `ThreadLocal` per virtual thread can add memory; limit concurrency to protected resources (use a `Semaphore`, not pool size); thread dumps need the JSON format.

**Panama in plain words:** call C libraries and manage off-heap memory with a safer, deterministic API (`Arena`, `MemorySegment`, `Linker`) instead of JNI boilerplate or `sun.misc.Unsafe`.

**Valhalla in plain words:** today a `List<Point>` is an array of pointers to separate objects with headers. Value classes would let the JVM store the data flat, improving cache locality and memory.

**Pitfalls / follow-up probes:**
- *Does Loom replace reactive programming?* For many I/O-bound services, blocking code on virtual threads is simpler and good enough; reactive still has uses (backpressure streams).
- *Do virtual threads speed up CPU work?* No.
- *Preview/incubator features* need `--enable-preview` and aren't for production without reviewing support policy.
- Don't claim a feature is final unless you're sure of the JDK version.

**Remember:** *Loom = virtual threads (final in 21), Panama = native/off-heap API, Valhalla = value objects, Leyden = faster startup; say "check the JDK docs" for status.*

---

# 6. Multithreading & Concurrency

### Q144. Thread lifecycle and ways to create threads.

**States:** `NEW → RUNNABLE → (BLOCKED | WAITING | TIMED_WAITING) → TERMINATED`.
- `BLOCKED` — waiting for a monitor lock.
- `WAITING` — `wait()`, `join()`, `LockSupport.park()`.
- `TIMED_WAITING` — `sleep(n)`, `wait(n)`, `await(n)`.

**Creation (worst → best):**
1. `extends Thread` — burns your one inheritance slot.
2. `implements Runnable` + `new Thread(r).start()`.
3. `Callable<V>` + `ExecutorService.submit()` → `Future<V>`.
4. `CompletableFuture.supplyAsync(sup, executor)` — composable async.
5. `Executors.newVirtualThreadPerTaskExecutor()` (Java 21) for IO-bound work.

**Never** call `run()` directly — that executes on the current thread. `start()` can only be called once (`IllegalThreadStateException`).

**Reading the states in a real thread dump:** many threads `BLOCKED` on the same lock → lock contention (someone holds a `synchronized` block too long). Many `WAITING` in `parking to wait for <...>` inside a pool → idle workers or exhausted DB connection pool (they're waiting for a connection). `RUNNABLE` threads stuck at socket read are still "runnable" to the JVM even though they're waiting on the network.

```java
ExecutorService pool = Executors.newFixedThreadPool(4);          // illustration only — see Q147 for sizing/safety
Future<Integer> f = pool.submit(() -> compute());                // Callable returns a result / can throw
Integer result = f.get(2, TimeUnit.SECONDS);                     // blocks with a timeout
pool.shutdown();                                                 // always shut pools down
```

**Remember:** *Runnable over Thread, Executor over Runnable, CompletableFuture over Future.*

---

### Q145. `synchronized` vs `volatile` vs `Atomic*` vs `Lock`.

| Tool | Guarantees | Cost | Use for |
|---|---|---|---|
| `volatile` | **Visibility + ordering**, NOT atomicity | Cheap (memory barrier) | Flags: `volatile boolean running` |
| `synchronized` | Visibility + **atomicity** (mutual exclusion) | Medium (biased/thin/fat lock) | Compound state updates |
| `AtomicInteger/Reference` | Lock-free atomic (CAS) | Cheap under low contention | Counters, CAS state machines |
| `ReentrantLock` | Like synchronized + `tryLock`, timeout, fairness, interruptible, multiple `Condition`s | Medium | Advanced locking |
| `ReadWriteLock` / `StampedLock` | Many readers, one writer | — | Read-heavy shared structure |
| `LongAdder` | Striped counter | Best under high contention | Metrics/counters |

**Key point:** `volatile int count; count++;` is **still broken** — `++` is read-modify-write, three operations. Use `AtomicInteger.incrementAndGet()`.

**Why each exists:** threads cache variables in CPU registers/caches, so one thread's write may not be seen by another (*visibility*), and compound operations like check-then-act can interleave (*atomicity*). `volatile` fixes only visibility; `synchronized`/`Lock` fix both by letting one thread at a time in; **CAS** (*compare-and-set*) — used by `Atomic*` — says "set to new only if the current value is still what I read; otherwise retry", giving atomicity with no blocking.

```java
class Stopper { private volatile boolean running = true;           // one writer, many readers → volatile is enough
    void stop() { running = false; }  void loop() { while (running) work(); } }

class Counter { private final AtomicLong hits = new AtomicLong();  // many writers → atomic
    void hit() { hits.incrementAndGet(); } }

class Account { private long balance;                              // two fields/invariants → need a lock
    synchronized void transfer(Account to, long amt) { /* check + debit + credit as one unit */ } }
```
**Probe:** "Why `ReentrantLock` over `synchronized`?" — when you need `tryLock` with a timeout, interruptible waits, fairness, or several wait-conditions; and always unlock in `finally`.

**Remember:** *volatile = visibility, synchronized = atomicity, Atomic = both without a lock.*

---

### Q146. Explain the Java Memory Model and `happens-before`.

Without synchronization, the compiler/CPU may reorder instructions and each thread may read stale cached values. The JMM defines **happens-before** edges that guarantee visibility:

1. **Program order** within a thread.
2. **Monitor lock**: unlock of M happens-before any subsequent lock of M.
3. **Volatile**: a write to a volatile happens-before every later read of it.
4. **Thread start**: `t.start()` happens-before everything in t.
5. **Thread join**: everything in t happens-before `t.join()` returning.
6. **Final fields**: correctly constructed final fields are visible without sync.
7. Transitivity: if A hb B and B hb C then A hb C.

**In plain words:** "A *happens-before* B" means *everything A did (all its writes) is guaranteed to be visible to B*. Without such an edge a thread may legally see stale or reordered values — not a bug in the JVM, just what hardware caches and optimisers are allowed to do.

```java
int data = 0; boolean ready = false;          // shared, NOT volatile
// Thread 1: data = 42; ready = true;
// Thread 2: while (!ready) {}  print(data);  // may loop forever, or print 0 — no hb edge
```
Make `ready` `volatile` and it works: the write to `data` precedes the volatile write (program order), the volatile write hb the volatile read, so Thread 2 is guaranteed to see `42` (transitivity). This is exactly why safely publishing an object via a `volatile` field, a `synchronized` block, a `ConcurrentHashMap` or `final` fields is safe, while handing it over through a plain field is not.

**Double-checked locking done right:**
```java
private volatile Config config;              // volatile is MANDATORY
public Config get() {
    Config c = config;
    if (c == null) {
        synchronized (this) {
            c = config;
            if (c == null) config = c = load();
        }
    }
    return c;
}
```
Without `volatile`, another thread can see a **partially constructed** object because the constructor writes may be reordered after the reference assignment.

**Remember:** *No happens-before edge = no visibility guarantee, no matter how "obvious" the code looks.*

---

### Q147. Explain `ThreadPoolExecutor` parameters and why `Executors.newFixedThreadPool` is risky.

```java
new ThreadPoolExecutor(
    corePoolSize,        // always-alive threads
    maximumPoolSize,     // upper bound
    keepAliveTime, unit, // idle timeout for threads above core
    workQueue,           // BlockingQueue
    threadFactory,       // NAME YOUR THREADS - vital for thread dumps
    rejectedExecutionHandler
);
```

**Task admission order (crucial):**
1. threads < core → **create a new thread**.
2. else → **queue the task**.
3. queue full and threads < max → create a thread.
4. queue full and threads == max → **reject**.

So with an **unbounded queue** (`LinkedBlockingQueue` with no capacity — what `newFixedThreadPool` uses) `maximumPoolSize` is **never reached** and the queue grows until OOM. And `newCachedThreadPool` uses `SynchronousQueue` + `Integer.MAX_VALUE` max threads → unbounded thread creation.

**Rejection policies:** `AbortPolicy` (default, throws), `CallerRunsPolicy` (backpressure — my usual choice), `DiscardPolicy`, `DiscardOldestPolicy`.

**Sizing:** CPU-bound → `cores + 1`. IO-bound → `cores × (1 + waitTime/serviceTime)`. Or just use virtual threads for IO.

**Spring version:**
```java
@Bean("emailExecutor")
public ThreadPoolTaskExecutor emailExecutor() {
    var ex = new ThreadPoolTaskExecutor();
    ex.setCorePoolSize(8); ex.setMaxPoolSize(16); ex.setQueueCapacity(500);
    ex.setThreadNamePrefix("email-");
    ex.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    ex.setWaitForTasksToCompleteOnShutdown(true);
    ex.setAwaitTerminationSeconds(30);          // graceful shutdown
    return ex;
}
```

**Remember:** *Bounded queue + bounded pool + named threads + CallerRunsPolicy = production-safe executor.*

---

### Q148. `Future` vs `CompletableFuture`. Show a real orchestration.

`Future` is blocking (`get()`), has no chaining, no exception composition, no manual completion.
`CompletableFuture` implements `CompletionStage`: non-blocking composition, combination, and error handling.

```java
CompletableFuture<User>    user   = supplyAsync(() -> userClient.get(id), executor);
CompletableFuture<List<Order>> orders = supplyAsync(() -> orderClient.get(id), executor);

CompletableFuture<Profile> profile = user
    .thenCombine(orders, Profile::new)                     // join two
    .orTimeout(2, TimeUnit.SECONDS)                        // Java 9
    .exceptionally(ex -> Profile.fallback(id))             // recover
    .whenComplete((res, ex) -> log.info("done {}", id));

// fan-out over a list
List<CompletableFuture<Item>> futures = ids.stream()
        .map(i -> supplyAsync(() -> client.fetch(i), executor)).toList();
List<Item> items = CompletableFuture.allOf(futures.toArray(CompletableFuture[]::new))
        .thenApply(v -> futures.stream().map(CompletableFuture::join).toList()).join();
```

| Method | Meaning |
|---|---|
| `thenApply` | map (sync, returns value) |
| `thenCompose` | flatMap (returns another CF — avoids nesting) |
| `thenCombine` | zip two independent futures |
| `thenAccept` / `thenRun` | consume / side effect |
| `allOf` / `anyOf` | wait for all / first |
| `exceptionally` / `handle` / `whenComplete` | recover / transform both / peek |

**Always pass your own executor** — the default is the common ForkJoinPool.

**Remember:** *thenApply = map, thenCompose = flatMap, thenCombine = zip; always supply an executor.*

---

### Q149. What is a deadlock? How do you detect and prevent it?

**Four Coffman conditions:** mutual exclusion, hold-and-wait, no preemption, circular wait. Break any one.

**Prevention:**
1. **Global lock ordering** — always acquire locks in the same order (e.g. sort by account id).
2. **`tryLock(timeout)`** — back off and retry.
3. Reduce lock scope; prefer immutability and single-threaded ownership.
4. Never call foreign/unknown code while holding a lock.
5. Use higher-level constructs (`ConcurrentHashMap`, atomics, queues) instead of hand-rolled locking.

**Detection:** `jstack <pid>` prints `Found one Java-level deadlock:` with the cycle. Also `jcmd <pid> Thread.print`, JConsole's "Detect deadlock", or `ThreadMXBean.findDeadlockedThreads()` exposed as a health check.

**Related terms:** **livelock** (threads keep responding to each other, no progress), **starvation** (a thread never gets the lock), **race condition** (result depends on timing).

**Classic example & fix:** two transfers run at once — `transfer(A→B)` locks A then B, while `transfer(B→A)` locks B then A. Each holds one lock and waits for the other forever.
```java
void transfer(Account from, Account to, long amt) {
    Account first = from.id() < to.id() ? from : to;      // global ordering by id
    Account second = first == from ? to : from;
    synchronized (first) { synchronized (second) { from.debit(amt); to.credit(amt); } }
}
```
**DB variant:** two transactions updating the same rows in opposite order deadlock in the database; the DB kills one (`deadlock detected`). Fix with consistent row-access order (`ORDER BY id ... FOR UPDATE`) and short transactions, and retry the victim.

**Remember:** *Same lock order every time; jstack finds the cycle.*

---

### Q150. `wait/notify` vs `CountDownLatch` vs `CyclicBarrier` vs `Semaphore` vs `Phaser`.

| Construct | Purpose | Reusable |
|---|---|---|
| `wait/notify/notifyAll` | Low-level condition wait (must be inside `synchronized`, always in a `while` loop to guard spurious wakeups) | Yes |
| `CountDownLatch(n)` | Wait for n events once (`await()` / `countDown()`) | **No** |
| `CyclicBarrier(n)` | n threads meet repeatedly at a barrier | Yes |
| `Semaphore(n)` | Limit concurrent access to n permits (rate limiting, bulkhead) | Yes |
| `Exchanger` | Swap objects between two threads | Yes |
| `Phaser` | Dynamic party count, multi-phase | Yes |

**`sleep` vs `wait`:** `sleep` is on `Thread`, keeps the lock, time-based. `wait` is on `Object`, **releases the monitor**, needs a `notify`.

**Examples of when to use which:**
```java
// CountDownLatch — main waits for 3 startup checks to finish (one-shot)
CountDownLatch ready = new CountDownLatch(3);
checks.forEach(c -> pool.submit(() -> { c.run(); ready.countDown(); }));
ready.await(10, TimeUnit.SECONDS);

// Semaphore — allow at most 5 concurrent calls to a fragile downstream (simple bulkhead)
Semaphore permits = new Semaphore(5);
permits.acquire(); try { callDownstream(); } finally { permits.release(); }
```
- **CyclicBarrier** — N worker threads each compute a partition, all wait at the barrier, then proceed to the next round together (e.g. iterative simulations); it resets automatically.
- **wait/notify** is rarely hand-written today; prefer `BlockingQueue` (producer–consumer) or `Condition`. If you do use it, loop `while (!condition) wait();` because wakeups can be spurious.

**Remember:** *Latch = one-shot gate, Barrier = reusable meeting point, Semaphore = permit counter.*

---

### Q151. `ThreadLocal` — where is it used and what's the danger?

**Used by Spring for:** `RequestContextHolder`, `SecurityContextHolder`, `TransactionSynchronizationManager` (the current `EntityManager`/`Connection`), `LocaleContextHolder`, MDC in SLF4J (correlation IDs).

**Danger:** with pooled threads the value persists into the **next request** → data leakage between users and memory leaks. Always clean up:
```java
public class TenantFilter implements Filter {
    public void doFilter(...) {
        TenantContext.set(req.getHeader("X-Tenant"));
        try { chain.doFilter(req, res); }
        finally { TenantContext.clear(); }        // MANDATORY
    }
}
```
Also: `ThreadLocal` does **not** propagate to child threads (use `InheritableThreadLocal`, or `TaskDecorator` in Spring, or Micrometer `ContextPropagation`) — a very common bug when moving to `@Async`, where `SecurityContext` and MDC vanish.

```java
// propagate MDC + SecurityContext into @Async threads
executor.setTaskDecorator(runnable -> {
    var ctx = SecurityContextHolder.getContext();
    var mdc = MDC.getCopyOfContextMap();
    return () -> {
        try { SecurityContextHolder.setContext(ctx); if (mdc != null) MDC.setContextMap(mdc); runnable.run(); }
        finally { SecurityContextHolder.clearContext(); MDC.clear(); }
    };
});
```

**Remember:** *ThreadLocal + thread pool = leak, unless you clear it in a finally.*

---

### Q152. How do you make a Singleton thread-safe? Which approach is best?

1. **Eager static final** — simple, thread-safe by class-init semantics, but always created.
2. **Bill Pugh / holder idiom** — lazy + thread-safe, no synchronization:
```java
public class Cache {
    private Cache() {}
    private static class Holder { static final Cache INSTANCE = new Cache(); }
    public static Cache getInstance() { return Holder.INSTANCE; }
}
```
3. **enum Singleton** — best per *Effective Java*: serialization-safe and reflection-safe.
4. **Double-checked locking with volatile** — works, but verbose.
5. `synchronized getInstance()` — correct but a lock on every read.

**Why it's tricky:** a Singleton means "exactly one instance per JVM (per classloader)". The naive lazy version `if (instance == null) instance = new X();` is a race: two threads can both see `null` and create two instances. Why the holder idiom works: the JVM guarantees a class is initialized **once, under an internal lock**, and `Holder` is only loaded on the first `getInstance()` call — so you get laziness and thread-safety for free.

**Ways singletons get broken (follow-up probe):** reflection (`setAccessible(true)` on the private constructor), serialization (deserializing creates a new instance unless you implement `readResolve()`), cloning, and multiple classloaders. An `enum` Singleton (`enum Cache { INSTANCE; }`) is immune to the first three because the JVM forbids reflective enum instantiation and handles enum serialization specially.

**In Spring you don't write singletons** — a `@Component` is already a singleton-scoped bean, container-managed and injectable (therefore testable). Handwritten singletons are static global state and a testing hazard.

**Remember:** *Enum or holder idiom; in Spring, just use a bean.*

---

### Q153. Process vs thread, and concurrency vs parallelism?

**Say this first:** A process is an isolated program with its own memory; threads are lightweight workers *inside* a process that share its memory. Concurrency is about *dealing with* many things at once; parallelism is about *doing* many things at the same instant.

**Answer:**
- **Process** — an OS-level running program. It has its own virtual address space, file handles and at least one thread. Processes cannot see each other's memory, so they talk through pipes, sockets or shared files (*IPC* = inter-process communication). A crash in one process usually does not kill another.
- **Thread** — a unit of execution *inside* a process. All threads share the **heap** (objects, statics) but each has its own **stack** (local variables, call frames) and program counter. Sharing memory makes communication cheap — and is exactly why we need synchronization.
- **Why threads exist:** creating a process and switching between processes is heavy. Threads are cheaper (a platform thread reserves ~1 MB of stack by default on 64-bit JVMs, adjustable with `-Xss`), and they let one program overlap waiting (I/O) with work.

| | Process | Thread |
|---|---|---|
| Memory | Own address space | Shared heap, own stack |
| Creation / context switch | Expensive | Cheaper |
| Communication | IPC (sockets, pipes) | Shared variables (needs sync) |
| Failure isolation | Strong | Weak (an `OutOfMemoryError` or bad shared state hurts all) |

**Concurrency vs parallelism:**
```
Concurrency (1 core):   T1 ██░░██░░██     tasks interleave; only one runs at any instant
                        T2 ░░██░░██░░     (a cook switching between two pans)
Parallelism (2 cores):  T1 ██████████     tasks really run at the same time
                        T2 ██████████     (two cooks, two pans)
```
- Concurrency is a **design** property: structure the program as independent tasks so that waiting on one does not stall the others. It works even on one CPU core.
- Parallelism is an **execution** property: needs multiple cores. It is used to cut wall-clock time for CPU-heavy work (`parallelStream`, Fork/Join).
- A program can be concurrent but not parallel (single core), or parallel but not very concurrent (SIMD number crunching).

**Pitfalls / follow-up probes:**
- "Does adding threads always speed things up?" No — **Amdahl's law**: the serial fraction limits speedup (if 10% is serial, the max is 10x however many cores). Thread switching, lock contention and cache misses add cost.
- "Where do Java threads live?" A platform thread maps 1:1 to an OS thread; a virtual thread (Java 21) is scheduled by the JVM on a few carrier OS threads (see the virtual-thread question).
- Static fields and heap objects are shared; local variables and method parameters (primitives/references) are not — they live on the thread's stack.

**Remember:** *Process = separate house, thread = roommate sharing the kitchen. Concurrency = juggling, parallelism = many hands.*

---

### Q154. `start()` vs `run()`, and `sleep` / `join` / `yield` — what does each really do?

**Say this first:** `start()` asks the JVM to create a new thread that will call `run()`; calling `run()` yourself is just an ordinary method call on the current thread.

**Answer:**
- `start()` — registers the thread with the OS/JVM, moves it `NEW → RUNNABLE`, and the new thread later invokes `run()`. It can be called **once**; a second call throws `IllegalThreadStateException`.
- `run()` — just the body. Calling it directly runs it synchronously on the caller's thread — no concurrency at all.
- `Thread.sleep(ms)` — static; pauses the *current* thread (`TIMED_WAITING`) **without releasing any lock it holds**. Can throw `InterruptedException`.
- `t.join()` — the *current* thread waits until `t` finishes (`WAITING`). It also gives a happens-before edge: everything `t` did is visible after `join()` returns. `join(ms)` adds a timeout.
- `Thread.yield()` — a *hint* to the scheduler "I can give up my time slice". It may be ignored; never build correctness on it.
- `t.interrupt()` — sets the interrupt flag; see the interruption question.

```java
Thread t = new Thread(() -> System.out.println("in " + Thread.currentThread().getName()), "worker-1");
t.run();    // prints "in main"      -> no new thread
t.start();  // prints "in worker-1"  -> new thread
t.join();   // main waits for worker-1 to finish
```

**Why `start()` rather than just calling `run()`?** `start()` is the native hook that actually allocates a thread stack and tells the scheduler about it. `run()` is only the code to execute.

**Pitfalls / follow-up probes:**
- `t.sleep(1000)` on an instance still sleeps the *current* thread (static method called via an instance) — a classic trick question.
- Overriding `start()` and forgetting `super.start()` means no thread is created.
- `join()` without a timeout can hang forever if `t` never ends; prefer `join(timeout)` and check `t.isAlive()`.
- Exceptions thrown inside a thread do not propagate to the caller of `start()`; use `setUncaughtExceptionHandler`, or `Future.get()` which wraps them in `ExecutionException`.
- `Thread.sleep` vs `Object.wait` — compared in the wait/notify question.

**Remember:** *start() = new thread; run() = plain method; sleep keeps locks; join waits for another thread; yield is only a hint.*

---

### Q155. How do you stop a thread properly? Explain the interruption protocol and `InterruptedException`.

**Say this first:** You never *force* a Java thread to stop. You *ask* it to stop (interrupt) and the thread's own code cooperates.

**Answer:**
- `Thread.stop()` is deprecated for removal (and throws `UnsupportedOperationException` on modern JDKs) because killing a thread mid-way would leave locks released with half-updated objects.
- Instead Java has **cooperative cancellation** using an **interrupt flag** (a boolean inside each `Thread`):
  1. Someone calls `t.interrupt()` → flag set to `true`.
  2. If `t` is blocked in `sleep`, `wait`, `join`, `BlockingQueue.take/put`, `Lock.lockInterruptibly`, etc., that call wakes up immediately, **clears the flag**, and throws `InterruptedException`.
  3. If `t` is running CPU code, nothing happens until the code checks `Thread.currentThread().isInterrupted()`.
- `isInterrupted()` reads the flag; the static `Thread.interrupted()` reads **and clears** it.

```java
class Worker implements Runnable {
    public void run() {
        try {
            while (!Thread.currentThread().isInterrupted()) {
                Job job = queue.take();            // blocks; throws if interrupted
                process(job);
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();    // restore the flag for callers
        } finally {
            cleanup();                             // release resources
        }
    }
}
// owner: worker.interrupt(); worker.join(5000);
```

**Handling `InterruptedException` — the rules:**
1. **Best:** declare `throws InterruptedException` and let it bubble up.
2. If you can't (e.g. inside `Runnable.run`), **restore the flag**: `Thread.currentThread().interrupt();` and stop what you are doing.
3. **Never** swallow it with an empty `catch` — the interrupt request vanishes (because the flag was cleared when it was thrown) and the thread becomes unstoppable by pools (`shutdownNow` relies on interrupts).

A `volatile boolean running` flag also works for simple loops but doesn't wake a thread blocked in `take()` — interrupts do.

**Pitfalls / follow-up probes:**
- Blocking I/O on classic `InputStream`/`Socket.read` ignores interrupts; close the stream/socket to unblock.
- Spinning on `while(true)` with no check is uninterruptible.
- Catching `InterruptedException` inside a loop and `continue`-ing causes a busy-loop or an unstoppable thread.
- `Future.cancel(true)` works by interrupting the running thread.

**Remember:** *Interrupt = polite tap on the shoulder. Check the flag, clean up, and if you catch the exception, re-set the flag.*

---

### Q156. Daemon vs user threads — and why does it matter at shutdown?

**Say this first:** The JVM exits when the last **user** thread ends; daemon threads are "background helpers" that do not keep the JVM alive and are killed abruptly.

**Answer:**
- **User (non-daemon) thread** — the default. The JVM stays up until all of them finish (or `System.exit` is called).
- **Daemon thread** — service thread (GC, finalizer/reference handlers, JIT compilers are internal examples; timers, heartbeats, cache cleaners are typical app uses). When only daemon threads remain, the JVM exits *immediately*, mid-instruction.
- Set with `t.setDaemon(true)` **before** `start()`; otherwise `IllegalThreadStateException`.
- A new thread **inherits** the daemon status of the thread that created it.
- Virtual threads are always daemon.

```java
Thread heartbeat = new Thread(() -> { while (true) { ping(); sleepQuietly(1000); } });
heartbeat.setDaemon(true);   // must precede start()
heartbeat.start();
// main ends -> JVM exits even though heartbeat is still looping
```

**Why it matters:**
- A forgotten **non-daemon** pool thread keeps an app (or a unit-test run) from exiting — the classic "my `main` finished but the process hangs" bug. Fix: call `shutdown()`, or give the pool a `ThreadFactory` that makes daemon threads.
- Daemon threads must **not** do work that needs guaranteed completion (writing files, committing transactions): `finally` blocks and shutdown of resources may never run when the JVM dies.

**Pitfalls / follow-up probes:**
- "Can a daemon thread create a user thread?" Yes, by explicitly `setDaemon(false)`; otherwise the child inherits daemon-ness.
- Use a **shutdown hook** (`Runtime.addShutdownHook`) for graceful cleanup — it runs on `System.exit`/SIGTERM, not on `kill -9`.
- Thread priority is unrelated; it is only an OS hint.

**Remember:** *Daemon = disposable background staff; the JVM turns off the lights when only they are left.*

---

### Q157. Race condition vs data race vs critical section — and visibility vs atomicity vs ordering?

**Say this first:** Three things can go wrong with shared mutable state: threads don't **see** each other's writes (visibility), operations get **interleaved** (atomicity), or instructions appear **reordered** (ordering).

**Answer — the vocabulary:**
- **Critical section** — a block of code that touches shared mutable data and must be executed by one thread at a time (guarded by `synchronized` / `Lock`).
- **Race condition** — the program's *result depends on timing* of threads. Typical shape: **check-then-act** (`if (!map.containsKey(k)) map.put(k, v)`) or **read-modify-write** (`count++`). It is a *logic* bug.
- **Data race** — a *memory-model* term: two threads access the same variable, at least one writes, and there is **no happens-before** between them. The JMM then allows surprising results (stale values, reordered effects).
- They overlap but differ: you can have a race condition with no data race (every access is synchronized, but you lock per-call instead of around the whole check-then-act), and a data race that is "benign" (e.g. `String.hashCode` caching — carefully designed).

**The three hazards:**

| Hazard | Meaning | Fixed by |
|---|---|---|
| **Visibility** | A write by T1 may sit in a CPU cache/register; T2 keeps reading the old value | `volatile`, `synchronized`, `Lock`, `Atomic*`, `final` |
| **Atomicity** | A compound action (`i++`, check-then-act) can be interleaved mid-way | `synchronized`, `Lock`, `Atomic*` / CAS, concurrent collections' atomic methods |
| **Ordering** | Compiler/CPU may reorder instructions that look independent to a single thread | `volatile`, `synchronized`, happens-before rules |

```java
// Race condition even though ConcurrentHashMap itself is thread-safe:
if (!map.containsKey(key)) {      // T1 and T2 both see "absent"
    map.put(key, compute());      // both compute, last write wins
}
// Fix: one atomic operation
map.computeIfAbsent(key, k -> compute());
```

**Pitfalls / follow-up probes:**
- "Is a thread-safe collection enough?" No — each call is atomic, a *sequence* of calls is not.
- Without synchronization a loop `while (!done) {}` may never terminate because the JIT hoists the read of `done` out of the loop → mark `done` `volatile`.
- Reads/writes of `long`/`double` that are not `volatile` are technically allowed to be non-atomic (two 32-bit halves) by the JLS, though 64-bit JVMs do it atomically in practice.

**Remember:** *Visibility: do I see it? Atomicity: is it all-or-nothing? Ordering: in what sequence? Miss any one and you have a race.*

---

### Q158. Show that `volatile int count; count++` is not thread-safe, and fix it three ways.

**Say this first:** `count++` is really *read → add one → write*. `volatile` makes each read and write visible, but another thread can sneak in between them.

**Answer:**
```java
public class LostUpdate {
    static volatile int counter = 0;                 // visible, but NOT atomic

    public static void main(String[] a) throws Exception {
        Runnable r = () -> { for (int i = 0; i < 100_000; i++) counter++; };
        Thread t1 = new Thread(r), t2 = new Thread(r);
        t1.start(); t2.start(); t1.join(); t2.join();
        System.out.println(counter);                 // expected 200000, usually less
    }
}
```
**Why updates are lost:**
```
counter = 5
T1: read 5 ─────────► add → 6 ─────────► write 6
T2:        read 5 ─► add → 6 ─► write 6          (both wrote 6; one increment lost)
```

**Fixes:**
```java
// 1. synchronized — mutual exclusion
static int counter; static synchronized void inc() { counter++; }

// 2. AtomicInteger — lock-free CAS
static final AtomicInteger counter = new AtomicInteger();
counter.incrementAndGet();

// 3. LongAdder — best when many threads hammer one counter
static final LongAdder counter = new LongAdder();
counter.increment();  ...  counter.sum();
```
- `volatile` *is* enough when only **one** thread writes and others read (a stop flag, a published reference to an immutable object), or when the new value doesn't depend on the old one.

**Pitfalls / follow-up probes:**
- "Why does the test sometimes print 200000?" Races are timing dependent; run more iterations/threads to expose them. Absence of failure ≠ correctness.
- `synchronized` on a *non-static* method with a *static* counter locks `this`, not the class → different instances don't exclude each other. Lock the same object everywhere.
- `volatile` on an array does not make the elements volatile (only the reference). Use `AtomicIntegerArray`/`VarHandle`.

**Remember:** *volatile guards the door's visibility, not the three-step dance inside the room.*

---

### Q159. How does `synchronized` work internally? (monitor, object header, thin/fat locks, reentrancy)

**Say this first:** Every Java object can act as a **monitor** (a lock + wait set). `synchronized` acquires that monitor on entry and releases it on exit — the JVM makes it cheap when uncontended and escalates when threads fight.

**Answer:**
- **Syntax → bytecode:** a `synchronized` *block* compiles to `monitorenter` / `monitorexit` (with an exception handler that also exits); a `synchronized` *method* sets the `ACC_SYNCHRONIZED` flag and the JVM acquires the monitor implicitly. `static synchronized` locks the `Class` object; instance ones lock `this`.
- **Object header:** each object has a header (the **mark word**) that stores lock state, identity hash code and GC age. The lock state lives there, so no extra lock object is needed until contention.
- **Lock escalation (HotSpot, conceptually):**
  1. **Unlocked** → 2. **Thin / lightweight lock** — a thread CAS-es a pointer into the mark word; no OS involvement; ideal when only one thread (or non-overlapping threads) use it → 3. **Fat / inflated lock** — under real contention the lock is *inflated* to an `ObjectMonitor` with an entry queue; blocked threads are parked by the OS (cost: context switches).
  - **Biased locking** (lock "owned" by the first thread to skip CAS) existed in old JDKs but was disabled by default in JDK 15 and later removed — don't quote it as current behaviour; just know the term.
- **Reentrant:** the owning thread may enter the same monitor again; the JVM keeps a recursion count. So a `synchronized` method can call another `synchronized` method on the same object without self-deadlock.
- **Memory effects:** entering = acquire (sees all writes made before the previous release); exiting = release (flushes writes). That is the monitor happens-before rule.
- **JIT optimizations:** lock elision (escape analysis proves the object is thread-local → lock removed), lock coarsening (merge adjacent locks), adaptive spinning.

```java
class Counter {
    private int n;
    public synchronized void inc() { n++; }          // lock = this
    public void inc2() { synchronized (this) { n++; } }  // same thing
    public static synchronized void s() { }          // lock = Counter.class
}
```

**Intrinsic vs explicit locks:** `synchronized` = *intrinsic* (built into the language, auto-release, block-structured). `Lock` = *explicit* library class (flexible, you unlock in `finally`). See the ReentrantLock question.

**Pitfalls / follow-up probes:**
- Never lock on a `String` literal, boxed `Integer`, or a publicly exposed object — other code may lock the same instance. Use a `private final Object lock = new Object();`.
- Don't call unknown/alien code (callbacks, I/O) while holding a lock — long holds and deadlocks.
- `synchronized` is not fair and cannot be interrupted or timed out while waiting to enter.
- Before JDK 24, a virtual thread blocked inside `synchronized` pins its carrier thread (see the virtual threads question).

**Remember:** *Object = lock + mark word; thin when alone, fat when crowded, reentrant always.*

---

### Q160. `ReentrantLock` in depth — `tryLock`, `lockInterruptibly`, fairness, `Condition`.

**Say this first:** `ReentrantLock` does what `synchronized` does, plus the knobs you need when you must not block forever: try, time out, be interrupted, be fair, and wait on several conditions.

**Answer:**
| Feature | `synchronized` | `ReentrantLock` |
|---|---|---|
| Release | Automatic at block end | **You** call `unlock()` in `finally` |
| Try without blocking | No | `tryLock()`, `tryLock(time, unit)` |
| Interruptible wait for lock | No | `lockInterruptibly()` |
| Fairness | Never fair | `new ReentrantLock(true)` → longest waiter first (lower throughput) |
| Wait/notify | One wait set per object | Many `Condition`s per lock |
| Introspection | None | `isLocked()`, `getQueueLength()`, `isHeldByCurrentThread()` |

**Standard idiom:**
```java
private final ReentrantLock lock = new ReentrantLock();

void update() {
    lock.lock();
    try {
        // critical section
    } finally {
        lock.unlock();               // always, even on exception
    }
}

boolean tryUpdate() throws InterruptedException {
    if (lock.tryLock(200, TimeUnit.MILLISECONDS)) {   // give up after 200 ms
        try { /* work */ return true; } finally { lock.unlock(); }
    }
    return false;                                      // fallback: retry / fail fast
}
```

**`Condition` = `wait/notify` per lock** (`await()`, `signal()`, `signalAll()`). Having several lets you wake only the right group — e.g. a bounded buffer with separate `notFull` and `notEmpty` conditions (see the bounded-buffer question), avoiding the "wake everyone" cost of `notifyAll`.

**Reentrancy:** hold count increments each time the owner locks again; it must call `unlock()` the same number of times. `lock.getHoldCount()` shows it.

**Fairness caveat:** a *fair* lock prevents starvation but is often several times slower, since it forbids "barging" (a newly arriving thread grabbing the lock the instant it frees). Default non-fair is usually right.

**Pitfalls / follow-up probes:**
- Placing `lock()` *inside* the `try` — if `lock()` itself fails you'd unlock a lock you don't hold (`IllegalMonitorStateException`). Lock **before** `try`.
- `tryLock()` (no args) barges even on a fair lock.
- Forgetting `unlock()` on one path = lock leak = hung app.
- Java 21 virtual threads handle `ReentrantLock` well (they park instead of pinning).

**Remember:** *lock(); try { … } finally { unlock(); } — and reach for tryLock when "wait forever" is unacceptable.*

---

### Q161. `ReentrantReadWriteLock` vs `StampedLock` — optimistic reads explained.

**Say this first:** When reads vastly outnumber writes, let many readers in together and make only writers exclusive. `StampedLock` goes further: readers may read **without any lock** and just *validate* afterwards.

**Answer:**
**`ReentrantReadWriteLock`** — two locks: a shared **read lock** and an exclusive **write lock**.
- Many threads can hold the read lock at once; the write lock excludes everyone.
- **Downgrading** is allowed (hold write → acquire read → release write). **Upgrading** (read → write) is **not** — it deadlocks if two readers try it.
- It is reentrant and supports `Condition` (on the write lock only).

```java
class PriceBook {
    private final ReadWriteLock rw = new ReentrantReadWriteLock();
    private final Map<String, Double> prices = new HashMap<>();

    Double get(String k) { rw.readLock().lock();  try { return prices.get(k); }  finally { rw.readLock().unlock(); } }
    void   put(String k, double v) { rw.writeLock().lock(); try { prices.put(k, v); } finally { rw.writeLock().unlock(); } }
}
```

**`StampedLock` (Java 8)** — returns a long **stamp** (a version token) instead of a lock object. Three modes: write, read, and **optimistic read**.
```java
class Point {
    private final StampedLock sl = new StampedLock();
    private double x, y;

    double distanceFromOrigin() {
        long stamp = sl.tryOptimisticRead();     // no blocking, no write to shared lock state
        double cx = x, cy = y;                   // read optimistically
        if (!sl.validate(stamp)) {               // a writer slipped in? then fall back
            stamp = sl.readLock();
            try { cx = x; cy = y; } finally { sl.unlockRead(stamp); }
        }
        return Math.hypot(cx, cy);
    }
    void move(double dx, double dy) {
        long stamp = sl.writeLock();
        try { x += dx; y += dy; } finally { sl.unlockWrite(stamp); }
    }
}
```
- Optimistic reads are fastest because readers don't contend on lock state; the price is that you must read into locals and re-check (`validate`) before using them.

| | RWLock | StampedLock |
|---|---|---|
| Reentrant | Yes | **No** (re-locking yourself deadlocks) |
| `Condition` | Yes (write) | No |
| Optimistic read | No | Yes |
| Complexity | Lower | Higher, easy to misuse |

**Pitfalls / follow-up probes:**
- RW lock can **starve writers** (policy-dependent) when readers keep arriving; and for short critical sections the bookkeeping overhead can make it *slower* than plain `synchronized` — measure.
- `StampedLock` does not support interrupt-safe reentrancy and its stamps must be passed to the matching unlock.
- If the read data is just a map, `ConcurrentHashMap` may be the simpler answer.

**Remember:** *RWLock: many readers or one writer. StampedLock: read first, validate later, lock only if invalidated.*

---

### Q162. What is `AbstractQueuedSynchronizer` (AQS) and how does it power locks?

**Say this first:** AQS is the shared skeleton behind `ReentrantLock`, `Semaphore`, `CountDownLatch`, `ReentrantReadWriteLock` and friends: one `int state` plus a queue of waiting threads. Each class just decides what the state *means*.

**Answer:**
- **State:** a `volatile int state` changed with CAS.
  - `ReentrantLock`: 0 = free, n = held n times (reentrancy count).
  - `Semaphore`: number of available permits.
  - `CountDownLatch`: remaining count; threads wait until it hits 0.
- **Wait queue:** a FIFO linked queue (a variant of the CLH lock queue) of threads that failed to acquire.
- **Template method pattern** — a subclass overrides only:
  - `tryAcquire(int)` / `tryRelease(int)` for exclusive mode,
  - `tryAcquireShared(int)` / `tryReleaseShared(int)` for shared mode.
  AQS supplies `acquire`, `release`, queuing, blocking and waking.

**How `lock()` flows (conceptual):**
```
lock()
 └─ tryAcquire(): CAS state 0 → 1 ?
      ├─ yes → set owner = current thread → done
      └─ no  → enqueue a node, park the thread (LockSupport.park)
unlock()
 └─ tryRelease(): state → 0, then unpark the head's successor
      woken thread loops and retries tryAcquire (it may lose to a "barging" thread if non-fair)
```
- `LockSupport.park/unpark` is the primitive that suspends/resumes a thread without a monitor.
- **Fair vs non-fair** differ only in `tryAcquire`: fair checks "is anyone queued before me?" first.
- **Condition** objects (`newCondition()`) keep their own wait queue; `signal()` moves a node from the condition queue to the lock queue.

**Minimal custom synchronizer (a mutex):**
```java
class Mutex {
    private static class Sync extends AbstractQueuedSynchronizer {
        protected boolean tryAcquire(int ignored) { return compareAndSetState(0, 1); }
        protected boolean tryRelease(int ignored) { setState(0); return true; }
    }
    private final Sync sync = new Sync();
    void lock()   { sync.acquire(1); }
    void unlock() { sync.release(1); }
}
```
(A production one would also record the owner thread; this shows the idea.)

**Pitfalls / follow-up probes:**
- You rarely write an AQS subclass; knowing it explains why `Semaphore`, `Latch`, `ReentrantLock` have the same queuing and interrupt/timeout behaviour.
- "Why not just spin on CAS?" Spinning burns CPU; AQS spins briefly then **parks** the thread.
- JDK 9+ replaced the old node-based classes with `VarHandle`s internally; behaviour is the same.

**Remember:** *AQS = volatile int + CAS + a parked-thread queue. Subclasses only define "what does state mean".*

---

### Q163. CAS, the ABA problem, `AtomicStampedReference`, `LongAdder` vs `AtomicLong`, and false sharing.

**Say this first:** **CAS** (*compare-and-swap*) is a single CPU instruction: "set this variable to NEW only if it still equals EXPECTED; tell me whether it worked." Atomics loop on CAS instead of taking a lock.

**Answer:**
```java
// what AtomicInteger.incrementAndGet() effectively does
int prev, next;
do { prev = value.get(); next = prev + 1; }
while (!value.compareAndSet(prev, next));   // retry if someone changed it meanwhile
```
- **Optimistic** concurrency: no blocking, no deadlock, great under low/moderate contention. Under *high* contention many threads fail and retry → wasted CPU.

**ABA problem:** CAS only checks that the value *equals* what you read, not that it was *unchanged*. If it went `A → B → A`, your CAS succeeds although the world changed.
- Classic example: a lock-free stack. T1 reads head=A (next=B), is paused. T2 pops A, pops B, pushes A back (now A.next is something else). T1 resumes, CAS(head, A, B) succeeds and corrupts the stack (B was already popped).
- Fixes: add a **version stamp** — `AtomicStampedReference<V>` (reference + int stamp) or `AtomicMarkableReference` (reference + boolean).
```java
AtomicStampedReference<String> ref = new AtomicStampedReference<>("A", 0);
int[] holder = new int[1];
String cur = ref.get(holder);                        // value + current stamp
ref.compareAndSet(cur, "B", holder[0], holder[0] + 1);   // fails if the stamp moved, even if value is "A" again
```
- In Java with garbage collection ABA is rarer for plain references (a freed node cannot be reused while you still hold it), but it matters for pooled/recycled objects and numeric values.

**`AtomicLong` vs `LongAdder`:**
- `AtomicLong` — one hot variable; all threads CAS the same memory location → contention retries.
- `LongAdder` — keeps a `base` plus an array of **cells**; contended threads update different cells, `sum()` adds them up. Much higher throughput for write-heavy counters. Trade-off: `sum()` is not an atomic snapshot, and no `compareAndSet`/get-and-increment semantics (use it for statistics, not sequence generators/IDs).
- `LongAccumulator` generalizes it with your own function (max, min).

**False sharing (awareness):** CPUs move memory in **cache lines** (~64 bytes). If two threads write *different* variables that sit on the *same* line, each write invalidates the other core's cache copy and they ping-pong — slow despite no logical sharing. `LongAdder`'s cells are padded using the JDK-internal `@jdk.internal.vm.annotation.Contended` annotation to avoid exactly this (application code needs `-XX:-RestrictContended` to use it; usually you manually pad or restructure data instead).

**Pitfalls / follow-up probes:**
- `AtomicInteger` ops are atomic individually; `if (a.get() < 10) a.incrementAndGet()` is a **race** — use `updateAndGet`/`compareAndSet` loop.
- `AtomicReference` compares with `==` (reference identity), not `equals`.
- Java 9+ `VarHandle` offers CAS on plain fields with fine-grained memory modes.

**Remember:** *CAS = "change it only if nobody beat me to it". ABA needs a stamp; LongAdder spreads contention; padding fights false sharing.*

---

### Q164. What is safe publication, and why does double-checked locking need `volatile`?

**Say this first:** Making an object *reachable* by another thread isn't enough — you must also make its **contents** visible. Without a happens-before edge, another thread can see the reference but a half-constructed object.

**Answer:**
- **Publication** = making an object visible to other threads (storing into a field, queue, collection). **Unsafe publication** can expose stale or default field values (0/null) because the constructor's writes may be reordered after the reference write.
- **Safe publication idioms** (each creates happens-before):
  1. Initialize via a `static` initializer (class-init lock).
  2. Store in a `volatile` field or `AtomicReference`.
  3. Store in a `final` field of a properly constructed object (don't leak `this` from the constructor).
  4. Store inside a `synchronized` block/`Lock` and read under the same lock.
  5. Hand over through a thread-safe collection (`ConcurrentHashMap`, `BlockingQueue`) or `Thread.start()`/`join()`.

**Double-checked locking (DCL):**
```java
class Service {
    private static volatile Service instance;      // volatile is REQUIRED
    static Service get() {
        Service s = instance;                      // 1st check, no lock (fast path)
        if (s == null) {
            synchronized (Service.class) {
                s = instance;
                if (s == null) instance = s = new Service();   // 2nd check under lock
            }
        }
        return s;
    }
}
```
**Why volatile?** `instance = new Service()` is three steps: (1) allocate memory, (2) run constructor, (3) assign the reference. Without `volatile` the JIT/CPU may reorder 3 before 2, so thread B sees non-null `instance` whose fields aren't initialized yet. `volatile` forbids that reordering and gives the happens-before edge. (Java 5+ semantics; broken in older JMM.)

```
Without volatile (possible):  A: allocate → assign ref → [constructor still running]
                              B: sees ref != null → uses half-built object ✗
```
**Alternatives often better than DCL:** the Initialization-on-demand holder idiom, or an `enum` singleton (covered in the Singleton question) — the JVM's class-loading lock gives lazy, thread-safe init for free.

**Pitfalls / follow-up probes:**
- A `final` field's guarantee is lost if `this` escapes during construction (e.g. registering a listener in the constructor).
- Mutable object published safely is still not thread-safe to *modify*; only the initial state is guaranteed visible.
- `HashMap` published through a non-volatile field, then read by others: visible reference, potentially stale contents.

**Remember:** *Publish through a happens-before door: static init, volatile, final, lock, or concurrent queue.*

---

### Q165. Thread-safe vs immutable vs thread confinement — what are your options to avoid sharing problems?

**Say this first:** The cheapest synchronization is no synchronization: don't share, or don't mutate.

**Answer — a strategy ladder (prefer top to bottom):**
1. **Don't share state** — **thread confinement**: data is used by only one thread.
   - *Stack confinement* — local variables never escape the method.
   - `ThreadLocal` — per-thread copy (e.g. `SimpleDateFormat` in old code; clean up in pools).
   - *Single-writer / actor style* — only one thread mutates; others send it messages through a queue.
2. **Share but never mutate** — **immutable objects** (`final` fields, no setters, defensive copies, `String`, `record`, `List.of`). Always thread-safe, no locks, safe to publish. (See the immutable class question.)
3. **Share and mutate, with protection:**
   - Confine to a lock (`synchronized`/`Lock`),
   - Use lock-free (`Atomic*`, CAS),
   - Use concurrent collections (`ConcurrentHashMap`, `CopyOnWriteArrayList`, `BlockingQueue`),
   - **Copy-on-write** — mutate a copy and swap a `volatile` reference: readers never block.

**Thread-safe ≠ immutable:** a thread-safe object may be mutable but guards its state internally (`AtomicInteger`, `ConcurrentHashMap`, `StringBuffer`). Immutable objects are the simplest kind of thread-safe.

```java
// copy-on-write snapshot: readers lock-free, writers rare
private volatile List<Rule> rules = List.of();
void reload(List<Rule> fresh) { rules = List.copyOf(fresh); }   // atomic swap of an immutable list
boolean allow(Req r) { for (Rule x : rules) if (!x.ok(r)) return false; return true; }
```

**Levels of thread-safety (Bloch's vocabulary):** immutable → unconditionally thread-safe (`AtomicLong`) → conditionally thread-safe (needs external lock for some sequences, e.g. iterating `Collections.synchronizedList`) → not thread-safe (`ArrayList`) → thread-hostile.

**Pitfalls / follow-up probes:**
- `final List<String> list` is a final *reference*; the list is still mutable.
- `Collections.unmodifiableList(x)` is just a read-only **view** — changes to `x` show through. `List.copyOf` is a true immutable copy.
- Document the policy: `@ThreadSafe`, `@GuardedBy("lock")` annotations (from JCIP) communicate intent.

**Remember:** *Confine, then freeze, then lock — in that order of preference.*

---

### Q166. `Executors` factory methods — fixed, cached, single, scheduled, work-stealing — and how do you name your threads?

**Say this first:** Each factory is a pre-tuned `ThreadPoolExecutor`/`ForkJoinPool` configuration; know the hidden queue and size limits before using them in production.

**Answer:**
| Factory | Core / max | Queue | When / danger |
|---|---|---|---|
| `newFixedThreadPool(n)` | n / n | **Unbounded** `LinkedBlockingQueue` | Steady load; unbounded queue → OOM under overload |
| `newCachedThreadPool()` | 0 / **unbounded** | `SynchronousQueue` (no buffering); idle threads die after 60 s | Many short tasks; spike → thousands of threads → OOM |
| `newSingleThreadExecutor()` | 1 / 1 | Unbounded | Serial execution in order; replaces the thread if it dies; wrapped so you can't reconfigure it |
| `newScheduledThreadPool(n)` | n / unbounded | Delay queue | Delayed/periodic tasks |
| `newWorkStealingPool()` | `ForkJoinPool` with parallelism = CPUs | Per-worker deques | Many small, CPU-bound, uneven tasks; threads are daemon |
| `newVirtualThreadPerTaskExecutor()` (21) | One virtual thread per task | None | Blocking I/O tasks |

(Details on core/max/queue semantics and rejection are in the `ThreadPoolExecutor` question.) In production prefer constructing a `ThreadPoolExecutor` with a **bounded queue** and an explicit rejection policy.

**ScheduledExecutorService:**
```java
ScheduledExecutorService ses = Executors.newScheduledThreadPool(2, namedFactory("sched"));
ses.schedule(() -> cleanup(), 5, TimeUnit.SECONDS);                       // once, after delay
ses.scheduleAtFixedRate(() -> poll(), 0, 10, TimeUnit.SECONDS);           // start-to-start every 10 s
ses.scheduleWithFixedDelay(() -> poll(), 0, 10, TimeUnit.SECONDS);        // end-to-start: wait 10 s after each run ends
```
- `FixedRate`: if a run takes longer than the period, the next starts late (runs never overlap, but they bunch up).
- **If a periodic task throws, further runs are silently cancelled** — wrap the body in `try/catch`.
- Prefer it over `java.util.Timer` (single thread, an exception kills the timer).

**Naming threads with a `ThreadFactory`** — essential for thread dumps/logs:
```java
static ThreadFactory namedFactory(String prefix) {
    AtomicInteger n = new AtomicInteger(1);
    return r -> {
        Thread t = new Thread(r, prefix + "-" + n.getAndIncrement());
        t.setUncaughtExceptionHandler((th, ex) -> log.error("Uncaught in " + th.getName(), ex));
        return t;
    };
}
// Java 21 style:  Thread.ofPlatform().name("worker-", 1).daemon(false).factory()
//                 Thread.ofVirtual().name("vt-", 0).factory()
```

**Pitfalls / follow-up probes:**
- Default names `pool-3-thread-7` tell you nothing in a dump. Name by purpose (`orders-io-4`).
- Don't create an executor per request — pools are long-lived, shared and shut down at application stop.
- `submit()` swallows task exceptions into the `Future`; `execute()` sends them to the uncaught handler.

**Remember:** *Know the queue behind each factory: fixed = unbounded queue, cached = unbounded threads. Name your threads.*

---

### Q167. `shutdown()` vs `shutdownNow()` vs `awaitTermination()`, and rejection policies.

**Say this first:** `shutdown` = "finish what you have, accept nothing new"; `shutdownNow` = "stop ASAP"; `awaitTermination` = "I'll wait for you".

**Answer:**
| Call | Effect |
|---|---|
| `shutdown()` | Stops accepting new tasks; already submitted tasks (running **and queued**) complete. Non-blocking. |
| `shutdownNow()` | Interrupts running workers, **drains and returns** the never-started tasks (`List<Runnable>`). Best effort — tasks that ignore interrupts keep running. |
| `awaitTermination(t, unit)` | Blocks until all tasks finish or timeout; returns `true` if terminated. Doesn't itself stop anything. |
| `isShutdown()` / `isTerminated()` | State checks (terminated = shut down **and** all done). |

**Canonical graceful-shutdown (from the `ExecutorService` Javadoc):**
```java
void stop(ExecutorService pool) {
    pool.shutdown();                                   // 1. no new work
    try {
        if (!pool.awaitTermination(30, TimeUnit.SECONDS)) {
            pool.shutdownNow();                        // 2. cancel stragglers via interrupt
            if (!pool.awaitTermination(10, TimeUnit.SECONDS))
                log.warn("Pool did not terminate");
        }
    } catch (InterruptedException e) {
        pool.shutdownNow();
        Thread.currentThread().interrupt();
    }
}
```
- Java 19+: `ExecutorService` is `AutoCloseable`; `close()` does shutdown + await. So `try (var ex = Executors.newFixedThreadPool(4)) { ... }` is enough for short-lived pools.

**Rejection policies** (when queue is full *and* threads are at max, or the pool is shut down) — set via `RejectedExecutionHandler`:
| Policy | Behaviour |
|---|---|
| `AbortPolicy` (default) | Throws `RejectedExecutionException` |
| `CallerRunsPolicy` | The submitting thread runs the task itself → natural **back-pressure** (slows producer) |
| `DiscardPolicy` | Silently drops the new task (dangerous: invisible data loss) |
| `DiscardOldestPolicy` | Drops the oldest queued task and retries the new one |
| Custom | Log, meter, persist to a dead-letter store, or block with `queue.put` |

**Pitfalls / follow-up probes:**
- Forgetting `shutdown()` on a non-daemon pool keeps the JVM alive.
- `CallerRunsPolicy` on a web-request thread can make a slow downstream stall the *request* thread — know your back-pressure.
- `shutdownNow()` returns tasks that were never started; decide whether to re-queue or log them.
- Interrupt only works if tasks are interruptible (see the interruption question).

**Remember:** *shutdown → await → shutdownNow → await. Bounded queue + CallerRuns gives back-pressure.*

---

### Q168. How do you size a thread pool? (CPU-bound vs I/O-bound, Little's law)

**Say this first:** Pool size should match the bottleneck: CPU-bound work ≈ number of cores; I/O-bound work ≈ cores × (1 + wait/compute).

**Answer:**
- Get cores: `int n = Runtime.getRuntime().availableProcessors();` (container-aware on modern JDKs).
- **CPU-bound** (hashing, parsing, compression): extra threads only add context switches. Use **N or N+1** (the +1 covers an occasional page fault or pause).
- **I/O-bound** (DB, HTTP, disk): threads spend most time waiting, so more threads keep the CPUs busy. Brian Goetz's formula:

```
threads = N_cpu × U_cpu × (1 + W / C)
  N_cpu = cores, U_cpu = target CPU utilization (0..1)
  W = wait time per task, C = compute time per task
Example: 8 cores, 80% target, a call waits 90 ms and computes 10 ms
  → 8 × 0.8 × (1 + 90/10) = 64 threads
```
- **Little's law:** `L = λ × W` — average number of requests in the system = arrival rate × average time in system. If you receive 200 req/s and each takes 0.5 s, about **100** requests are in flight → you need ~100 concurrent slots (threads, connections) to avoid queuing. Use it to size pools *and* downstream connection pools.

**Practical rules:**
1. Separate pools per workload (CPU vs blocking I/O vs scheduler) — a slow dependency must not starve unrelated work (the **bulkhead** pattern).
2. Downstream limits win: a pool of 200 threads is pointless if the DB pool has 20 connections.
3. Use a **bounded queue** and a rejection policy so overload is visible.
4. Measure: watch `getActiveCount()`, `getQueue().size()`, `getCompletedTaskCount()`, task latency percentiles; load test and adjust.
5. For huge I/O fan-out, virtual threads (Java 21) remove the need for sizing blocking pools; bound the *resource* (e.g. `Semaphore`) instead.

```java
int n = Runtime.getRuntime().availableProcessors();
ThreadPoolExecutor cpu = new ThreadPoolExecutor(n, n, 0, TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(1000), namedFactory("cpu"), new ThreadPoolExecutor.CallerRunsPolicy());
```

**Pitfalls / follow-up probes:**
- Blocking tasks submitted to a tiny CPU pool → starvation; tasks that wait on other tasks in the *same* pool → **thread-starvation deadlock**.
- Hyper-threads count as "cores" in `availableProcessors`, but give < 2× real throughput.
- Formulas give a starting point; the real answer comes from profiling.

**Remember:** *CPU-bound ≈ N; I/O-bound ≈ N·U·(1+W/C); Little's law: L = λW. Measure, then tune.*

---

### Q169. `Runnable` vs `Callable`, and `invokeAll` / `invokeAny` — how do you run a batch of tasks?

**Say this first:** `Runnable` returns nothing and cannot throw checked exceptions; `Callable<V>` returns a value and may throw. `invokeAll` waits for all results, `invokeAny` returns the first success.

**Answer:**
| | `Runnable` | `Callable<V>` |
|---|---|---|
| Method | `void run()` | `V call() throws Exception` |
| Result / checked exception | No | Yes (via `Future.get()` → `ExecutionException`) |
| Used with | `Thread`, `execute`, `submit` | `submit`, `invokeAll`, `invokeAny` |

**`invokeAll`** — submits all, **blocks until all are done** (or timeout), returns `List<Future<T>>` in the *same order as the input*:
```java
List<Callable<Quote>> jobs = vendors.stream().<Callable<Quote>>map(v -> () -> v.quote(item)).toList();
List<Future<Quote>> futures = pool.invokeAll(jobs, 2, TimeUnit.SECONDS);   // on timeout, unfinished tasks are cancelled
for (Future<Quote> f : futures) {
    if (f.isCancelled()) continue;               // timed out
    try { use(f.get()); }
    catch (ExecutionException e) { log.warn("vendor failed", e.getCause()); }
}
```
**`invokeAny`** — returns the result of **one task that completes successfully**, and cancels the rest. If all fail → `ExecutionException`. Useful for "ask 3 mirrors, take the fastest".
```java
Quote fastest = pool.invokeAny(jobs, 1, TimeUnit.SECONDS);   // throws TimeoutException if none finishes in time
```
**Alternative — `ExecutorCompletionService`:** gives results in **completion order**, so you process the fastest first while others still run.
```java
CompletionService<Quote> cs = new ExecutorCompletionService<>(pool);
jobs.forEach(cs::submit);
for (int i = 0; i < jobs.size(); i++) use(cs.take().get());
```

**Pitfalls / follow-up probes:**
- `Future.get()` without a timeout can hang forever; always use a timeout at service boundaries.
- `ExecutionException.getCause()` is the real exception.
- `invokeAll` with a **single-thread** pool runs sequentially — it is not magic parallelism.
- `submit(Runnable)` returns a `Future<?>` whose `get()` yields `null` — still useful for exception propagation/cancellation.
- Lambdas: `() -> 42` is a `Callable` only if the context needs a return value; `() -> { doIt(); }` is a `Runnable` — overload ambiguity may need a cast.

**Remember:** *Callable returns + throws. invokeAll = wait for everyone, invokeAny = first winner, CompletionService = results as they finish.*

---

### Q170. How does `ForkJoinPool` work (work stealing, `RecursiveTask`), and what are the common-pool and parallel-stream pitfalls?

**Say this first:** Fork/Join splits a big problem into small subtasks, runs them on a few workers, and idle workers **steal** work from busy ones' queues so all cores stay busy.

**Answer:**
- **Divide & conquer:** `fork()` pushes a subtask onto the current worker's own **deque** (double-ended queue); `join()` waits for its result. A worker takes its own tasks from one end (LIFO, cache-friendly); a thieving worker steals from the *other* end (FIFO, usually bigger chunks) → low contention.
- **`RecursiveTask<V>`** returns a value; **`RecursiveAction`** returns nothing.

```java
class SumTask extends RecursiveTask<Long> {
    private static final int THRESHOLD = 10_000;
    private final long[] a; private final int lo, hi;
    SumTask(long[] a, int lo, int hi) { this.a = a; this.lo = lo; this.hi = hi; }

    @Override protected Long compute() {
        if (hi - lo <= THRESHOLD) { long s = 0; for (int i = lo; i < hi; i++) s += a[i]; return s; }
        int mid = (lo + hi) >>> 1;
        SumTask left = new SumTask(a, lo, mid), right = new SumTask(a, mid, hi);
        left.fork();                          // run left asynchronously
        long r = right.compute();             // run right in THIS thread (don't fork both)
        return left.join() + r;
    }
}
long total = ForkJoinPool.commonPool().invoke(new SumTask(data, 0, data.length));
```
- **Threshold** matters: too small → task overhead dominates; too big → no parallelism.
- **Common pool:** `ForkJoinPool.commonPool()` is a JVM-wide shared pool, parallelism ≈ cores − 1 (the calling thread also helps). Used by **`parallelStream()`**, `CompletableFuture.supplyAsync` (without an executor), and `Arrays.parallelSort`.

**Pitfalls (very commonly asked):**
- **Blocking in the common pool** (JDBC, HTTP, `sleep`) in a `parallelStream` or `supplyAsync` starves every other user of the pool, application-wide. Use your own executor for blocking work.
- Parallel streams are only worth it for **large, CPU-bound, stateless** pipelines on easily splittable sources (`ArrayList`, arrays, ranges). `LinkedList`, `Stream.iterate`, and small data are usually slower.
- Lambdas in parallel streams must be **stateless, non-interfering and associative**; don't mutate shared lists (`forEach(list::add)` on an `ArrayList` is a race) — use `collect`.
- `forEachOrdered`/`sorted` limit parallelism; `findFirst` vs `findAny` differ in cost.
- To limit parallelism for one stream, run it inside a custom `new ForkJoinPool(n).submit(...)` (works in practice, though it is not a documented guarantee) or use an executor yourself.
- Don't `fork` both halves then `join` in the wrong order — fork one, compute the other, then join.

**Remember:** *Split, fork, steal, join. Keep the common pool for short CPU work only — never block in it.*

---

### Q171. Why must `wait()`/`notify()` be inside `synchronized` and called in a `while` loop? Also `sleep` vs `wait`, and producer–consumer with `wait/notify`.

**Say this first:** `wait()` atomically *releases the monitor and sleeps*; `notify()` wakes a sleeper. Both need the monitor so the "condition check → wait" is one atomic step, and the wake-up must be re-checked in a loop because waking does not guarantee the condition is true.

**Answer — the rules:**
1. Call `wait/notify/notifyAll` only while **holding the object's monitor**, otherwise `IllegalMonitorStateException`. Reason: without the lock, a **lost wake-up** is possible — thread A checks "queue empty", then B adds an item and notifies *before* A starts waiting; A then sleeps forever.
2. Always wait in a **`while` loop**, never `if`: **spurious wakeups** (the JVM/OS may wake a thread with no notify) and **stolen conditions** (another consumer woke first and took the item) mean the condition may be false again.
3. Prefer `notifyAll()` unless you are sure only one waiter can proceed and all waiters are equivalent; `notify()` can wake the wrong kind (a producer instead of a consumer) → hang.

| | `Thread.sleep(ms)` | `Object.wait()` |
|---|---|---|
| Class | `Thread` (static) | `Object` (instance) |
| Releases the monitor? | **No** | **Yes** |
| Needs `synchronized`? | No | **Yes** |
| Woken by | Timeout / interrupt | `notify`/`notifyAll`, timeout, interrupt, (spurious) |
| Purpose | Pause | Inter-thread coordination |

**Producer–consumer with `wait/notify` (bounded buffer):**
```java
class Buffer<T> {
    private final Queue<T> q = new LinkedList<>();
    private final int cap;
    Buffer(int cap) { this.cap = cap; }

    public synchronized void put(T item) throws InterruptedException {
        while (q.size() == cap) wait();      // full → release monitor and sleep
        q.add(item);
        notifyAll();                         // wake consumers (and producers; they re-check)
    }
    public synchronized T take() throws InterruptedException {
        while (q.isEmpty()) wait();          // empty → wait
        T item = q.remove();
        notifyAll();                         // space freed → wake producers
        return item;
    }
}
// Producer thread: for (...) buf.put(i);     Consumer thread: buf.take();
```
- Each wake-up re-evaluates its `while` condition: if still false it simply waits again.

**Pitfalls / follow-up probes:**
- `wait()` on one object while holding another lock keeps the *other* lock → deadlock risk.
- Using `if` instead of `while` is the #1 bug interviewers look for.
- Modern code should use `BlockingQueue` or `Condition`; hand-written `wait/notify` is for understanding.

**Remember:** *Lock, `while (!cond) wait();`, change state, `notifyAll()`. Sleep holds the lock, wait lets go.*

---

### Q172. Implement producer–consumer with a `BlockingQueue`. How do you stop the consumers cleanly?

**Say this first:** `BlockingQueue` already does the locking and waiting: `put()` blocks when full, `take()` blocks when empty. You only write the business logic — and a shutdown strategy.

**Answer:**
- Main implementations: `ArrayBlockingQueue` (bounded, array, one lock), `LinkedBlockingQueue` (optionally bounded, two locks → higher throughput), `PriorityBlockingQueue` (unbounded, ordered), `DelayQueue`, `SynchronousQueue` (zero capacity hand-off), `LinkedTransferQueue`.
- Method families: `put/take` (block), `offer/poll` (non-blocking or with timeout), `add/remove` (throw).
- A **bounded** queue gives **back-pressure**: fast producers are slowed down instead of exhausting memory.

**Full example with a poison pill (a special sentinel telling consumers to stop):**
```java
import java.util.concurrent.*;

public class ProducerConsumer {
    private static final String POISON = "__STOP__";   // compared by identity

    public static void main(String[] args) throws Exception {
        BlockingQueue<String> queue = new ArrayBlockingQueue<>(10);
        int consumers = 3;
        ExecutorService pool = Executors.newFixedThreadPool(consumers + 1);

        pool.submit(() -> {                                // single producer
            try {
                for (int i = 1; i <= 20; i++) queue.put("job-" + i);
                for (int i = 0; i < consumers; i++) queue.put(POISON);   // one pill per consumer
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        });

        for (int c = 0; c < consumers; c++) {
            pool.submit(() -> {
                try {
                    while (true) {
                        String job = queue.take();
                        if (job == POISON) break;          // identity check on sentinel
                        System.out.println(Thread.currentThread().getName() + " handled " + job);
                    }
                } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            });
        }
        pool.shutdown();
        pool.awaitTermination(1, TimeUnit.MINUTES);
    }
}
```
**Other stop strategies:** interrupt consumers (`shutdownNow`), use `poll(timeout)` and check a `volatile` flag, or close via a sentinel per consumer (as above).

**Pitfalls / follow-up probes:**
- Using `String` literal as the sentinel and comparing with `==` works only for the same instance — use a dedicated `new Object()`/typed marker in real code.
- Unbounded queue + faster producer = `OutOfMemoryError`.
- `poll()` busy-looping without timeout burns CPU.
- Multiple producers: send the poison pills only after **all** producers are done (e.g. a `CountDownLatch`/`Phaser`).
- Ordering and failure: if a consumer dies, the item taken is lost unless you handle retries/acks.

**Remember:** *Bounded BlockingQueue = built-in back-pressure; one poison pill per consumer to stop.*

---

### Q173. Print odd/even numbers (and A-B-C) alternately using threads — write the code.

**Say this first:** It's a **turn-taking** problem. Keep shared state saying whose turn it is; each thread waits until it's its turn, acts, then passes the turn.

**Answer — Odd/Even with `wait/notify`:**
```java
public class OddEven {
    private int n = 1;
    private final int max = 10;

    synchronized void print(boolean odd) throws InterruptedException {
        while (n <= max) {
            while (n <= max && (n % 2 == 1) != odd) wait();   // not my turn
            if (n > max) break;
            System.out.println(Thread.currentThread().getName() + " : " + n++);
            notifyAll();                                      // hand the turn over
        }
        notifyAll();                                          // let the other thread exit too
    }

    public static void main(String[] args) {
        OddEven o = new OddEven();
        new Thread(() -> { try { o.print(true);  } catch (InterruptedException e) { } }, "odd").start();
        new Thread(() -> { try { o.print(false); } catch (InterruptedException e) { } }, "even").start();
    }
}
```
**A-B-C repeated 5 times with `Semaphore`s** (cleaner for N threads — each thread waits on its own permit and releases the next one's):
```java
import java.util.concurrent.Semaphore;

public class PrintABC {
    public static void main(String[] args) {
        Semaphore a = new Semaphore(1), b = new Semaphore(0), c = new Semaphore(0);
        new Thread(() -> loop("A", a, b)).start();
        new Thread(() -> loop("B", b, c)).start();
        new Thread(() -> loop("C", c, a)).start();
    }
    static void loop(String s, Semaphore mine, Semaphore next) {
        try {
            for (int i = 0; i < 5; i++) {
                mine.acquire();                // wait for my turn
                System.out.print(s);
                next.release();                // give turn to next thread
            }
        } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}
// prints ABCABCABCABCABC
```
**Alternative with `ReentrantLock` + `Condition`s:** one lock, one `Condition` per thread plus an `int turn`; each does `while (turn != me) cond[me].await(); … turn = next; cond[next].signal();` — avoids waking threads who can't proceed.

**Why this problem is asked:** checks your understanding of `wait` in a loop, `notifyAll`, state-driven coordination, and termination (the extra `notifyAll()` after the loop prevents the other thread from waiting forever).

**Pitfalls / follow-up probes:**
- `notify()` instead of `notifyAll()` with 3+ threads can wake the wrong thread → hang.
- Two threads with `if` instead of `while` → spurious wake-up prints out of order.
- Use `Thread.join()` or an executor to wait for the printers.
- A `volatile int turn` with busy-spinning works but wastes CPU.

**Remember:** *Shared `turn` + wait until it's mine + do + pass the baton. Semaphores chain turns elegantly.*

---

### Q174. What are livelock, starvation and priority inversion — and how do lock ordering and `tryLock` timeouts avoid deadlock?

**Say this first:** Deadlock isn't the only liveness failure: threads can be alive but making no progress (livelock), never get a turn (starvation), or be delayed by a lower-priority thread (priority inversion).

**Answer:**
| Problem | What happens | Example |
|---|---|---|
| **Deadlock** | Threads block forever waiting on each other's locks | A holds L1 wants L2; B holds L2 wants L1 |
| **Livelock** | Threads keep *reacting* to each other and never progress; CPU busy | Two people in a corridor sidestepping the same way; two threads both `tryLock`, back off, retry in lockstep |
| **Starvation** | A thread never gets the resource because others always win | Unfair lock; low-priority thread; greedy readers starving a writer; one task hogging a small pool |
| **Priority inversion** | High-priority thread waits for a lock held by a low-priority thread that is itself preempted by medium-priority work | Famous Mars Pathfinder resets; Java doesn't do priority inheritance for monitors, so avoid relying on priorities |

**Avoiding deadlock — four Coffman conditions** (mutual exclusion, hold-and-wait, no preemption, circular wait): break any one.
1. **Global lock ordering** — always acquire in the same order (e.g. by account id, or `System.identityHashCode` with a tie-breaker lock).
```java
void transfer(Account from, Account to, long amt) {
    Account first = from.id < to.id ? from : to;
    Account second = first == from ? to : from;
    synchronized (first) { synchronized (second) { from.debit(amt); to.credit(amt); } }
}
```
2. **`tryLock` with timeout** — give up and back off rather than waiting forever:
```java
boolean transfer(Account a, Account b, long amt) throws InterruptedException {
    long deadline = System.nanoTime() + TimeUnit.MILLISECONDS.toNanos(100);
    while (true) {
        if (a.lock.tryLock()) {
            try {
                if (b.lock.tryLock()) {
                    try { a.debit(amt); b.credit(amt); return true; } finally { b.lock.unlock(); }
                }
            } finally { a.lock.unlock(); }
        }
        if (System.nanoTime() > deadline) return false;
        Thread.sleep(ThreadLocalRandom.current().nextInt(1, 10));   // RANDOM backoff avoids livelock
    }
}
```
3. **Hold fewer locks / smaller scopes**; do I/O outside locks; use lock-free or concurrent structures.
4. **Open calls** — don't invoke foreign code while holding a lock.

**Fixes for the others:** randomized backoff (livelock); fair locks, bounded queues, separate pools per workload (starvation); avoid shared locks between priority levels (inversion).

**Pitfalls / follow-up probes:**
- `tryLock` retry without random jitter is the classic accidental livelock.
- Detect deadlock: `jstack`/thread dump prints "Found one Java-level deadlock"; programmatically `ThreadMXBean.findDeadlockedThreads()` (see the diagnosis question and the earlier deadlock question).
- Don't depend on `Thread.setPriority` for correctness — it is only a hint.

**Remember:** *Deadlock = stuck, livelock = busy but stuck, starvation = never picked. Order locks, use timeouts, add jitter.*

---

### Q175. Virtual threads (Java 21): when do they help, what is pinning, and what about structured concurrency and `ScopedValue`?

**Say this first:** Virtual threads are cheap, JVM-scheduled threads, so you can run **one thread per blocking task** (millions possible). They make *I/O-bound* code scale; they do nothing for *CPU-bound* work.

**Answer:**
- A **platform thread** = one OS thread (~1 MB stack, costly to create, a few thousand max). A **virtual thread** is a lightweight object whose stack lives on the heap; the JVM mounts it on a small pool of **carrier threads** (a `ForkJoinPool`, ~ #cores). When it blocks on I/O, `sleep`, `lock`, or `BlockingQueue`, it **unmounts** and the carrier runs another virtual thread.
```java
try (ExecutorService ex = Executors.newVirtualThreadPerTaskExecutor()) {   // close() waits for tasks
    for (int i = 0; i < 10_000; i++) {
        ex.submit(() -> { Thread.sleep(Duration.ofSeconds(1)); return fetch(); });
    }
}
Thread.ofVirtual().name("vt-1").start(() -> work());      // or Thread.startVirtualThread(r)
```
- Write plain **blocking, synchronous** code — no callbacks/reactive chains — and still scale like async.

**When they do NOT help:**
- **CPU-bound** work: bounded by cores, not by waiting; no gain (and no more cores appear).
- Code limited by a downstream resource (DB pool of 20): 10,000 virtual threads just queue. Limit with a `Semaphore` or the pool itself.
- **Don't pool them** — create one per task. Don't use `ThreadLocal` heavily (millions of copies); virtual threads are always daemon; priority is ignored.

**Pinning:** a virtual thread is *pinned* to its carrier when it cannot unmount, so the carrier stays blocked.
- Causes: blocking while inside a **`synchronized`** block/method (**JDK 21–23**; fixed in **JDK 24** by JEP 491), and while a **native method / foreign call** is on the stack (still pins).
- Effect: with few carriers, many pinned threads can stall throughput. Mitigations on 21–23: use `ReentrantLock` for long blocking sections; diagnose with `-Djdk.tracePinnedThreads=full` or JFR event `jdk.VirtualThreadPinned`.

**Structured concurrency & `ScopedValue` (preview/evolving — check your JDK):**
- **Structured concurrency** (`StructuredTaskScope`): treat a group of subtasks as one unit with a clear lifetime — if one fails, siblings are cancelled; the parent can't finish before children, so no leaked threads. Preview API in Java 21 through at least 25, with the API shape changing between releases — verify the current JEP/status before using or quoting exact method names.
- **`ScopedValue`**: immutable, scope-bound alternative to `ThreadLocal` for passing context (user, trace id) to callees and child tasks; cheap with virtual threads. Preview in 21–24; I believe finalized in 25 — check your JDK.

**Pitfalls / follow-up probes:**
- Virtual threads are not faster per task; they raise *throughput under blocking concurrency*.
- Thread dumps with millions of threads: use `jcmd <pid> Thread.dump_to_file -format=json file`.
- `Thread.currentThread()` inside works as usual; `Executors.newVirtualThreadPerTaskExecutor()` is not a pool.

**Remember:** *Virtual threads = cheap blocking; one per task; don't pool; useless for CPU-bound; avoid blocking inside `synchronized` before JDK 24.*

---

### Q176. `CompletableFuture` follow-ups: `thenApply` vs `thenCompose` vs `thenCombine`, `allOf`/`anyOf`, timeouts, custom executors, and exception handling.

**Say this first:** Think `map` (`thenApply`), `flatMap` (`thenCompose`), and "zip two independent futures" (`thenCombine`). Always pass your own executor for blocking work and always add a timeout and an error path.

**Answer:**
| Method | Use when | Analogy |
|---|---|---|
| `thenApply(f)` | `f` is a plain function `T → U` | `map` |
| `thenCompose(f)` | `f` itself returns a `CompletableFuture<U>` (dependent async call) | `flatMap` (avoids `CF<CF<U>>`) |
| `thenCombine(other, bf)` | Two **independent** futures, combine both results | zip |
| `thenAccept` / `thenRun` | Consume the result / just run something | `forEach` |
| `allOf(f1, f2, …)` | Wait for all; returns `CF<Void>` — read each future's result afterwards | join |
| `anyOf(f1, f2, …)` | First to complete (success **or failure**); returns `CF<Object>` | race |

```java
CompletableFuture<User>  u = supplyAsync(() -> userSvc.find(id), ioPool);
CompletableFuture<Order> o = u.thenCompose(user -> supplyAsync(() -> orderSvc.last(user), ioPool)); // dependent
CompletableFuture<Prefs> p = supplyAsync(() -> prefSvc.get(id), ioPool);                            // independent
CompletableFuture<Page>  page = o.thenCombine(p, Page::new);

CompletableFuture<Void> all = CompletableFuture.allOf(a, b, c);
all.thenRun(() -> System.out.println(a.join() + b.join() + c.join()));   // join() is safe: already done
```
**Timeouts (Java 9+):**
```java
page.orTimeout(2, TimeUnit.SECONDS)                       // fails with TimeoutException
    .completeOnTimeout(Page.EMPTY, 2, TimeUnit.SECONDS);  // or: substitute a default instead of failing
```
**Executors:** methods ending in `Async` without an executor use the **common pool**. Non-`Async` callbacks (`thenApply`) run on whichever thread completes the previous stage (or the caller if already done). Pass an executor to `…Async(fn, executor)` for blocking or heavy stages.

**Exception handling:**
```java
cf.exceptionally(ex -> fallback)                     // recover only on failure
  .handle((val, ex) -> ex == null ? val : fallback)  // success AND failure; can transform
  .whenComplete((val, ex) -> log(val, ex));          // side effect; does not change the result
```
- Exceptions travel downstream **skipping** `thenApply` stages until a `handle/exceptionally`.
- `join()` throws unchecked `CompletionException`; `get()` throws checked `ExecutionException`; the real error is `getCause()`. In `exceptionally`, `ex` may itself be a `CompletionException` wrapping the cause.

**Pitfalls / follow-up probes:**
- A stage that does blocking I/O inside the common pool starves others.
- `allOf` doesn't fail-fast for you: it completes after all complete; if one fails, it completes exceptionally (after all finish).
- `cf.cancel(true)` does **not** interrupt the running work (`CompletableFuture` can't control the thread).
- Don't call `get()`/`join()` inside the pool that must run the dependency — deadlock/starvation.
- Forgetting an error handler = silently lost exception when no one calls `join()`.

**Remember:** *apply = map, compose = flatMap, combine = zip; give it an executor, a timeout and an error path.*

---

### Q177. How would you build a thread-safe cache, a rate limiter, and a bounded buffer?

**Say this first:** Pick the simplest correct primitive for each: `ConcurrentHashMap.computeIfAbsent` for the cache, a token bucket under one lock (or `Semaphore`) for the rate limiter, and `ReentrantLock` with two `Condition`s for the bounded buffer.

**Answer:**

**1) Thread-safe cache (memoizing).**
```java
class Cache<K, V> {
    private final ConcurrentHashMap<K, V> map = new ConcurrentHashMap<>();
    private final Function<K, V> loader;
    Cache(Function<K, V> loader) { this.loader = loader; }

    V get(K key) { return map.computeIfAbsent(key, loader); }   // loader runs at most once per key
}
```
- `computeIfAbsent` is atomic per key; the loader must be **short and must not touch the same map** (can deadlock or throw). For slow loads, store `Future`/`CompletableFuture` values so waiters join the same in-flight load:
```java
ConcurrentMap<K, CompletableFuture<V>> m = new ConcurrentHashMap<>();
V get(K k) { return m.computeIfAbsent(k, x -> CompletableFuture.supplyAsync(() -> load(x), ioPool)).join(); }
```
- Real caches need **eviction** (LRU via `LinkedHashMap` + lock, or just use Caffeine), TTL, size bounds, and negative caching. Don't hand-roll for production.

**2) Rate limiter (token bucket).**
```java
class TokenBucket {
    private final long capacity, refillPerSec;
    private double tokens; private long last = System.nanoTime();
    TokenBucket(long capacity, long refillPerSec) { this.capacity = capacity; this.refillPerSec = refillPerSec; this.tokens = capacity; }

    synchronized boolean tryAcquire() {
        long now = System.nanoTime();
        tokens = Math.min(capacity, tokens + (now - last) / 1e9 * refillPerSec);   // refill lazily
        last = now;
        if (tokens >= 1) { tokens -= 1; return true; }
        return false;                                                              // reject (or compute wait time)
    }
}
```
- Variants: fixed window (simple, bursty at edges), sliding window, leaky bucket (smooths output). A `Semaphore` limits *concurrent* calls (a bulkhead), which differs from limiting *rate*. Across multiple JVMs you need a shared store (e.g. Redis) — a per-JVM limiter limits only that node.

**3) Bounded buffer with `Lock` + `Condition`s.**
```java
class BoundedBuffer<T> {
    private final Object[] items; private int head, tail, count;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition(), notEmpty = lock.newCondition();
    BoundedBuffer(int n) { items = new Object[n]; }

    void put(T x) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length) notFull.await();
            items[tail] = x; tail = (tail + 1) % items.length; count++;
            notEmpty.signal();                      // only a consumer can proceed
        } finally { lock.unlock(); }
    }
    @SuppressWarnings("unchecked")
    T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) notEmpty.await();
            T x = (T) items[head]; items[head] = null; head = (head + 1) % items.length; count--;
            notFull.signal();                       // only a producer can proceed
            return x;
        } finally { lock.unlock(); }
    }
}
```
(This is how `ArrayBlockingQueue` works; in real code just use it.)

**Pitfalls / follow-up probes:**
- Cache stampede: many threads load the same missing key — solve with `computeIfAbsent` or in-flight futures.
- Check-then-act on a `ConcurrentHashMap` using `get` then `put` is a race.
- A `synchronized` limiter becomes a bottleneck at huge rates; shard it or use `AtomicLong` with CAS.

**Remember:** *Cache = atomic compute per key; rate limiter = token bucket; bounded buffer = lock + notFull/notEmpty.*

---

### Q178. How do you diagnose and test concurrency bugs? (thread dumps, `jstack`/`jcmd`, JFR, stress tests)

**Say this first:** Concurrency bugs are timing-dependent, so you *observe* a live process (thread dumps, profiling) and *provoke* the bug with stress tests, instead of hoping a normal test fails.

**Answer — diagnosing a live/hung JVM:**
1. **Thread dump** — a snapshot of every thread's state and stack. Take **3 dumps ~5–10 s apart** and compare: threads in the same place every time are stuck.
   - `jcmd <pid> Thread.print` (preferred), `jstack <pid>`, `kill -3 <pid>` (Linux; prints to stdout/log), or `ThreadMXBean.dumpAllThreads`. In Java 21: `jcmd <pid> Thread.dump_to_file -format=json out.json` (includes virtual threads).
2. **Read the states:**
   - Many `BLOCKED ... waiting to lock <0x...>` on one address → lock contention; find the thread `locked <0x...>` and see what it is doing (slow I/O inside a lock?).
   - Many `WAITING (parking)` on a pool/queue → idle workers, or pool exhaustion, or waiting for a DB connection.
   - A "Found one Java-level deadlock" section → a lock cycle.
   - A `RUNNABLE` thread at 100% CPU in a loop → spin/livelock (cross-check with `top -H -p <pid>` → convert the thread id to hex → match `nid=0x...`).
3. **JFR (Java Flight Recorder)** — low-overhead continuous profiler: `jcmd <pid> JFR.start duration=60s filename=rec.jfr`, open in JDK Mission Control. Events like `jdk.JavaMonitorEnter`, `jdk.ThreadPark`, `jdk.VirtualThreadPinned` show lock waits and contention.
4. **Metrics:** executor queue size, active count, rejected count, pool-wait time, lock wait time; heap dump if queues grow unbounded.

**Programmatic deadlock detection:**
```java
ThreadMXBean mx = ManagementFactory.getThreadMXBean();
long[] ids = mx.findDeadlockedThreads();            // null if none
if (ids != null) for (ThreadInfo ti : mx.getThreadInfo(ids, true, true)) log.error("{}", ti);
```

**Testing concurrent code:**
- **Stress / repetition:** many threads, many iterations, a `CountDownLatch` start gate so all threads begin together, assert invariants at the end (e.g. counter == threads × iterations).
```java
@Test void counterIsExact() throws Exception {
    int threads = 16, iters = 100_000;
    Counter c = new Counter();
    CountDownLatch go = new CountDownLatch(1);
    ExecutorService ex = Executors.newFixedThreadPool(threads);
    List<Future<?>> fs = new ArrayList<>();
    for (int i = 0; i < threads; i++) fs.add(ex.submit(() -> { go.await(); for (int j = 0; j < iters; j++) c.inc(); return null; }));
    go.countDown();
    for (Future<?> f : fs) f.get(30, TimeUnit.SECONDS);   // timeout → a hung test fails instead of hanging CI
    ex.shutdown();
    assertEquals((long) threads * iters, c.get());
}
```
- Use **timeouts** on every wait; never `Thread.sleep` to "wait for the other thread" — use latches, `Awaitility`, or `Future.get`.
- Tools: **jcstress** (OpenJDK harness for memory-model tests), **JMH** for benchmarks, ThreadSanitizer-like tools aren't available for JVM by default; static analysis (SpotBugs, `@GuardedBy` checks), code review for shared mutable state.
- Design for testability: inject the `Executor`/`Clock`; in unit tests use a same-thread executor to make logic deterministic, and separate tests for the concurrency itself.

**Pitfalls / follow-up probes:**
- A passing test proves little — races may appear once in a million runs; run in a loop and on a different CPU count.
- Adding logging/`System.out` can hide a race (changes timing, adds a lock).
- Debuggers pause threads and change interleavings; thread dumps are less intrusive.

**Remember:** *Dump three times and compare; JFR for contention; stress with a start gate and timeouts.*

---

# 7. Design Patterns & Design Principles

### Q179. Explain SOLID with examples from code you've written.

| Principle | Meaning | Violation smell | Fix |
|---|---|---|---|
| **S**ingle Responsibility | One reason to change | 2000-line `OrderService` doing validation + pricing + email + PDF | Split into `OrderValidator`, `PricingEngine`, `NotificationService` |
| **O**pen/Closed | Extend without modifying | `if (type == UPI) … else if (type == CARD) …` growing forever | Strategy: `Map<PaymentType, PaymentHandler>` injected by Spring |
| **L**iskov Substitution | Subtypes must honour the contract | `SquareRectangle` breaking `setWidth`; a subclass throwing `UnsupportedOperationException` | Prefer composition |
| **I**nterface Segregation | Small, focused interfaces | `UserService` with 30 methods; implementers stubbing half | Split `UserReader` / `UserWriter` |
| **D**ependency Inversion | Depend on abstractions | `new MySqlUserRepo()` inside a service | Constructor-inject the interface — this is literally Spring DI |

**Bonus one-liner:** "Spring is an SOLID enabler — DI gives you D, interface-driven beans give you O and I."

**Remember:** **SOLID** = *Single, Open, Liskov, Interface, Dependency.*

---

### Q180. Simple Factory vs Factory Method vs Abstract Factory — what is the difference and when do you use each?

**Say this first:** "All three hide `new`. Simple Factory is one method with a switch, Factory Method lets a subclass decide, Abstract Factory creates a whole *family* of related objects."

**Answer:**
- **Why factories at all:** `new ConcreteClass()` glues the caller to one implementation. A factory moves the "which class?" decision to one place, so callers depend only on an interface.
- **Simple Factory** (not a GoF pattern, just a static helper) — one method picks the class by a parameter.
- **Factory Method** (GoF) — the base class declares an abstract `create…()` method; each subclass overrides it to return its own product. The base class uses the product without knowing the concrete type.
- **Abstract Factory** (GoF) — an interface with several `create…()` methods that return products that must work *together* (same family). Swap the factory and the whole family changes.
- **JDK examples:** `Calendar.getInstance()`, `List.of(...)`, `Executors.newFixedThreadPool(n)` (static factory / simple factory); `Collection.iterator()` (Factory Method — each collection decides its iterator); `DocumentBuilderFactory`, `TransformerFactory` (Abstract Factory).

```java
// 1) Simple Factory
final class NotifierFactory {
    static Notifier of(Channel c) {
        return switch (c) {
            case EMAIL -> new EmailNotifier();
            case SMS   -> new SmsNotifier();
        };                                   // new channel = edit this switch
    }
}

// 2) Factory Method — subclass decides the product
abstract class ReportExporter {
    protected abstract Writer openWriter() throws IOException;   // the factory method
    public final void export(Report r) throws IOException {
        try (Writer w = openWriter()) { w.write(r.render()); }   // base code is product-agnostic
    }
}
class FileExporter extends ReportExporter {
    protected Writer openWriter() throws IOException { return Files.newBufferedWriter(Path.of("r.txt")); }
}
class ConsoleExporter extends ReportExporter {
    protected Writer openWriter() { return new PrintWriter(System.out) { @Override public void close() { flush(); } }; }
}

// 3) Abstract Factory — a family of DAOs that must match
interface DaoFactory { UserDao userDao(); OrderDao orderDao(); }
class JdbcDaoFactory implements DaoFactory {
    private final DataSource ds;
    JdbcDaoFactory(DataSource ds) { this.ds = ds; }
    public UserDao userDao()   { return new JdbcUserDao(ds); }
    public OrderDao orderDao() { return new JdbcOrderDao(ds); }
}
class InMemoryDaoFactory implements DaoFactory {          // great for tests
    public UserDao userDao()   { return new InMemoryUserDao(); }
    public OrderDao orderDao() { return new InMemoryOrderDao(); }
}
```

**When NOT to use:** if there is only one implementation and no real variation, a factory is just indirection. Prefer a plain constructor until a second variant appears.

**Pitfalls / follow-up probes:**
- Simple Factory violates Open/Closed (switch grows). Fix: `Map<Channel, Supplier<Notifier>>` registry, or `ServiceLoader`.
- Factory Method uses **inheritance**; if you only need to pass behaviour, pass a `Supplier<Writer>` instead — lighter.
- Abstract Factory is painful when you add a *new product type* (every factory must change).
- "Factory vs Builder?" — Factory picks *which* type; Builder assembles *one complex* object step by step.

**Remember:** *Simple = a switch; Method = subclass decides; Abstract = family of products.*

---

### Q181. Builder pattern — how do you write one by hand (no Lombok), with validation, and why use it for many optional parameters?

**Say this first:** "Builder replaces telescoping constructors with named setters, then validates once in `build()` and returns an immutable object."

**Answer:**
- **Problem:** a class with 8 parameters, 5 optional. Constructors `Order(a,b)`, `Order(a,b,c)`, `Order(a,b,c,d)`… explode, and `new Order("x", null, 0, null, true)` is unreadable (which `null` is which?). JavaBeans setters fix readability but leave the object **half-built and mutable** between calls.
- **Solution:** a nested `Builder` collects values with fluent, *named* methods. `build()` checks the rules (required fields, cross-field rules) and creates the final immutable object through a private constructor.
- **JDK examples:** `StringBuilder`, `Stream.builder()`, `HttpRequest.newBuilder()`, `ProcessBuilder`, `Locale.Builder`.

```java
public final class HttpCall {
    private final String url;
    private final String method;
    private final Map<String, String> headers;
    private final int timeoutMs;
    private final String body;

    private HttpCall(Builder b) {                       // only the builder can call this
        this.url = b.url;
        this.method = b.method;
        this.headers = Map.copyOf(b.headers);           // defensive copy => truly immutable
        this.timeoutMs = b.timeoutMs;
        this.body = b.body;
    }
    public static Builder builder(String url) { return new Builder(url); }   // required arg up front

    public static final class Builder {
        private final String url;
        private String method = "GET";                  // sensible defaults for optionals
        private final Map<String, String> headers = new LinkedHashMap<>();
        private int timeoutMs = 3_000;
        private String body;

        private Builder(String url) { this.url = Objects.requireNonNull(url, "url"); }
        public Builder method(String m)          { this.method = m; return this; }
        public Builder header(String k, String v){ headers.put(k, v); return this; }
        public Builder timeoutMs(int t)          { this.timeoutMs = t; return this; }
        public Builder body(String b)            { this.body = b; return this; }

        public HttpCall build() {                       // validate ONCE, at the end
            if (timeoutMs <= 0) throw new IllegalStateException("timeout must be > 0");
            if (body != null && "GET".equals(method)) throw new IllegalStateException("GET cannot have a body");
            return new HttpCall(this);
        }
    }
    // getters...
}

HttpCall call = HttpCall.builder("https://api.example.com/orders")
        .method("POST").header("Content-Type", "application/json").body("{}").timeoutMs(5_000).build();
```

**Records alternative (Java 16+):** records are great for few fields; for many optional fields combine a record with a hand-written builder, and validate in the **compact constructor**:
```java
record Range(int from, int to) { Range { if (from > to) throw new IllegalArgumentException("from > to"); } }
```

**When NOT to use:** classes with 2–3 required fields — a normal constructor or a record is clearer.

**Pitfalls / follow-up probes:**
- Validate in `build()` / the private constructor, **not** in setters (cross-field rules can only be checked at the end).
- Copy mutable inputs (lists, maps, dates) — otherwise the caller can still change your "immutable" object.
- A builder instance is **not thread-safe**; don't share it. Create a new one per object.
- Builder vs telescoping vs JavaBeans: Builder is the only one giving readability **and** immutability **and** validation.
- Lombok `@Builder` generates the same shape; knowing the manual version shows you understand what it generates.

**Remember:** *Named steps in, validated immutable object out.*

---

### Q182. Prototype pattern — `clone()`, copy constructors, and shallow vs deep copy.

**Say this first:** "Prototype creates new objects by copying an existing one, useful when construction is expensive. In Java prefer a copy constructor over `Object.clone()`."

**Answer:**
- **Problem:** building an object from scratch is costly (DB lookups, parsing a big config) or you need many near-identical objects with small differences (e.g. game units, document templates).
- **Idea:** keep a configured "prototype" and **copy** it, then tweak the copy.
- **Java's built-in way:** `Cloneable` + `Object.clone()`. It is widely considered **broken**: `Cloneable` has no `clone()` method, `clone()` is `protected`, it bypasses constructors, and it does a **shallow** copy by default.
- **Shallow copy** — copies field references; the copy and original share the same nested objects. **Deep copy** — nested mutable objects are copied too.

```java
final class Address {
    String city;
    Address(String city) { this.city = city; }
    Address(Address other) { this(other.city); }                 // copy constructor
}

final class Employee {
    private final String name;
    private final Address address;
    private final List<String> skills;

    Employee(String name, Address address, List<String> skills) {
        this.name = name; this.address = address; this.skills = skills;
    }
    // Deep copy via copy constructor — explicit, no Cloneable, works with final fields
    Employee(Employee other) {
        this(other.name, new Address(other.address), new ArrayList<>(other.skills));
    }
    Employee withName(String n) { return new Employee(n, new Address(address), new ArrayList<>(skills)); }
}

// Prototype registry
class EmployeeTemplates {
    private final Map<String, Employee> protos = new HashMap<>();
    void register(String key, Employee e) { protos.put(key, e); }
    Employee create(String key) { return new Employee(protos.get(key)); }   // fresh deep copy each time
}
```

**Shallow clone trap:**
```java
ArrayList<Address> a = new ArrayList<>(List.of(new Address("Pune")));
@SuppressWarnings("unchecked")
ArrayList<Address> b = (ArrayList<Address>) a.clone();   // new list, SAME Address objects
b.get(0).city = "Delhi";                                  // a.get(0).city is now "Delhi" too!
```

**Other ways to deep copy:** serialize/deserialize (slow, needs `Serializable`), JSON round trip (Jackson), or a copy constructor / static factory `copyOf` (best). Immutable objects don't need copying at all — share them.

**When NOT to use:** if the object is cheap to build or immutable, just call the constructor.

**Pitfalls / follow-up probes:**
- Arrays: `arr.clone()` is the one case where `clone()` is fine (but still shallow for object arrays).
- If you must use `clone()`: implement `Cloneable`, override `clone()` as `public`, call `super.clone()`, then deep-copy mutable fields manually. Does not work with `final` mutable fields.
- Subclass + `clone()` breaks easily; copy constructors don't have that problem.
- *Effective Java* Item 13: "Override clone judiciously" — i.e. usually don't.

**Remember:** *Copy constructor > clone(); shallow shares nested objects, deep doesn't.*

---

### Q183. How can a Singleton be broken (reflection, serialization, cloning, classloaders), and how do you guard against each?

**Say this first:** "A private constructor is only a convention. Reflection, deserialization and `clone()` can all create a second instance. An `enum` singleton is the only one that is safe against all of them."

**Answer:** (Basic thread-safe variants are covered in Q152 — here we focus on *breaking* and *guarding*.)

| Attack | How it breaks | Guard |
|---|---|---|
| **Reflection** | `Constructor.setAccessible(true)` then `newInstance()` | Throw in constructor if instance already exists; or use `enum` (JVM refuses) |
| **Serialization** | Each deserialization makes a **new** object | `readResolve()` returns the existing instance; or `enum` |
| **Cloning** | `clone()` on a `Cloneable` singleton | Don't implement `Cloneable`; or override `clone()` to throw |
| **Multiple classloaders** | Each classloader loads its own `Singleton` class → one instance *per loader* (app servers, plugins) | Put it in a shared parent classloader; understand "singleton per JVM" is really "per classloader" |

```java
public final class Config implements Serializable {
    private static final Config INSTANCE = new Config();

    private Config() {
        if (INSTANCE != null) {                         // blocks reflection AFTER first creation
            throw new IllegalStateException("Already initialised");
        }
    }
    public static Config getInstance() { return INSTANCE; }

    @Serial private Object readResolve() { return INSTANCE; }          // blocks serialization attack
    @Override protected Object clone() throws CloneNotSupportedException {
        throw new CloneNotSupportedException("Singleton");              // blocks clone
    }
}

// The attack
Constructor<Config> c = Config.class.getDeclaredConstructor();
c.setAccessible(true);
Config second = c.newInstance();      // throws IllegalStateException thanks to the guard above
```
Note the reflection guard above works for **eager** init. A lazy singleton can still be broken if the attacker calls the constructor *before* `getInstance()` — which is one more reason to prefer enum.

**The enum way (Effective Java Item 3):**
```java
public enum AppRegistry {
    INSTANCE;
    private final Map<String, Object> items = new ConcurrentHashMap<>();
    public void put(String k, Object v) { items.put(k, v); }
}
```
- Reflection: `Constructor.newInstance` on an enum throws `IllegalArgumentException: Cannot reflectively create enum objects`.
- Serialization: the JVM serializes enums by *name* and resolves back to the same constant.
- Cloning: `Enum.clone()` is `final` and throws.
- Limit: an enum cannot extend another class (it can implement interfaces), and the instance is created eagerly at class load.

**JDK examples:** `Runtime.getRuntime()`, `Desktop.getDesktop()`. Notice `Runtime` is a classic eager singleton.

**When NOT to use:** singletons are global state — hard to mock, hidden dependencies, order-of-init bugs. Prefer creating one object at startup and *passing* it to whoever needs it (see the DI-without-framework question).

**Pitfalls / follow-up probes:**
- Double-checked locking **without `volatile`** can expose a partly constructed object.
- Singleton holding mutable state needs its own thread-safety.
- Singletons make unit tests order-dependent (state leaks between tests).

**Remember:** *Reflection, serialization, clone, classloader — enum beats the first three.*

---

### Q184. Adapter and Facade — what is the difference, and when do you use each?

**Say this first:** "Adapter makes one interface *look like another*. Facade gives a *simpler front* to a complicated subsystem."

**Answer:**
- **Adapter** — you have a class with the *wrong* interface and you cannot (or don't want to) change it (third-party SDK, legacy code). You write a small wrapper that implements the interface your code expects and translates calls. **Why:** your core code stays clean and depends on *your* interface, not the vendor's.
- **Facade** — a subsystem has many classes and steps. A facade exposes one or two high-level methods (`placeOrder`) and calls the right classes in the right order. **Why:** fewer dependencies for clients, one place that knows the choreography.
- Key difference: Adapter changes the *interface* (1 wrapped object, same functionality). Facade changes the *complexity* (many wrapped objects, new simpler API).
- **JDK examples (Adapter):** `Arrays.asList(array)` (array → `List`), `InputStreamReader` (bytes `InputStream` → chars `Reader`), `Collections.enumeration(c)`, `Executors.callable(runnable)`. **Facade:** SLF4J is a logging facade over Logback/Log4j; `java.net.http.HttpClient` hides sockets, TLS, redirects.

```java
// ---- Adapter: legacy SDK -> our port
interface PaymentGateway { Receipt charge(String customerId, long amountPaise); }

class LegacyBankSdk {                                  // we can't change this
    int doTxn(String acct, double rupees, String curr) { /* ... */ return 0; }
}

class LegacyBankAdapter implements PaymentGateway {
    private final LegacyBankSdk sdk;
    LegacyBankAdapter(LegacyBankSdk sdk) { this.sdk = sdk; }
    public Receipt charge(String customerId, long amountPaise) {
        int code = sdk.doTxn(customerId, amountPaise / 100.0, "INR");   // translate units and names
        if (code != 0) throw new PaymentFailedException("code " + code);
        return new Receipt(customerId, amountPaise);
    }
}

// ---- Facade: one simple method over 4 subsystems
class OrderFacade {
    private final InventoryService inventory; private final PaymentGateway payment;
    private final ShippingService shipping;   private final NotificationService notify;
    OrderFacade(InventoryService i, PaymentGateway p, ShippingService s, NotificationService n) {
        this.inventory = i; this.payment = p; this.shipping = s; this.notify = n;
    }
    public OrderResult placeOrder(OrderRequest r) {
        inventory.reserve(r.items());
        Receipt rc = payment.charge(r.customerId(), r.totalPaise());
        String awb = shipping.schedule(r.address(), r.items());
        notify.orderPlaced(r.customerId(), awb);
        return new OrderResult(rc, awb);
    }
}
```

**When NOT to use:** don't add a facade that just forwards one call (pure indirection). Don't force clients through the facade if they genuinely need the fine-grained subsystem API — keep it optional.

**Pitfalls / follow-up probes:**
- Facade turning into a **God class** — keep it thin, orchestration only, no business rules.
- Adapter vs Decorator vs Proxy — Adapter *changes* the interface; Decorator and Proxy *keep* the same interface.
- Object adapter (composition, shown above) vs class adapter (inheritance) — Java prefers the object adapter.
- Adapter is the natural place to **translate exceptions and units** from third-party code.

**Remember:** *Adapter = translate interface; Facade = simplify subsystem.*

---

### Q185. Decorator pattern — how does `java.io` use it, and how do you write your own?

**Say this first:** "Decorator wraps an object with the *same interface* to add behaviour at runtime, without subclassing. `new BufferedReader(new InputStreamReader(...))` is the textbook example."

**Answer:**
- **Problem:** you want optional extras (buffering, compression, encryption, logging, retry). With inheritance you'd need a subclass for every combination: `BufferedGzipInputStream`, `BufferedEncryptedInputStream`, … — a class explosion.
- **Solution:** each extra is a small wrapper that implements the same interface, holds a delegate, and adds its behaviour before/after delegating. You **stack** wrappers in any order.
- **JDK:** `BufferedInputStream`, `GZIPInputStream`, `DataInputStream` wrap another `InputStream`. `Collections.unmodifiableList()`, `synchronizedList()`, `checkedList()` wrap a `List`. `FilterInputStream` is the abstract base decorator.

```java
// java.io — behaviours stacked at runtime
try (BufferedReader r = new BufferedReader(
        new InputStreamReader(new GZIPInputStream(new FileInputStream("log.gz")), StandardCharsets.UTF_8))) {
    String line = r.readLine();
}

// Your own: decorate a PaymentGateway with retry + metrics, same interface in and out
interface PaymentGateway { Receipt charge(String id, long amount); }

class RetryingGateway implements PaymentGateway {
    private final PaymentGateway delegate; private final int max;
    RetryingGateway(PaymentGateway d, int max) { this.delegate = d; this.max = max; }
    public Receipt charge(String id, long amount) {
        for (int attempt = 1; ; attempt++) {
            try { return delegate.charge(id, amount); }
            catch (TransientException e) { if (attempt == max) throw e; }
        }
    }
}
class TimedGateway implements PaymentGateway {
    private final PaymentGateway delegate;
    TimedGateway(PaymentGateway d) { this.delegate = d; }
    public Receipt charge(String id, long amount) {
        long t0 = System.nanoTime();
        try { return delegate.charge(id, amount); }
        finally { System.out.printf("charge took %d ms%n", (System.nanoTime() - t0) / 1_000_000); }
    }
}

PaymentGateway gw = new TimedGateway(new RetryingGateway(new BankGateway(), 3));   // order matters
```

**Decorator vs Proxy vs Adapter:**
- Decorator: **client chooses** the wrapping to *add* behaviour; may stack many.
- Proxy: *controls access* (lazy load, security, remote); usually the proxy creates/owns the real object and the client doesn't know.
- Adapter: *changes* the interface.

**When NOT to use:** if there is only one fixed extra, put it in the class. Deep stacks are hard to debug (stack traces get long).

**Pitfalls / follow-up probes:**
- Order matters: `Buffered(Gzip(file))` ≠ `Gzip(Buffered(file))`. Retry *outside* a timing decorator measures total time; inside measures each attempt.
- Always close the **outermost** wrapper (it closes the inner ones) — use try-with-resources.
- Interfaces with many methods make decorators verbose; extend an abstract `ForwardingX` base (Guava has `ForwardingList` etc.).
- `equals`/identity: the wrapper is not the same object as the delegate.

**Remember:** *Same interface, wrap and add — stack behaviours instead of subclasses.*

---

### Q186. Proxy pattern — static proxy, JDK dynamic proxy, and CGLIB/bytecode proxies.

**Say this first:** "A proxy is a stand-in with the same type as the real object that controls access to it. Frameworks use dynamic proxies for `@Transactional`, security, caching, lazy loading."

**Answer:**
- **Why:** to add something *around* a call without changing the real class or its callers: lazy initialisation (virtual proxy), access control (protection proxy), remote calls (remote proxy — RMI stubs), logging/caching/transactions.
- **Static proxy** — you hand-write the proxy class for one interface. Simple, but one class per interface, lots of repetitive forwarding.
- **JDK dynamic proxy** — `java.lang.reflect.Proxy` creates a proxy class **at runtime**; all calls go to one `InvocationHandler.invoke(proxy, method, args)`. Works **only for interfaces**.
- **CGLIB / ByteBuddy (concept)** — generate a **subclass** of the target at runtime and override methods. Works for classes without interfaces, but the class and methods can't be `final`, and it needs a visible constructor. Spring uses JDK proxies if the bean has an interface, CGLIB otherwise (Spring Boot defaults to CGLIB). Hibernate uses bytecode proxies for lazy entities.

```java
interface UserService { User find(long id); }

// Static proxy: lazy loading + caching
class CachingUserService implements UserService {
    private final Supplier<UserService> factory;        // expensive to create
    private UserService real;                            // created lazily
    private final Map<Long, User> cache = new HashMap<>();
    CachingUserService(Supplier<UserService> f) { this.factory = f; }
    public User find(long id) {
        if (real == null) real = factory.get();
        return cache.computeIfAbsent(id, real::find);
    }
}

// JDK dynamic proxy: ONE handler works for ANY interface
@SuppressWarnings("unchecked")
static <T> T timed(T target, Class<T> iface) {
    return (T) Proxy.newProxyInstance(
        iface.getClassLoader(), new Class<?>[] { iface },
        (proxy, method, args) -> {
            long t0 = System.nanoTime();
            try {
                return method.invoke(target, args);
            } catch (InvocationTargetException e) {
                throw e.getCause();                      // unwrap so callers see the real exception
            } finally {
                System.out.println(method.getName() + " " + (System.nanoTime() - t0) / 1_000 + " µs");
            }
        });
}
UserService svc = timed(new UserServiceImpl(), UserService.class);
svc.find(7);
```

**Why `this.method()` skips Spring's `@Transactional`:** the proxy sits *between caller and target*. Inside the target, `this` is the raw object, not the proxy, so the extra behaviour is skipped (self-invocation problem).

**When NOT to use:** simple one-off cases — a hand-written wrapper is clearer than reflection magic. Dynamic proxies are slower than direct calls (reflection) and hard to step through in a debugger.

**Pitfalls / follow-up probes:**
- JDK proxy: casting to the **implementation class** fails (`ClassCastException`) — you only get the interface type.
- CGLIB: `final` classes/methods can't be overridden → advice silently skipped; private methods aren't proxied.
- `InvocationTargetException` wraps the real exception — always unwrap `getCause()`.
- `equals/hashCode/toString` also pass through `invoke` — handle them if you care.
- Proxy vs Decorator: same shape; intent differs (control access vs add behaviour).

**Remember:** *JDK proxy = interfaces via InvocationHandler; CGLIB = runtime subclass; neither works on final.*

---

### Q187. Composite and Bridge — when do you use them? (Facade/Adapter covered separately.)

**Say this first:** "Composite treats a single object and a group of objects the same way (tree). Bridge splits an abstraction from its implementation so both can vary independently."

**Answer — Composite:**
- **Problem:** you have a tree (file system, menu, org chart, price bundle) and callers shouldn't have to check "is this a leaf or a group?".
- **Solution:** one interface for both leaf and container; the container holds children of that interface and forwards operations recursively.
- **JDK:** `java.awt.Container` holds `Component`s (and is one); Swing; `javax.swing.tree`.

```java
interface PriceItem { long pricePaise(); }

record Product(String name, long pricePaise) implements PriceItem { }          // leaf

final class Bundle implements PriceItem {                                       // composite
    private final List<PriceItem> children = new ArrayList<>();
    private final int discountPercent;
    Bundle(int discountPercent) { this.discountPercent = discountPercent; }
    Bundle add(PriceItem i) { children.add(i); return this; }
    public long pricePaise() {
        long sum = children.stream().mapToLong(PriceItem::pricePaise).sum();    // recursion for free
        return sum - sum * discountPercent / 100;
    }
}
PriceItem kit = new Bundle(10).add(new Product("Camera", 50_000_00))
                              .add(new Bundle(5).add(new Product("Lens", 20_000_00)).add(new Product("Bag", 2_000_00)));
long total = kit.pricePaise();     // caller doesn't care which is which
```

**Answer — Bridge:**
- **Problem:** two independent dimensions of variation. E.g. `Notification` kinds (Alert, Reminder) × delivery channels (Email, SMS, Push). With inheritance you'd need 3×2 = 6 classes, and each new dimension multiplies them.
- **Solution:** keep two hierarchies — the **abstraction** holds a reference to an **implementor** interface and delegates to it. Combine at runtime with composition.
- **JDK:** JDBC — `java.sql.Connection/Statement` (abstraction) talks to vendor `Driver` implementations; SLF4J API ↔ logging backends.

```java
interface Channel { void deliver(String to, String text); }              // implementor
class EmailChannel implements Channel { public void deliver(String to, String t) { /* SMTP */ } }
class SmsChannel   implements Channel { public void deliver(String to, String t) { /* gateway */ } }

abstract class Notification {                                            // abstraction
    protected final Channel channel;
    protected Notification(Channel c) { this.channel = c; }
    abstract void send(String to);
}
class Alert extends Notification {
    Alert(Channel c) { super(c); }
    void send(String to) { channel.deliver(to, "ALERT: server down"); }
}
class Reminder extends Notification {
    Reminder(Channel c) { super(c); }
    void send(String to) { channel.deliver(to, "Reminder: meeting at 5"); }
}
new Alert(new SmsChannel()).send("+91...");        // any abstraction x any channel
```

**When NOT to use:** Composite when leaves and containers really have different operations (forcing one interface leads to `UnsupportedOperationException` — a Liskov smell). Bridge when only one dimension varies — plain Strategy is enough.

**Pitfalls / follow-up probes:**
- Composite: where to put `add/remove` — on the common interface (transparent but unsafe) or only on the composite (safe but clients must know the type)?
- Cycles in a composite = infinite recursion; guard if children can be shared.
- Bridge vs Strategy: structurally identical; Bridge is a *design-time* split of a whole abstraction/implementation, Strategy swaps one algorithm.
- Bridge vs Adapter: Bridge is designed **up front**; Adapter is retrofitted.

**Remember:** *Composite = tree, treat one like many. Bridge = two hierarchies joined by composition.*

---

### Q188. Flyweight pattern — how do the Integer cache and String pool use it?

**Say this first:** "Flyweight saves memory by sharing identical immutable objects instead of creating millions of copies."

**Answer:**
- **Problem:** you need a huge number of small objects (characters in a document, trees in a game map, price-tick objects) and most of them hold the same data. Memory and GC suffer.
- **Idea:** split state into **intrinsic** (shared, immutable, kept inside the flyweight) and **extrinsic** (varies per use, passed in by the caller). A factory returns an existing shared instance if it already exists.
- **JDK examples:**
  - `Integer.valueOf(int)` caches `-128..127` (the upper bound can be raised with `-XX:AutoBoxCacheMax=`). `Byte`, `Short`, `Long`, `Character` (0–127) and `Boolean.valueOf` cache too.
  - **String pool** — literals are interned; `s.intern()` returns the canonical pooled instance. (Since Java 7 the pool lives on the heap.)
  - Enum constants are naturally single shared instances.

```java
Integer a = 127, b = 127;     System.out.println(a == b);          // true  (cached)
Integer c = 128, d = 128;     System.out.println(c == d);          // false (new objects)
String s1 = "java", s2 = "java";           System.out.println(s1 == s2);                        // true  (pool)
String s3 = new String("java");            System.out.println(s1 == s3);                        // false
System.out.println(s1 == s3.intern());                                                           // true
```

**Own flyweight — map tile types shared across a million tiles:**
```java
final class TileType {                          // intrinsic state: immutable + shared
    final String name; final byte[] texture;
    private TileType(String name, byte[] texture) { this.name = name; this.texture = texture; }

    private static final Map<String, TileType> POOL = new ConcurrentHashMap<>();
    static TileType of(String name) {           // flyweight factory
        return POOL.computeIfAbsent(name, n -> new TileType(n, loadTexture(n)));
    }
    private static byte[] loadTexture(String n) { return new byte[1024 * 100]; }   // expensive, load once
}
record Tile(int x, int y, TileType type) { }    // extrinsic state (x,y) lives in the Tile

List<Tile> map = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) map.add(new Tile(i % 1000, i / 1000, TileType.of(i % 3 == 0 ? "grass" : "water")));
// only 2 textures in memory instead of 1,000,000
```

**When NOT to use:** when you don't have huge numbers of similar objects — the added complexity and the factory lookup cost aren't worth it. Never share **mutable** objects this way.

**Pitfalls / follow-up probes:**
- **Never compare boxed values or Strings with `==`** — it "works" for small values and fails mysteriously above the cache range. Use `equals`.
- A pool that never evicts is a memory leak; consider `WeakReference` values or a bounded cache.
- Flyweights must be immutable and thread-safe, since many callers share them.
- `String.intern()` has a cost; use it only when you have lots of duplicate strings (e.g. parsed repeated keys). Alternative: your own `Map<String,String>` dedup or G1's `-XX:+UseStringDeduplication`.

**Remember:** *Share the immutable intrinsic part, pass the changing part in — Integer cache and String pool.*

---

### Q189. Observer pattern — how do you implement it, why was `java.util.Observable` deprecated, and what are `PropertyChangeListener` and the Flow API?

**Say this first:** "Observer lets one object (subject) notify many others (listeners) of changes without knowing who they are. In modern Java, use a typed listener interface, `PropertyChangeSupport`, or `Flow` — not `Observable`."

**Answer:**
- **Problem:** when an order is placed, you must send an email, update stats, reserve stock, … If `OrderService` calls each directly it is coupled to all of them and grows with every new requirement.
- **Solution:** `OrderService` just publishes "order placed". Anyone interested registers a listener. Adding a listener changes **no** existing code.
- **`java.util.Observable` / `Observer` (deprecated since Java 9)** — reasons: `Observable` is a **class** (forces inheritance, can't use with another superclass), `setChanged()` is protected, notification payload is an untyped `Object`, not serializable, not thread-safe-friendly, and no ordering/guarantees. Use the alternatives below.

```java
// 1) Hand-written, typed, thread-safe observer
interface OrderListener { void onPlaced(OrderPlaced event); }
record OrderPlaced(String orderId, long amountPaise) { }

class OrderService {
    private final List<OrderListener> listeners = new CopyOnWriteArrayList<>();   // safe to iterate while (un)registering
    void addListener(OrderListener l)    { listeners.add(l); }
    void removeListener(OrderListener l) { listeners.remove(l); }
    void place(String id, long amt) {
        // ... save order ...
        OrderPlaced ev = new OrderPlaced(id, amt);
        for (OrderListener l : listeners) {
            try { l.onPlaced(ev); } catch (RuntimeException e) { /* one bad listener must not break the rest */ }
        }
    }
}
OrderService svc = new OrderService();
svc.addListener(e -> System.out.println("email for " + e.orderId()));
svc.addListener(e -> Metrics.count("orders"));

// 2) java.beans.PropertyChangeSupport — built-in for "property X changed from A to B"
class Account {
    private final PropertyChangeSupport pcs = new PropertyChangeSupport(this);
    private long balance;
    void addPropertyChangeListener(PropertyChangeListener l) { pcs.addPropertyChangeListener(l); }
    void setBalance(long nb) { long old = balance; balance = nb; pcs.firePropertyChange("balance", old, nb); }
}
new Account().addPropertyChangeListener(ev -> System.out.println(ev.getPropertyName() + ": " + ev.getOldValue() + " -> " + ev.getNewValue()));

// 3) java.util.concurrent.Flow (Java 9) — Observer + async + BACKPRESSURE (reactive streams contract)
try (SubmissionPublisher<String> pub = new SubmissionPublisher<>()) {
    pub.subscribe(new Flow.Subscriber<>() {
        Flow.Subscription sub;
        public void onSubscribe(Flow.Subscription s) { sub = s; s.request(1); }   // subscriber controls the pace
        public void onNext(String item)  { System.out.println(item); sub.request(1); }
        public void onError(Throwable t) { t.printStackTrace(); }
        public void onComplete()         { System.out.println("done"); }
    });
    pub.submit("hello");
}
```

**Push vs pull and sync vs async:** the simple listener is **synchronous** — a slow listener slows the publisher. `Flow`/`SubmissionPublisher` delivers asynchronously on an executor and lets subscribers request items (backpressure), so a slow consumer isn't flooded.

**When NOT to use:** when you need guaranteed delivery/ordering across processes — use a message broker. When the flow is a simple known sequence, a direct call is easier to read and debug.

**Pitfalls / follow-up probes:**
- **Lapsed listener leak:** listeners registered but never removed keep objects alive (memory leak). Provide `remove`, or hold `WeakReference`s.
- Iterating a plain `ArrayList` while a listener unregisters → `ConcurrentModificationException`; use `CopyOnWriteArrayList`.
- Exceptions in one listener — isolate them.
- Listeners that call back into the subject can cause **re-entrancy/loops**.
- Execution order is usually unspecified — don't depend on it.

**Remember:** *Subject publishes, listeners subscribe; Observable is dead — use typed listeners, PropertyChangeSupport or Flow.*

---

### Q190. Strategy vs Template Method in plain Java — how do lambdas change these patterns?

**Say this first:** "Strategy = *pass* the varying algorithm in (composition). Template Method = *override* the varying steps in a subclass (inheritance). With lambdas, most Strategies are just a function parameter."

**Answer:**
- **Strategy** — define a family of interchangeable algorithms behind one interface; the context uses whichever it is given. **Why:** removes `if/else` or `switch` on type, and obeys Open/Closed. **JDK:** `Comparator` (sort strategy), `RejectedExecutionHandler`, `ThreadFactory`.
- **Template Method** — a base class fixes the *skeleton* of an algorithm in a `final` method and leaves some steps abstract/hook methods. **Why:** share the invariant flow, let subclasses fill in the variable steps. **JDK:** `AbstractList` (implement `get`/`size`, you get the rest), `InputStream.read(byte[],…)` (calls your abstract `read()`), `HttpServlet.service` → `doGet/doPost`, Spring's `JdbcTemplate`.

```java
// ---------- Strategy with lambdas: no class per algorithm
@FunctionalInterface
interface DiscountPolicy { long apply(long amountPaise); }

class Checkout {
    private final DiscountPolicy policy;
    Checkout(DiscountPolicy p) { this.policy = p; }
    long total(long amount) { return amount - policy.apply(amount); }
}
Checkout festive = new Checkout(a -> a / 10);                       // 10% off
Checkout flat    = new Checkout(a -> Math.min(a, 50_00));           // flat 50 off
Checkout none    = new Checkout(a -> 0);

// Enum strategy: named, stateless, discoverable
enum Shipping implements ToLongFunction<Parcel> {
    STANDARD { public long applyAsLong(Parcel p) { return 40_00; } },
    EXPRESS  { public long applyAsLong(Parcel p) { return 40_00 + p.weightKg() * 10_00; } }
}

// ---------- Template Method: fixed skeleton, variable steps
abstract class DataImporter {
    public final void run(Path file) throws IOException {             // final: skeleton can't be changed
        List<String> lines = Files.readAllLines(file);
        for (String line : lines) {
            Row r = parse(line);                                       // step 1 (abstract)
            if (isValid(r)) save(r);                                   // step 2 (hook) + step 3 (abstract)
        }
        afterImport();                                                 // optional hook
    }
    protected abstract Row parse(String line);
    protected abstract void save(Row r);
    protected boolean isValid(Row r) { return true; }                  // hook with default
    protected void afterImport() { }
}
class CsvUserImporter extends DataImporter {
    protected Row parse(String l) { return Row.fromCsv(l); }
    protected void save(Row r)    { /* insert */ }
}

// ---------- The same skeleton as Strategy (composition): no subclass needed
class FunctionalImporter {
    <T> void run(List<String> lines, Function<String, T> parse, Predicate<T> valid, Consumer<T> save) {
        lines.stream().map(parse).filter(valid).forEach(save);
    }
}
```

**Choosing:**
| | Strategy | Template Method |
|---|---|---|
| Mechanism | Composition (inject) | Inheritance (override) |
| Change at runtime? | Yes | No (fixed at class definition) |
| Granularity | Whole algorithm | Individual steps |
| Typical modern form | Lambda / method ref | Often rewritten as functional parameters |

**When NOT to use:** Strategy with 2 stable branches — a plain `if` is simpler. Template Method when the base class would need many hooks — switch to composition.

**Pitfalls / follow-up probes:**
- Template Method: calling overridable methods from a **constructor** exposes half-initialised subclasses.
- Fragile base class — a change in the skeleton can break all subclasses. Mark the skeleton `final`.
- Strategy: stateful strategies shared across threads — keep them stateless or immutable.
- Say "Hollywood principle" for Template Method: *don't call us, we'll call you*.

**Remember:** *Strategy = inject the algorithm; Template Method = inherit the skeleton; lambdas make Strategy tiny.*

---

### Q191. Command and Memento — how do you implement undo/redo?

**Say this first:** "Command turns an action into an object (so it can be queued, logged, undone). Memento saves and restores an object's state without exposing its internals. Together they give undo/redo."

**Answer:**
- **Command:** wrap a request (`execute()` and `undo()`) in an object. The invoker doesn't know what it does. **Why:** you can store it in a history stack, queue it, retry it, schedule it, or log it. **JDK:** `Runnable` and `Callable` are commands you hand to an `ExecutorService`.
- **Memento:** a snapshot object holding the state needed to restore an *originator*. A *caretaker* keeps the snapshots but never looks inside. **Why:** undo without breaking encapsulation.
- **Which for undo?** Command with an explicit inverse (`undo()`) is memory-light — it stores only the change. Memento stores the whole state — simpler and safer when the inverse is hard to compute. Many editors mix both.

```java
// ---------- Command-based undo/redo
interface Command { void execute(); void undo(); }

final class InsertText implements Command {
    private final StringBuilder doc; private final int pos; private final String text;
    InsertText(StringBuilder doc, int pos, String text) { this.doc = doc; this.pos = pos; this.text = text; }
    public void execute() { doc.insert(pos, text); }
    public void undo()    { doc.delete(pos, pos + text.length()); }
}

final class History {
    private final Deque<Command> undoStack = new ArrayDeque<>();
    private final Deque<Command> redoStack = new ArrayDeque<>();
    void run(Command c) { c.execute(); undoStack.push(c); redoStack.clear(); }   // a new action kills the redo branch
    void undo() { if (!undoStack.isEmpty()) { Command c = undoStack.pop(); c.undo(); redoStack.push(c); } }
    void redo() { if (!redoStack.isEmpty()) { Command c = redoStack.pop(); c.execute(); undoStack.push(c); } }
}

StringBuilder doc = new StringBuilder();
History h = new History();
h.run(new InsertText(doc, 0, "Hello"));
h.run(new InsertText(doc, 5, " World"));   // "Hello World"
h.undo();                                   // "Hello"
h.redo();                                   // "Hello World"

// ---------- Memento: snapshot instead of inverse
final class Editor {
    record Snapshot(String text, int cursor) { }         // immutable memento (record = easy)
    private String text = ""; private int cursor;
    void type(String s) { text = text.substring(0, cursor) + s + text.substring(cursor); cursor += s.length(); }
    Snapshot save()          { return new Snapshot(text, cursor); }
    void restore(Snapshot s) { this.text = s.text(); this.cursor = s.cursor(); }
}
Deque<Editor.Snapshot> snaps = new ArrayDeque<>();    // caretaker
Editor ed = new Editor();
snaps.push(ed.save()); ed.type("abc");
ed.restore(snaps.pop());                              // back to empty
```

**Command in the real world:** job queues, `ExecutorService.submit(Runnable)`, macro recording, transaction scripts, GUI actions (menu item + toolbar button share one command), "outbox" tasks.

**When NOT to use:** if actions are irreversible (sent emails, charged cards) `undo()` is a lie — model a compensating action (refund) instead. For trivially small state a simple copy is enough.

**Pitfalls / follow-up probes:**
- Memento memory blow-up — cap the history (e.g. 100), or store diffs.
- Mementos of **mutable** objects must be deep copies, otherwise they change with the original.
- A `redo` stack must be cleared on any new command.
- Commands that capture stale state (e.g. a position that moved) corrupt undo — capture what's needed at `execute()` time.
- Command vs Strategy: Command is an *action with a receiver and possibly undo*; Strategy is an *algorithm variant*.

**Remember:** *Command = action as object (execute/undo); Memento = state snapshot; both give undo/redo.*

---

### Q192. Chain of Responsibility — how do you build a validation/processing pipeline?

**Say this first:** "Pass a request along a chain of handlers; each one decides to handle it, modify it, or pass it on. It's the shape of servlet filters and middleware."

**Answer:**
- **Problem:** a request goes through many independent checks/steps (validate, authenticate, rate-limit, enrich, log). One big method with 15 `if` blocks is hard to test and reorder.
- **Solution:** each step is a small handler with a reference to the next. Handlers can **stop** the chain (reject) or continue. Add/remove/reorder without touching others.
- **JDK/Servlet:** `javax/jakarta.servlet.Filter` + `FilterChain.doFilter`, `java.util.logging` handlers/parent loggers, exception `catch` blocks are a language-level chain. Spring Security's filter chain is the same idea.

```java
// ---------- Classic linked chain
abstract class Handler {
    private Handler next;
    Handler then(Handler n) { this.next = n; return n; }                 // fluent wiring
    final void handle(OrderRequest r) {
        check(r);                                                        // throws to stop the chain
        if (next != null) next.handle(r);
    }
    protected abstract void check(OrderRequest r);
}
class StockCheck extends Handler {
    protected void check(OrderRequest r) { if (!inStock(r)) throw new ValidationException("out of stock"); }
}
class FraudCheck extends Handler {
    protected void check(OrderRequest r) { if (r.totalPaise() > 10_00_000_00L) throw new ValidationException("manual review"); }
}
class CreditCheck extends Handler {
    protected void check(OrderRequest r) { /* ... */ }
}
Handler head = new StockCheck();
head.then(new FraudCheck()).then(new CreditCheck());
head.handle(request);

// ---------- Modern, simpler: a List of functions (no linking boilerplate)
interface Rule { Optional<String> check(OrderRequest r); }          // empty = OK, else error message

class OrderValidator {
    private final List<Rule> rules;
    OrderValidator(List<Rule> rules) { this.rules = List.copyOf(rules); }
    List<String> validateAll(OrderRequest r) {                       // collect ALL errors, not just first
        return rules.stream().map(rl -> rl.check(r)).flatMap(Optional::stream).toList();
    }
}
Rule notEmpty = r -> r.items().isEmpty() ? Optional.of("no items") : Optional.empty();
Rule positive = r -> r.totalPaise() <= 0 ? Optional.of("total must be > 0") : Optional.empty();
var validator = new OrderValidator(List.of(notEmpty, positive));

// ---------- Middleware style: each step wraps the next (like filters)
interface Step { Response apply(Request req, Function<Request, Response> next); }
Function<Request, Response> chain(List<Step> steps, Function<Request, Response> terminal) {
    Function<Request, Response> f = terminal;
    for (int i = steps.size() - 1; i >= 0; i--) {
        Step s = steps.get(i); Function<Request, Response> nxt = f;
        f = req -> s.apply(req, nxt);
    }
    return f;
}
```

**Two flavours:** *first-handler-wins* (like `catch` blocks / approval levels — one handler processes it and stops) vs *pipeline* (every handler runs, any can veto — filters, validation).

**When NOT to use:** if order is fixed and tiny, call methods directly. If *every* handler must run and the result is combined, a simple loop over a list is clearer than a linked chain.

**Pitfalls / follow-up probes:**
- A request may fall off the end **unhandled** — decide what that means (default handler / error).
- Hard to debug long chains — log which handler acted.
- Forgetting to call `next` silently drops the request (middleware style).
- Order bugs: authentication must run before authorization.
- Cycles in the chain cause infinite loops.

**Remember:** *Chain = ordered handlers, each handles or forwards — filters, middleware, validation pipelines.*

---

### Q193. Iterator and Visitor — how do they work, and is Visitor still needed with sealed types and pattern matching?

**Say this first:** "Iterator walks a collection without exposing its structure. Visitor adds new operations to a fixed class hierarchy without editing those classes — and in modern Java, `sealed` + `switch` patterns often replace it."

**Answer — Iterator:**
- **Why:** callers shouldn't know if data lives in an array, linked list, tree, or is generated lazily. `Iterable<T>` gives `for-each` support.
- **How:** implement `Iterator<T>` (`hasNext`, `next`, optional `remove`) and return it from `iterator()`.
- **Fail-fast:** `ArrayList`'s iterator checks a `modCount`; structural change during iteration throws `ConcurrentModificationException` (best effort, not a guarantee). `CopyOnWriteArrayList` and concurrent collections have snapshot/weakly consistent iterators.

```java
final class Range implements Iterable<Integer> {
    private final int from, to;
    Range(int from, int to) { this.from = from; this.to = to; }
    public Iterator<Integer> iterator() {
        return new Iterator<>() {
            private int cur = from;                       // each call to iterator() gets its own cursor
            public boolean hasNext() { return cur < to; }
            public Integer next() {
                if (!hasNext()) throw new NoSuchElementException();
                return cur++;
            }
        };
    }
}
for (int i : new Range(0, 3)) System.out.println(i);     // 0 1 2
```

**Answer — Visitor:**
- **Problem:** a fixed set of types (`Circle`, `Rect`) and you keep adding *operations* (area, draw, export-to-JSON). Putting every operation inside every class bloats them.
- **Classic solution (double dispatch):** each element has `accept(Visitor v)` which calls `v.visit(this)`; each operation is a visitor class.
- **JDK:** `Files.walkFileTree(path, FileVisitor)` / `SimpleFileVisitor`; `javax.lang.model` element visitors.

```java
// Classic Visitor
interface ShapeVisitor<R> { R visit(Circle c); R visit(Rect r); }
interface Shape { <R> R accept(ShapeVisitor<R> v); }
record Circle(double r) implements Shape { public <R> R accept(ShapeVisitor<R> v) { return v.visit(this); } }
record Rect(double w, double h) implements Shape { public <R> R accept(ShapeVisitor<R> v) { return v.visit(this); } }

class AreaVisitor implements ShapeVisitor<Double> {
    public Double visit(Circle c) { return Math.PI * c.r() * c.r(); }
    public Double visit(Rect r)   { return r.w() * r.h(); }
}
double a = new Circle(2).accept(new AreaVisitor());

// Java 21: sealed + records + pattern-matching switch — no accept(), no visitor interface
sealed interface Shape2 permits Circle2, Rect2 { }
record Circle2(double r) implements Shape2 { }
record Rect2(double w, double h) implements Shape2 { }

static double area(Shape2 s) {
    return switch (s) {                                       // compiler checks it is exhaustive
        case Circle2(double r)         -> Math.PI * r * r;    // record deconstruction pattern
        case Rect2(double w, double h) -> w * h;
    };                                                        // add Triangle2 => every switch fails to compile until updated
}

// Walking a file tree with the JDK visitor
Files.walkFileTree(Path.of("."), new SimpleFileVisitor<>() {
    @Override public FileVisitResult visitFile(Path f, BasicFileAttributes at) {
        System.out.println(f); return FileVisitResult.CONTINUE;
    }
});
```

**Visitor vs sealed + switch:**
- Both make "add a new *operation*" easy and "add a new *type*" hard (compiler forces updates).
- The switch version is far less boilerplate, no `accept` method, works with records, and exhaustiveness is checked. Pattern-matching `switch` is final in **Java 21** (sealed in 17, records in 16).
- Classic Visitor is still useful when you need **shared traversal logic**, can't use sealed/records (Java < 21, third-party hierarchy), or want visitors with state across the walk.

**When NOT to use Visitor:** if the type hierarchy changes often — every new type touches every visitor. Then ordinary polymorphism (method on each class) is better.

**Pitfalls / follow-up probes:**
- Calling `remove()` twice or before `next()` → `IllegalStateException`.
- Returning the *same* iterator from `iterator()` breaks nested loops — always return a new one.
- Modifying a list while iterating: use `iterator.remove()` or `list.removeIf(...)`.
- A pattern `switch` with a `default` hides missing cases — omit `default` on sealed types so the compiler helps you.

**Remember:** *Iterator hides structure; Visitor adds operations; sealed + switch is the modern Visitor.*

---

### Q194. Mediator and Null Object — what problems do they solve?

**Say this first:** "Mediator centralises how objects talk so they don't reference each other. Null Object replaces `null` with a do-nothing object so callers don't need null checks."

**Answer — Mediator:**
- **Problem:** many objects interact with many others (chat users, form widgets, airport planes). Direct references create an N×N web: change one and everything breaks.
- **Solution:** they talk only to a *mediator*, which knows the rules of who gets what. Colleagues become reusable and simple; complexity is in one place.
- **Real-world:** `ExecutorService` sits between task submitters and worker threads; a message broker or an in-process event bus plays the same role.
- **Risk:** the mediator can turn into a **God object**. Keep it to coordination only.

```java
interface ChatRoom { void send(String from, String msg); void join(User u); }

class SimpleChatRoom implements ChatRoom {
    private final List<User> users = new CopyOnWriteArrayList<>();
    public void join(User u) { users.add(u); }
    public void send(String from, String msg) {
        for (User u : users) if (!u.name().equals(from)) u.receive(from, msg);   // routing rules live HERE
    }
}
class User {
    private final String name; private final ChatRoom room;      // knows only the mediator
    User(String name, ChatRoom room) { this.name = name; this.room = room; room.join(this); }
    String name() { return name; }
    void say(String m) { room.send(name, m); }
    void receive(String from, String m) { System.out.println(name + " got " + m + " from " + from); }
}
```

**Answer — Null Object:**
- **Problem:** `null` forces `if (x != null)` everywhere; forget one and you get an NPE.
- **Solution:** provide an implementation of the interface that safely does nothing (or returns a neutral value).
- **JDK:** `Collections.emptyList()`, `Collections.emptyIterator()`, `OutputStream.nullOutputStream()`, `InputStream.nullInputStream()`, `Writer.nullWriter()` (Java 11). `Optional` is the *"explicit absence"* cousin — good for return values, not for fields/parameters.

```java
interface Logger { void info(String msg); }
class ConsoleLogger implements Logger { public void info(String m) { System.out.println(m); } }
enum NoOpLogger implements Logger { INSTANCE; public void info(String m) { } }        // stateless => one shared instance

class OrderService {
    private final Logger log;
    OrderService(Logger log) { this.log = Objects.requireNonNullElse(log, NoOpLogger.INSTANCE); }
    void place() { log.info("placing"); }                // no null check anywhere
}

// Discount example: "no discount" is a real object, not null
interface Discount { long apply(long amt); }
Discount NONE = amt -> amt;
```

**When NOT to use:** if the *absence must be handled differently* (e.g. "user not found" should be a 404), hiding it with a null object silently swallows an error. Don't use a null object to hide real failures.

**Pitfalls / follow-up probes:**
- Mediator vs Observer: Observer is one-to-many broadcast; Mediator is many-to-many *with routing logic*.
- Mediator vs Facade: Facade is one-way (client → subsystem); Mediator is two-way between peers.
- Null Object makes bugs quieter — a no-op logger is fine, a no-op payment processor is not.
- Never return `null` for collections — return `List.of()` (see Effective Java question).

**Remember:** *Mediator = one hub instead of a web; Null Object = a harmless stand-in for null.*

---

### Q195. How do you do Dependency Injection without a framework (constructor injection and manual wiring)?

**Say this first:** "DI is just *passing dependencies in* instead of creating them inside. You don't need Spring for it — you need constructors and one place (the composition root) where everything is wired."

**Answer:**
- **Dependency Inversion Principle** = depend on abstractions. **Dependency Injection** = the technique of supplying those abstractions from outside. **IoC container** (Spring/Guice) = a tool that automates the wiring.
- **Problem with `new` inside a class:** `class OrderService { Repo repo = new JdbcRepo(); }` — you can't test without a DB, can't swap implementations, and the dependency is hidden.
- **Constructor injection:** declare dependencies as `final` fields set in the constructor. The object can never exist half-configured, dependencies are obvious from the signature, and tests just pass fakes.
- **Composition root:** one place near `main()` that builds the object graph. Nothing else calls `new` on services.

```java
interface OrderRepository { void save(Order o); Optional<Order> find(String id); }
interface PaymentGateway  { Receipt charge(String customerId, long amount); }

class OrderService {                                              // depends only on abstractions
    private final OrderRepository repo;
    private final PaymentGateway payments;
    OrderService(OrderRepository repo, PaymentGateway payments) {
        this.repo = Objects.requireNonNull(repo);
        this.payments = Objects.requireNonNull(payments);
    }
    Receipt place(Order o) { Receipt r = payments.charge(o.customerId(), o.total()); repo.save(o); return r; }
}

// Composition root — the ONLY place that knows concrete classes
public final class App {
    public static void main(String[] args) {
        DataSource ds = Pools.create(System.getenv("DB_URL"));
        OrderRepository repo = new JdbcOrderRepository(ds);
        PaymentGateway gw = new RetryingGateway(new BankGateway(Config.bankUrl()), 3);    // decorators plug in here
        OrderService orders = new OrderService(repo, gw);
        new HttpServer(orders).start();
    }
}

// Test — no framework, no DB
@Test void savesAfterCharging() {
    var repo = new InMemoryOrderRepository();
    var svc = new OrderService(repo, (cust, amt) -> new Receipt(cust, amt));   // lambda as fake gateway
    svc.place(new Order("o1", "c1", 500));
    assertTrue(repo.find("o1").isPresent());
}
```

**Injection types:** *constructor* (preferred; immutable, mandatory deps), *setter* (optional deps, mutable), *field* (reflection-based; needs a framework; can't be `final`; hides deps — avoid).

**Why a container then?** At 5 classes manual wiring is perfect. At 500 classes with scopes, lifecycle, AOP and config profiles, a container saves effort. The design is the same either way — a container just does the `new` calls.

**Anti-pattern contrast — Service Locator:** `ServiceLocator.get(PaymentGateway.class)` *inside* the class hides dependencies and makes tests awkward (see anti-patterns question).

**Pitfalls / follow-up probes:**
- **Too many constructor parameters (>5)** = the class does too much (SRP violation), not a reason to switch to field injection.
- **Circular dependencies** (A needs B, B needs A) — redesign; extract a third class or use an interface/event.
- Don't inject the container itself into classes.
- Inject **factories/Suppliers** (`Supplier<Connection>`) when you need a *new* instance each time, instead of `new` inside.
- Injecting `Clock`/`Random`/`UUID` suppliers makes time- and randomness-dependent code testable.

**Remember:** *Pass collaborators in through the constructor; build the graph once at the composition root.*

---

### Q196. SOLID with a violation → fix example for every letter.

**Say this first:** "I'll show each principle as the smell I've actually seen and how I'd refactor it." (Summary table is in Q179 — here is the code.)

**Answer:**

**S — Single Responsibility: one reason to change**
```java
// ❌ calculates, persists, and formats — 3 reasons to change
class Invoice {
    long total() { ... }
    void saveToDb() { ... }
    String toPdf() { ... }
}
// ✅ each class has one reason
class Invoice { long total() { ... } }
class InvoiceRepository { void save(Invoice i) { ... } }
class InvoicePdfRenderer { byte[] render(Invoice i) { ... } }
```

**O — Open/Closed: open for extension, closed for modification**
```java
// ❌ every new shape edits this method
double area(Object s) { if (s instanceof Circle c) return ...; else if (s instanceof Rect r) return ...; throw ...; }
// ✅ new shape = new class, no edit to existing code
interface Shape { double area(); }
record Circle(double r) implements Shape { public double area() { return Math.PI * r * r; } }
```
(With a sealed hierarchy + switch you trade OCP for exhaustiveness — fine when the set of types is closed.)

**L — Liskov Substitution: a subtype must work wherever its parent works**
```java
// ❌ Penguin "is a" Bird that can't fly -> breaks callers
class Bird { void fly() { } }
class Penguin extends Bird { @Override void fly() { throw new UnsupportedOperationException(); } }
// ✅ model capabilities, not a fat parent
interface Bird { void eat(); }
interface Flyer { void fly(); }
class Sparrow implements Bird, Flyer { public void eat() { } public void fly() { } }
class Penguin implements Bird { public void eat() { } }
```
LSP rules: don't strengthen preconditions, don't weaken postconditions, don't throw new unchecked exceptions, keep invariants. Classic: `Square extends Rectangle` where `setWidth` also changes height breaks callers.

**I — Interface Segregation: no client forced to depend on methods it doesn't use**
```java
// ❌ read-only report job forced to implement write methods
interface UserStore { User find(long id); void save(User u); void delete(long id); }
// ✅ split by client need
interface UserReader { User find(long id); }
interface UserWriter { void save(User u); void delete(long id); }
class JdbcUserStore implements UserReader, UserWriter { ... }
class ReportJob { ReportJob(UserReader users) { ... } }     // sees only what it needs
```

**D — Dependency Inversion: high-level code depends on abstractions**
```java
// ❌ business rule welded to MySQL and SMTP
class SignupService { private final MySqlUserDao dao = new MySqlUserDao(); private final SmtpMailer mail = new SmtpMailer(); }
// ✅ abstractions owned by the high-level module, implementations injected
interface UserDao { void save(User u); }
interface Mailer { void send(String to, String body); }
class SignupService {
    private final UserDao dao; private final Mailer mail;
    SignupService(UserDao dao, Mailer mail) { this.dao = dao; this.mail = mail; }
}
```
DIP ≠ DI: DIP is the design rule; DI is a way to follow it.

**Pitfalls / follow-up probes:**
- **Over-applying SOLID** produces interface-per-class and 40 tiny classes. Apply a principle when you *feel the pain* (growing `if` chain, hard to test).
- "How do you spot an SRP violation?" — describe the class without using "and"; count the different reasons it would change; look at import lists.
- "Does OCP mean never edit code?" — no; it means design the common change axes so they need only additions.
- LSP smell checklist: `instanceof` checks on parameters, empty overrides, `throw new UnsupportedOperationException()`.
- Which is violated by a god `switch` on type? Mostly **O** (and often **S**).

**Remember:** *Split reasons (S), add don't edit (O), subtype keeps promises (L), small interfaces (I), depend on abstractions (D).*

---

### Q197. DRY, KISS, YAGNI, Law of Demeter, Tell-Don't-Ask, composition over inheritance, immutability, high cohesion / low coupling — explain each with code.

**Say this first:** "These are the everyday habits that make code easy to change. I'll give one-line meaning and a small before/after."

**Answer:**

- **DRY (Don't Repeat Yourself)** — every piece of *knowledge* has one home. It's about duplicated *rules*, not duplicated characters. Two similar-looking pieces that change for different reasons should **not** be merged (wrong abstraction is worse than duplication).
- **KISS** — pick the simplest design that works. A `for` loop beats a 5-class framework.
- **YAGNI** — don't build features "we might need later". Unused flexibility has a cost (code to read, test, and keep compatible).

- **Law of Demeter** ("only talk to your friends"): a method should call methods only on itself, its fields, its parameters, objects it creates. Avoid chains of `a.getB().getC().doX()` — they couple you to the whole structure.
```java
// ❌ train wreck: knows Customer -> Address -> City -> Zone
double fee = order.getCustomer().getAddress().getCity().getZone().getShippingRate();
// ✅ ask the object that owns the knowledge
double fee = order.shippingRate();       // Order delegates internally
```
(Fluent builders and streams are not violations — they return the *same* type or a new wrapper, not reach into internals.)

- **Tell, Don't Ask** — tell objects what to do instead of pulling their data out and deciding yourself.
```java
// ❌ asking, then deciding outside (logic leaks out of Account)
if (account.getBalance() >= amt) account.setBalance(account.getBalance() - amt); else throw ...;
// ✅ telling — the rule lives with the data
account.withdraw(amt);                    // throws InsufficientFundsException inside
```

- **Composition over inheritance** — "has-a" is more flexible than "is-a". Inheritance couples subclass to parent internals and is fixed at compile time.
```java
// ❌ breaks: HashSet.addAll() internally calls add(), so count is doubled
class CountingSet<E> extends HashSet<E> { int count; @Override public boolean add(E e) { count++; return super.add(e); } }
// ✅ wrap and delegate (decorator)
class CountingSet<E> { private final Set<E> inner = new HashSet<>(); private int count;
    boolean add(E e) { count++; return inner.add(e); } }
```
- **Favor immutability** — `final` fields, no setters, defensive copies, records. Immutable objects are thread-safe, safe as map keys, and easy to reason about.
```java
record Money(long paise, String currency) {
    Money { Objects.requireNonNull(currency); if (paise < 0) throw new IllegalArgumentException(); }
    Money plus(Money o) { return new Money(paise + o.paise, currency); }       // returns new, never mutates
}
```
- **Program to an interface, not an implementation** — `List<String> x = new ArrayList<>();`, parameters typed by the narrowest interface. You can swap implementations and fake them in tests.
- **High cohesion** — everything in a class/module belongs together (one clear purpose). **Low coupling** — classes know as little as possible about each other (talk through small interfaces, not internals). Together: a change touches one place.

**Pitfalls / follow-up probes:**
- DRY taken too far → premature abstraction with 8 boolean flags. "Rule of three": wait for the third repetition.
- Law of Demeter ≠ "never chain dots" — it's about structural knowledge, not syntax. DTO getters chains are less harmful than domain object chains.
- Inheritance is fine for true "is-a" with a stable base designed for it (`AbstractList`); otherwise *design and document for inheritance or prohibit it* (`final`).
- Anemic classes (getters/setters only) force everyone to "ask" — contradicts Tell-Don't-Ask.
- YAGNI doesn't mean no design — keep the code easy to change, just don't build the unneeded part.

**Remember:** *One home per rule, keep it simple, build only what's needed, talk to friends, tell don't ask, has-a over is-a, immutable by default.*

---

### Q198. Common anti-patterns — and how do you choose the right pattern for a scenario?

**Say this first:** "An anti-pattern is a common 'solution' that causes more problems than it solves. Choosing a pattern means naming the *problem force* first, then matching it."

**Answer — Anti-patterns:**
- **God object** — one class that knows and does everything (`OrderManager` 3000 lines). Violates SRP, impossible to test, merge conflicts. *Fix:* extract cohesive collaborators.
- **Anemic domain model** — entities with only getters/setters; all logic in `XService` classes. You lose encapsulation and invariants. *Fix:* put behaviour with the data (`order.addItem()`, `account.withdraw()`); keep services for orchestration.
- **Singleton abuse** — global mutable state accessed by `X.getInstance()` from everywhere: hidden dependencies, test pollution, concurrency bugs. *Fix:* create once, inject.
- **Service Locator** — classes pull dependencies from a global registry:
```java
// ❌ the dependency is invisible in the constructor; tests must set up the global registry
class ReportService { void run() { Mailer m = ServiceLocator.get(Mailer.class); m.send(...); } }
// ✅ explicit
class ReportService { private final Mailer m; ReportService(Mailer m) { this.m = m; } }
```
- **Premature optimization** — "premature optimization is the root of all evil" (Knuth). Measure with a profiler/JMH first; optimise the proven bottleneck.
- **Others worth naming:** *Golden hammer* (using one pattern/tech everywhere), *Spaghetti code*, *Copy-paste programming*, *Primitive obsession* (`String` for email/money — use value types), *Magic numbers/strings*, *Constant interface*, *Exceptions for control flow*, *Swallowing exceptions* (`catch (Exception e) {}`), *Over-engineering* (factory-of-factories for one class).

**Answer — "Which pattern would you use for X?"**

| Scenario | Pattern(s) | Why / sketch |
|---|---|---|
| Multiple **payment methods** (UPI, card, wallet), new ones added often | **Strategy** (+ Factory/registry) | `Map<PaymentType, PaymentHandler>`; adding a method = adding one class, no `if/else` |
| **Notification channels** (email, SMS, push), user picks channels | **Strategy + Observer** (Bridge if message types × channels both vary) | `List<Notifier>`; event → each channel sends; failures isolated |
| **Undo/redo** in an editor | **Command (+ Memento)** | `execute/undo` objects on two stacks |
| **Pipeline of validations/filters** | **Chain of Responsibility** (or list of rules) | ordered handlers, any can veto |
| **Event listeners** (order placed → mail, stats) | **Observer** / event bus | publisher doesn't know subscribers |
| **Object creation with many params** | **Builder** (record for few fields) | named steps, validate in `build()` |
| Different **objects behave differently by state** (order: NEW→PAID→SHIPPED) | **State** | each state class defines allowed transitions |
| Add **cross-cutting behaviour** (retry, cache, metrics) without editing the class | **Decorator / Proxy** | same interface wrapper |
| **Legacy/third-party API** with wrong shape | **Adapter** | translate at the boundary |
| **Complex subsystem**, clients need a simple entry | **Facade** | one `placeOrder()` method |
| **Tree structure** (menu, folders, org chart) | **Composite** | leaf and group same interface |
| **Expensive object**, rarely needed | **Lazy proxy / holder idiom** | create on first use |
| **Millions of similar small objects** | **Flyweight** | share intrinsic state |
| **Same algorithm, steps differ** (ETL for CSV/JSON/XML) | **Template Method** (or Strategy with lambdas) | fixed skeleton |
| **Many objects talking to each other** (chat, UI form) | **Mediator** | one hub |
| **Avoid null checks** for optional collaborator | **Null Object / Optional** | no-op implementation |

**How to answer in the interview (framework):** (1) restate the *problem* and what changes often, (2) name the pattern and the one-sentence why, (3) sketch the interface + 2 implementations, (4) mention the trade-off or the simpler alternative (a lambda, a switch, a plain method).

**Pitfalls / follow-up probes:**
- "Patterns are not goals" — forcing patterns where a function would do is itself an anti-pattern (over-engineering).
- Many patterns are the same shape (Strategy / State / Bridge / Decorator / Proxy) — **intent** distinguishes them.
- Language features replace patterns: lambdas (Strategy, Command, Template), enums (Singleton, Strategy), records (Value Object/DTO), sealed + switch (Visitor), `Optional` (Null Object), `try-with-resources` (Dispose).

**Remember:** *Name the force, pick the pattern, prefer the simplest thing that still works.*

---

### Q199. Low-level design: Parking Lot and Vending Machine (class design + core code).

**Say this first:** "I clarify requirements, find the nouns (entities) and verbs (operations), put variation behind interfaces, then write the core flow. Parking lot = Strategy for pricing; Vending machine = State."

**Answer — Parking Lot**

*Requirements:* multiple vehicle types, fixed number of spots per type, issue ticket on entry, compute fee on exit by duration, thread-safe, full-lot handling.

```
ParkingLot ──has many──▶ ParkingSpot (id, VehicleType)
    │  issues                       
    ▼                              
  Ticket (id, spotId, type, entryTime) ──▶ PricingStrategy (HourlyPricing / FlatPricing)
```
```java
enum VehicleType { BIKE, CAR, TRUCK }
record Ticket(String id, ParkingSpot spot, VehicleType type, Instant entry) { }
record ParkingSpot(String id, VehicleType type) { }

interface PricingStrategy { long fee(VehicleType t, Duration parked); }          // Strategy: pricing changes often
class HourlyPricing implements PricingStrategy {
    private static final Map<VehicleType, Long> RATE = Map.of(VehicleType.BIKE, 10L, VehicleType.CAR, 30L, VehicleType.TRUCK, 60L);
    public long fee(VehicleType t, Duration d) {
        long hours = Math.max(1, (d.toMinutes() + 59) / 60);                      // round up, minimum 1 hour
        return hours * RATE.get(t);
    }
}

class ParkingLot {
    private final Map<VehicleType, Deque<ParkingSpot>> free = new EnumMap<>(VehicleType.class);
    private final Map<String, Ticket> active = new HashMap<>();
    private final PricingStrategy pricing; private final Clock clock;

    ParkingLot(List<ParkingSpot> spots, PricingStrategy pricing, Clock clock) {
        this.pricing = pricing; this.clock = clock;
        for (VehicleType t : VehicleType.values()) free.put(t, new ArrayDeque<>());
        spots.forEach(s -> free.get(s.type()).add(s));
    }
    synchronized Ticket park(VehicleType type) {                                  // one lock: simple and correct
        ParkingSpot spot = free.get(type).poll();                                  // O(1) allocation
        if (spot == null) throw new LotFullException(type);
        Ticket t = new Ticket(UUID.randomUUID().toString(), spot, type, clock.instant());
        active.put(t.id(), t);
        return t;
    }
    synchronized long exit(String ticketId) {
        Ticket t = active.remove(ticketId);
        if (t == null) throw new IllegalArgumentException("invalid ticket");
        free.get(t.type()).add(t.spot());                                          // release spot
        return pricing.fee(t.type(), Duration.between(t.entry(), clock.instant()));
    }
}
```
Extensions: spots by size (a bike can use a car spot — pick smallest fit), multi-floor (`Floor` holds free queues; lot picks nearest floor), entry/exit gates, payment (inject `PaymentGateway`), `Clock` injected so fee tests are deterministic.

**Answer — Vending Machine (State pattern)**

*Idea:* behaviour depends on state (idle, has money). Instead of `if (state == …)` everywhere, each state is a class that handles each event.

```
VendingMachine (balance, slots, current State) ◀── State { insertCoin, select, cancel }
                                                      ├─ IdleState
                                                      └─ HasMoneyState
```
```java
interface State {
    void insertCoin(VendingMachine m, int amount);
    void select(VendingMachine m, String code);
    void cancel(VendingMachine m);
}
class VendingMachine {
    record Slot(String name, int price, int[] qty) { }                 // qty in a 1-element array just to keep the sample short
    private State state = new IdleState();
    private int balance;
    private final Map<String, Slot> slots;
    VendingMachine(Map<String, Slot> slots) { this.slots = slots; }

    void insertCoin(int a)  { state.insertCoin(this, a); }
    void select(String c)   { state.select(this, c); }
    void cancel()           { state.cancel(this); }

    void setState(State s) { this.state = s; }
    void addBalance(int a) { balance += a; }
    int refundAll()        { int b = balance; balance = 0; return b; }
    Slot slot(String code) { return slots.get(code); }
    int balance()          { return balance; }
    void dispense(Slot s)  { s.qty()[0]--; balance -= s.price(); }
}
class IdleState implements State {
    public void insertCoin(VendingMachine m, int a) { m.addBalance(a); m.setState(new HasMoneyState()); }
    public void select(VendingMachine m, String c)  { throw new IllegalStateException("Insert money first"); }
    public void cancel(VendingMachine m)            { /* nothing to refund */ }
}
class HasMoneyState implements State {
    public void insertCoin(VendingMachine m, int a) { m.addBalance(a); }
    public void select(VendingMachine m, String code) {
        var s = m.slot(code);
        if (s == null || s.qty()[0] == 0) throw new IllegalArgumentException("Unavailable");
        if (m.balance() < s.price()) throw new IllegalStateException("Need " + (s.price() - m.balance()) + " more");
        m.dispense(s);
        System.out.println("Dispensed " + s.name() + ", change " + m.refundAll());
        m.setState(new IdleState());
    }
    public void cancel(VendingMachine m) { System.out.println("Refund " + m.refundAll()); m.setState(new IdleState()); }
}
```
Adding "OutOfService" or "Maintenance" = one new `State` class, no edits to the others (Open/Closed). A simpler alternative for small machines: `enum State` with a `switch` — use the class form when states have lots of behaviour.

**Pitfalls / follow-up probes:**
- State vs Strategy: Strategy is chosen by the *client*; State *changes itself* as events occur.
- Concurrency: parking lot uses one lock (fine for correctness); scale by per-type locks or `ConcurrentLinkedQueue` per type.
- Money: use `long` paise or `BigDecimal`, never `double`.
- Always ask: single vs multiple floors? Reservations? Payment modes? Concurrent users? — clarifying questions score points.

**Remember:** *Parking lot = entities + Strategy for fees + one lock; Vending machine = State classes per mode.*

---

### Q200. Low-level design: LRU Cache and Rate Limiter.

**Say this first:** "LRU = HashMap + doubly linked list for O(1) get/put/evict. Rate limiter = token bucket with lazy refill."

**Answer — LRU Cache**

*Requirement:* fixed capacity; `get` and `put` in O(1); evict the **least recently used** entry when full.

**Shortcut with the JDK** — `LinkedHashMap` in access-order mode:
```java
class LruCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;
    LruCache(int capacity) { super(16, 0.75f, true); this.capacity = capacity; }   // true = access order
    @Override protected boolean removeEldestEntry(Map.Entry<K, V> eldest) { return size() > capacity; }
}
Map<String, String> cache = Collections.synchronizedMap(new LruCache<>(100));      // NOT thread-safe by itself
```

**From scratch (what interviewers want to see):**
```
HashMap<K, Node>  ──O(1) lookup──▶  Node ⇄ Node ⇄ Node   (head = most recent, tail = least recent)
```
```java
final class LruCache2<K, V> {
    private static final class Node<K, V> { K key; V val; Node<K, V> prev, next; Node(K k, V v) { key = k; val = v; } }
    private final int capacity;
    private final Map<K, Node<K, V>> map = new HashMap<>();
    private final Node<K, V> head = new Node<>(null, null), tail = new Node<>(null, null);   // sentinels: no null checks

    LruCache2(int capacity) { this.capacity = capacity; head.next = tail; tail.prev = head; }

    synchronized V get(K key) {
        Node<K, V> n = map.get(key);
        if (n == null) return null;
        unlink(n); addFirst(n);                    // mark as most recently used
        return n.val;
    }
    synchronized void put(K key, V val) {
        Node<K, V> n = map.get(key);
        if (n != null) { n.val = val; unlink(n); addFirst(n); return; }
        if (map.size() == capacity) {              // evict least recently used = node before tail
            Node<K, V> lru = tail.prev; unlink(lru); map.remove(lru.key);
        }
        n = new Node<>(key, val); map.put(key, n); addFirst(n);
    }
    private void unlink(Node<K, V> n) { n.prev.next = n.next; n.next.prev = n.prev; }
    private void addFirst(Node<K, V> n) { n.next = head.next; n.prev = head; head.next.prev = n; head.next = n; }
}
```
Why a *doubly* linked list: to remove a node from the middle in O(1) you need its `prev`. Why a map: to find the node in O(1). For high concurrency, use sharded caches or Caffeine instead of one lock.

**Answer — Rate Limiter (Token Bucket)**

*Idea:* a bucket holds up to `capacity` tokens; tokens refill at a fixed rate; each request takes one. Allows short **bursts** but caps the average rate. Refill lazily (compute on each call) — no background thread.
```java
final class TokenBucket {
    private final long capacity; private final double refillPerNano;
    private double tokens; private long lastNanos;

    TokenBucket(long capacity, double tokensPerSecond) {
        this.capacity = capacity; this.refillPerNano = tokensPerSecond / 1_000_000_000.0;
        this.tokens = capacity; this.lastNanos = System.nanoTime();
    }
    synchronized boolean tryAcquire() {
        long now = System.nanoTime();
        tokens = Math.min(capacity, tokens + (now - lastNanos) * refillPerNano);   // lazy refill
        lastNanos = now;
        if (tokens >= 1) { tokens -= 1; return true; }
        return false;                                                              // caller returns HTTP 429
    }
}
class PerUserLimiter {
    private final ConcurrentMap<String, TokenBucket> buckets = new ConcurrentHashMap<>();
    boolean allow(String userId) {
        return buckets.computeIfAbsent(userId, id -> new TokenBucket(10, 5)).tryAcquire();   // 5 req/s, burst 10
    }
}
```
**Algorithm comparison:**
| Algorithm | Behaviour | Trade-off |
|---|---|---|
| **Token bucket** | allows bursts, smooth average | most common; simple |
| **Leaky bucket** | constant output rate | no bursts; queue needed |
| **Fixed window counter** | count per minute | spike at window boundary (2× limit possible) |
| **Sliding window log/counter** | accurate | more memory/CPU |

**Distributed:** in-process limiters only limit one node. For a cluster keep counters in a shared store (e.g. Redis with an atomic script) or use an API gateway.

**Pitfalls / follow-up probes:**
- Use `System.nanoTime()` (monotonic) for intervals, **not** `currentTimeMillis()` (can jump).
- `computeIfAbsent` + unbounded per-user map = memory leak; expire idle entries.
- Make the cache/limiter take `Clock`/time source for deterministic tests.
- LRU variants: LFU (least frequently used), TTL expiry, write-through; thread-safety via lock striping.
- `LinkedHashMap` is not thread-safe, and `get()` in access-order mode **modifies** structure — even readers need the lock.

**Remember:** *LRU = HashMap + doubly linked list; Token bucket = lazy-refilled counter with burst capacity.*

---

### Q201. Effective Java — which items matter most in day-to-day code?

**Say this first:** "My top items: static factories, builders, immutability, composition, equals/hashCode contract, try-with-resources, enums, interfaces for types, and never returning null collections."

**Answer:** (Item numbers are from *Effective Java*, 3rd edition.)

- **Item 1 — Static factory methods instead of constructors.** They have names (`BigInteger.valueOf`), can return cached instances (`Boolean.valueOf`), can return subtypes (`List.of` returns different internal classes), and hide implementation. Downside: not obvious in docs; classes with only private constructors can't be subclassed.
```java
public record Money(long paise) {
    public static Money ofRupees(long r) { return new Money(r * 100); }
    public static final Money ZERO = new Money(0);
}
```
- **Item 2 — Builder for many parameters** (see the Builder question).
- **Item 8 — Avoid finalizers and cleaners.** `finalize()` is unpredictable, slow, a security hole, and **deprecated** (for removal since Java 18). Use `AutoCloseable` + try-with-resources; `java.lang.ref.Cleaner` (Java 9) only as a safety net.
- **Items 10/11/12 — `equals`, `hashCode`, `toString`.** Override `equals` → **must** override `hashCode` (equal objects ⇒ equal hash codes), else `HashMap`/`HashSet` break. `equals` must be reflexive, symmetric, transitive, consistent, and false for `null`. Always override `toString`.
```java
@Override public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Point p)) return false;                // pattern matching instanceof (Java 16)
    return x == p.x && y == p.y;
}
@Override public int hashCode() { return Objects.hash(x, y); }
@Override public String toString() { return "Point[x=" + x + ", y=" + y + "]"; }
```
Records generate all three for you.
- **Item 17 — Minimize mutability.** No setters, `final` class and fields, defensive copies of mutable components. Immutable = thread-safe, cacheable, simple (`String`, `LocalDate`, `BigDecimal`).
- **Item 18 — Favor composition over inheritance** (a wrapper that delegates beats a subclass of a class not designed for it).
- **Item 34 — Use enums instead of `int` constants.** Type-safe, can have fields/methods, work in `switch`, with `EnumMap`/`EnumSet`.
```java
// ❌ public static final int STATUS_NEW = 0, STATUS_PAID = 1;   ← any int compiles, no meaning in logs
enum Status { NEW, PAID, SHIPPED; boolean isFinal() { return this == SHIPPED; } }
Map<Status, Long> counts = new EnumMap<>(Status.class);
```
- **Item 9 — Prefer try-with-resources to try-finally.** Closes in reverse order, and keeps the primary exception while attaching close failures as *suppressed* exceptions.
```java
try (var in = Files.newBufferedReader(src); var out = Files.newBufferedWriter(dst)) {
    in.transferTo(out);                                         // both closed automatically, even on error
}
```
- **Item 54 — Return empty collections (or arrays), not `null`.** Callers then loop without null checks. Use `Collections.emptyList()` / `List.of()`.
```java
List<Order> findByUser(long id) { var r = dao.query(id); return r == null ? List.of() : r; }
```
- **Item 64 — Refer to objects by their interfaces.** `Map<String,Integer> m = new HashMap<>();` — easy to swap to `LinkedHashMap`/`ConcurrentHashMap`.
- **Item 55 — Return `Optional` judiciously** — for "might not have a result" return values; not for fields, parameters, or collections.
- **Item 50 — Make defensive copies** when storing/returning mutable parameters (`new Date(d.getTime())`, `List.copyOf(list)`).
- **Item 3 — Singleton with a single-element enum** (see singleton questions).

**Pitfalls / follow-up probes:**
- `equals` with `getClass()` vs `instanceof`: `instanceof` is the usual choice, but with subclasses adding fields no override can satisfy the contract — another reason to prefer composition.
- Mutable fields in `hashCode` + `HashSet` → object "disappears" after mutation.
- `hashCode` of `Objects.hash(...)` boxes/allocates; in hot paths compute manually (`31 * a + b`).
- Why is `finalize()` bad? Unpredictable timing, may never run, resurrection, GC overhead, exceptions swallowed.
- Overriding `equals(Point p)` (wrong parameter type) **overloads** instead of overrides — always use `@Override`.

**Remember:** *Static factories, builders, immutability, composition, equals+hashCode together, try-with-resources, enums, interfaces, never null collections.*

---

# 8. Generics, Annotations, Reflection, I/O, Serialization & Date-Time

### Q202. Why do generics exist, and what is type erasure (with its consequences)?

**Say this first:** "Generics give compile-time type safety; the compiler then *erases* the types, so at runtime a `List<String>` is just a `List`."

**Answer:**
- **Why generics:** before Java 5 a `List` held `Object`. You cast on the way out and got a `ClassCastException` at runtime. Generics move that error to **compile time** and remove the casts.
- **Type erasure:** to stay backward-compatible with old bytecode, the compiler checks the types, then **removes** them. `T` becomes its bound (or `Object`), and the compiler inserts casts where needed.

```java
List<String> names = new ArrayList<>();
names.add("a");
String s = names.get(0);          // compiler emits: (String) names.get(0)
// At runtime: names.getClass() == new ArrayList<Integer>().getClass()  -> true
```

**Consequences of erasure (classic interview list):**

| You cannot do | Why |
|---|---|
| `new T()` / `new T[10]` | `T` does not exist at runtime |
| `obj instanceof List<String>` | only `List<?>` is checkable (see *reifiable types* in the raw-types/heap-pollution question) |
| `catch (MyEx<T> e)`, generic `Throwable` subclass | not allowed; JVM can't tell them apart |
| `static T field` | one static field is shared by all `T`s |
| overload `void f(List<String>)` and `void f(List<Integer>)` | same erasure → "name clash" compile error |
| `T.class` | use a `Class<T>` token parameter instead |

**Bridge methods:** erasure can break overriding. The compiler generates a hidden *bridge method* to keep polymorphism working.

```java
class Box<T> { void set(T t) { } }          // erased: set(Object)
class IntBox extends Box<Integer> {
    @Override void set(Integer i) { }       // erased: set(Integer)
}
// Compiler adds in IntBox:  synthetic bridge  set(Object o) { set((Integer) o); }
```
You can see it with `javap -p IntBox` (marked `synthetic bridge`). This is why a bad cast through a raw type fails with `ClassCastException` *inside the bridge*.

**What is preserved:** generic info in **signatures** of classes, fields, methods (`Field.getGenericType()`). That is how Jackson's `TypeReference<List<Foo>>` and Guice's `TypeLiteral` work — an anonymous subclass `new TypeReference<List<Foo>>(){}` keeps the type in its superclass signature ("super type token").

**Pitfalls / follow-up probes:**
- "Why did Java choose erasure?" → migration compatibility; no changes to the JVM; one class file for all instantiations (C++ templates generate code per type).
- Primitives can't be type arguments (`List<int>`) → boxing cost; Project Valhalla aims to improve this.
- Erasure means **no runtime check** → see heap pollution in the raw-types question.

**Remember:** *Generics are a compile-time contract; at runtime only the erasure (and bridge methods) remain.*

---

### Q203. Explain bounded types, generic methods and wildcards, including the PECS rule.

**Say this first:** "Producer Extends, Consumer Super — `? extends T` to read from, `? super T` to write into."

**Answer:**

**Bounded type parameter** limits what `T` can be:
```java
static <T extends Comparable<? super T>> T max(Collection<? extends T> c) {
    Iterator<? extends T> it = c.iterator();
    T best = it.next();
    while (it.hasNext()) { T x = it.next(); if (x.compareTo(best) > 0) best = x; }
    return best;
}
```
- `<T extends A & B>` → multiple bounds (class first, then interfaces).
- **Generic method:** the type parameter is declared before the return type (`<T> T pick(T a, T b)`), and the compiler **infers** `T` from the arguments.

**Why wildcards?** Generics are **invariant**: `List<Integer>` is *not* a `List<Number>` (otherwise you could add a `Double` to a list of `Integer`). Arrays are covariant (`Integer[]` is a `Number[]`) and fail at runtime with `ArrayStoreException`. Wildcards give *safe* flexibility.

| Syntax | Meaning | Can read as | Can write |
|---|---|---|---|
| `List<? extends Number>` | list of some unknown subtype of Number | `Number` | only `null` |
| `List<? super Integer>` | list of some unknown supertype of Integer | `Object` | `Integer` and subtypes |
| `List<?>` | unknown type | `Object` | only `null` |

**PECS — Producer Extends, Consumer Super** (Joshua Bloch):
```java
// src PRODUCES items for us -> extends ; dst CONSUMES items -> super
static <T> void copy(List<? super T> dst, List<? extends T> src) {
    for (T t : src) dst.add(t);
}
copy(new ArrayList<Number>(), List.of(1, 2, 3));   // Integer -> Number: OK
```
JDK examples: `Collections.copy`, `Comparator<? super T>` in `sort`, `Stream.map(Function<? super T, ? extends R>)`.

**Pitfalls / follow-up probes:**
- Don't use wildcards in **return types** (forces callers to deal with them).
- **Wildcard capture:** `void swap(List<?> l)` can't call `l.set(0, l.get(1))`; delegate to a private generic helper `<T> void swapHelper(List<T> l)`.
- `Comparable<? super T>` is needed so `java.sql.Timestamp` (extends `Date`, which implements `Comparable<Date>`) works.
- Unbounded `?` is used when you only need `Object` methods or `size()`.

**Remember:** *Get = extends, Put = super, both = no wildcard.*

---

### Q204. What are raw types, unchecked warnings, heap pollution, `@SafeVarargs` and reifiable types?

**Say this first:** "Raw types switch off the compiler's generic checks; heap pollution is when a variable holds an object of the wrong type — and the failure appears far away."

**Answer:**

- **Raw type:** using a generic class without arguments (`List list`). It exists only for legacy code. The compiler stops checking and issues an **unchecked warning**.
- **Heap pollution:** a variable of type `List<String>` actually refers to a list holding an `Integer`. Nothing fails at the bad write; a `ClassCastException` appears later at the compiler-inserted cast.

```java
List<String> strings = new ArrayList<>();
List raw = strings;
raw.add(42);                    // unchecked warning, compiles
String s = strings.get(0);      // ClassCastException here, far from the cause
```

- **Reifiable type:** a type whose full info is available at runtime: primitives, non-generic classes, raw types, `List<?>`, arrays of these. **Non-reifiable:** `List<String>`, `T`, `List<? extends Number>`.
  - Hence `instanceof List<String>` is illegal, and `new List<String>[10]` is illegal (generic array creation).
- **Why no generic arrays?** Arrays check element type at runtime (reified), generics don't. Allowing both would make `ArrayStoreException` protection unsound. Workarounds: `List<List<String>>`, or `(T[]) new Object[n]` with care (this is what `ArrayList` does internally).
- **Varargs + generics:** `static <T> void f(T... xs)` creates a `T[]` (really `Object[]`) → possible heap pollution → warning at declaration and at every call.
- **`@SafeVarargs`:** you promise the method does not store into the varargs array nor leak it. Allowed on `static`, `final`, `private` methods and constructors. Examples: `Arrays.asList`, `List.of`.

```java
@SafeVarargs
static <T> List<T> listOf(T... items) { return new ArrayList<>(Arrays.asList(items)); }
```

- `@SuppressWarnings("unchecked")` — use on the **smallest scope** (a single local variable assignment) with a comment explaining why the cast is safe.

**Pitfalls / follow-up probes:**
- `Collections.checkedList(list, String.class)` adds a runtime check to catch pollution at the point of the bad write.
- `Class<T>.cast()` is a safe runtime-checked cast.
- Never expose a `T[]` created via `(T[]) new Object[n]` in a public API: callers get `ClassCastException`.

**Remember:** *Raw types = no safety net; pollution fails late; reifiable = survives erasure.*

---

### Q205. What are built-in and meta-annotations? Explain `@Retention`, `@Target`, `@Inherited`, `@Repeatable`.

**Say this first:** "Annotations are metadata with no behaviour of their own; a tool (compiler, annotation processor, or reflection code) gives them meaning."

**Answer:**

**Common built-in annotations:**
| Annotation | Purpose |
|---|---|
| `@Override` | compiler verifies you really override something |
| `@Deprecated(since="9", forRemoval=true)` | warns users; `forRemoval` gives a stronger warning |
| `@SuppressWarnings("unchecked")` | silence specific warnings |
| `@FunctionalInterface` | compiler checks exactly one abstract method |
| `@SafeVarargs` | promise a generic varargs method is safe (see the raw-types/heap-pollution question) |

**Meta-annotations** (annotations that annotate annotations):

| Meta-annotation | Controls | Notes |
|---|---|---|
| `@Retention` | how long it lives: `SOURCE` (compiler only, e.g. `@Override`), `CLASS` (in `.class`, not at runtime — **the default**), `RUNTIME` (readable by reflection) | frameworks need `RUNTIME` |
| `@Target` | where it can be placed: `TYPE`, `METHOD`, `FIELD`, `PARAMETER`, `CONSTRUCTOR`, `TYPE_USE`, `ANNOTATION_TYPE`… | no `@Target` = most declaration contexts |
| `@Inherited` | subclasses inherit the annotation from the superclass | **only class-level**; not from interfaces, not methods |
| `@Repeatable` | same annotation can appear multiple times | needs a *container annotation* |
| `@Documented` | include in Javadoc | |

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@Repeatable(Roles.class)
@interface Role { String value(); }

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Roles { Role[] value(); }

@Role("ADMIN") @Role("AUDITOR")
void deleteAll() { }
// m.getAnnotationsByType(Role.class) -> both
```
- Annotation elements can only be primitives, `String`, `Class`, enums, annotations, and arrays of those; `default` values allowed. A single element named `value` can be written as `@Role("X")`.
- Java 8 added **type annotations** (`List<@NonNull String>`, `TYPE_USE`) used by checker tools.

**Pitfalls / follow-up probes:**
- Forgetting `RUNTIME` retention → `isAnnotationPresent` returns false (very common bug).
- Annotations can't extend each other or have `null` values.
- Interview twist: "Does `@Inherited` work on interface annotations?" — No.

**Remember:** *Retention = how long, Target = where, Inherited = down the class tree, Repeatable = more than once.*

---

### Q206. How do you write and process a custom annotation? What are annotation processors (Lombok/MapStruct idea)?

**Answer:**

**Two ways annotations get processed:**
1. **Runtime, via reflection** — a framework scans classes/methods when the app starts (Spring, JUnit, Jackson, Bean Validation).
2. **Compile time, via annotation processors** — run inside `javac`, generate code or validate source.

**1. Custom runtime annotation + processor (tiny validator):**
```java
@Retention(RetentionPolicy.RUNTIME) @Target(ElementType.FIELD)
@interface NotBlank { String message() default "must not be blank"; }

class User { @NotBlank String name; int age; }

static List<String> validate(Object o) throws IllegalAccessException {
    List<String> errors = new ArrayList<>();
    for (Field f : o.getClass().getDeclaredFields()) {
        NotBlank nb = f.getAnnotation(NotBlank.class);
        if (nb != null) {
            f.setAccessible(true);
            Object v = f.get(o);
            if (v == null || v.toString().isBlank())
                errors.add(f.getName() + " " + nb.message());
        }
    }
    return errors;
}
```
Steps: define annotation (`RUNTIME`) → mark code → read with `getAnnotation` → act.

**2. Annotation processors (JSR 269):**
- A class extending `AbstractProcessor`, registered in `META-INF/services/javax.annotation.processing.Processor` (or via `@AutoService`). `javac` calls it in **rounds** with the annotated elements.
- It can **generate new source files** (via `Filer`) or emit errors/warnings (via `Messager`). Generated sources are compiled in the next round.
- **MapStruct:** generates plain mapper implementation classes → no reflection at runtime, fast and debuggable.
- **Dagger:** generates DI wiring at compile time. **Immutables/AutoValue:** generate value classes.
- **Lombok:** goes further — it hooks into `javac` internals and *modifies the AST* of your class (adds getters, builders). This relies on non-public compiler APIs, so IDEs/JDK upgrades need a compatible Lombok.

| | Reflection (runtime) | Annotation processor (compile time) |
|---|---|---|
| When | startup / runtime | build |
| Cost | startup time, no static checking | none at runtime |
| Errors | at runtime | at compile time |
| GraalVM native-image friendly | needs config | yes |

**Pitfalls / follow-up probes:**
- Processors can't modify existing classes (by the official API), only create new ones.
- Maven: processors must be on `annotationProcessorPaths`; with Lombok + MapStruct ordering matters (`lombok-mapstruct-binding`).
- Why compile-time is trending: faster startup, native images, fewer runtime surprises.

**Remember:** *Runtime annotations are read by reflection; compile-time annotations are read by `javac` plug-ins that write code.*

---

### Q207. What is reflection, what can you do with it, and what are its costs, JPMS limits and security risks?

**Say this first:** "Reflection lets code inspect and invoke classes, methods and fields **by name at runtime** — it powers every framework, but it bypasses compile-time safety."

**Answer:**

**Entry point:** `Class<?>` object: `Foo.class`, `obj.getClass()`, `Class.forName("com.acme.Foo")`.

```java
Class<?> c = Class.forName("com.acme.Account");
Object acc = c.getDeclaredConstructor(String.class).newInstance("A-1");

Field f = c.getDeclaredField("balance");       // getDeclared* = any visibility, this class only
f.setAccessible(true);                          // bypass private
f.set(acc, 500L);

Method m = c.getDeclaredMethod("audit", int.class);
m.setAccessible(true);
Object r = m.invoke(acc, 3);                    // wraps exceptions in InvocationTargetException
```

| API | Returns |
|---|---|
| `getFields()/getMethods()` | **public**, including inherited |
| `getDeclaredFields()/getDeclaredMethods()` | **all** visibilities, only this class (not inherited) |
| `getModifiers()`, `getGenericType()`, `getAnnotations()`, `getRecordComponents()` | metadata |

**Real uses:** DI containers (create beans, inject private fields), ORMs (Hibernate populates entity fields), JSON mappers (Jackson), JUnit (finds `@Test`), serializers, plugin loading, test tools (Mockito).

**Costs:**
- Slower than direct calls: access checks, boxing of arguments, no JIT inlining. Modern JVMs (since JDK 18, JEP 416) reimplemented core reflection on **method handles**, which narrowed the gap, but it is still slower than a direct call.
- Breaks refactoring/IDE safety (names as strings), hides errors until runtime.
- **Cache** `Method`/`Field` objects; do the lookup once, call many times.

**JPMS (Java 9+) restrictions:**
- `setAccessible(true)` on a private member of a class in another **module** works only if that package is **opened** to you (`opens` in `module-info`, or `--add-opens java.base/java.lang=ALL-UNNAMED`).
- JDK internals: Java 9–15 were lenient (`--illegal-access=permit`); **Java 16 denied by default, Java 17 removed the flag** (JEP 396/403). You now get `InaccessibleObjectException`. This is why old libraries need `--add-opens`.
- You cannot modify `final` fields of **records** or hidden classes through reflection.

**Security concerns:** reflection can read private data and call internals; untrusted input used in `Class.forName(userInput)` or `Method.invoke` is a code-execution risk (gadget chains in deserialization, see the deserialization-security question). Validate against an allow-list. The old `SecurityManager` is deprecated for removal (JEP 411).

**Pitfalls / follow-up probes:**
- `InvocationTargetException` → call `getCause()` for the real exception.
- Reflection on `static final` compile-time constants: the value was inlined at compile time, so changing it has no effect on callers.
- GraalVM native image needs reflection metadata because of closed-world analysis.

**Remember:** *Reflection = power by string name; cost = speed, safety, encapsulation.*

---

### Q208. How do dynamic proxies (`java.lang.reflect.Proxy`) work, and what are `MethodHandle` and `VarHandle`?

**Answer:**

**Dynamic proxy:** at runtime the JDK generates a class that implements the **interfaces** you give it; every call is routed to one `InvocationHandler`. It is the basis of AOP, declarative transactions, remote clients and mock libraries.

```java
interface PaymentService { String pay(String id, long amount); }

class RealPayment implements PaymentService {
    public String pay(String id, long amount) { return "PAID-" + id; }
}

PaymentService target = new RealPayment();
PaymentService proxy = (PaymentService) Proxy.newProxyInstance(
    PaymentService.class.getClassLoader(),
    new Class<?>[] { PaymentService.class },
    (p, method, args) -> {
        long t0 = System.nanoTime();
        try {
            return method.invoke(target, args);            // delegate
        } catch (InvocationTargetException e) {
            throw e.getCause();                             // unwrap real exception
        } finally {
            System.out.println(method.getName() + " took " + (System.nanoTime() - t0) + "ns");
        }
    });
proxy.pay("42", 100);
```
How it works: `newProxyInstance` → JDK builds `$Proxy0 implements PaymentService` → each method calls `handler.invoke(proxy, method, args)`.

| | JDK proxy | CGLIB / ByteBuddy |
|---|---|---|
| Needs | an **interface** | a **non-final class** (creates a subclass) |
| Limit | only interface methods are intercepted | can't intercept `final` methods/classes |

(This is why a framework that proxies classes complains about `final` classes and why self-invocation `this.foo()` bypasses the proxy: the call never goes through the proxy object.)

**`MethodHandle` (Java 7):** a typed, directly executable reference to a method/field/constructor, found via `MethodHandles.lookup()`. Access checks happen at **lookup**, not on each call, and the JIT can inline `invokeExact` — close to direct-call speed. It is the foundation of `invokedynamic` (lambdas, string concatenation, records' `toString`).
```java
MethodHandle mh = MethodHandles.lookup()
    .findVirtual(String.class, "length", MethodType.methodType(int.class));
int n = (int) mh.invokeExact("hello");   // 5; exact signature required
```

**`VarHandle` (Java 9):** typed handle to a variable (field, array element) with fine-grained memory-ordering modes (plain, opaque, acquire/release, volatile) and atomic ops (`compareAndSet`, `getAndAdd`). It replaces `sun.misc.Unsafe` in `java.util.concurrent` internals (e.g. `AtomicXxx`, `ConcurrentHashMap`).

**Pitfalls / follow-up probes:**
- Proxy handler calls `method.invoke(proxy, …)` → infinite recursion; invoke on the **target**.
- Default interface methods need `InvocationHandler.invokeDefault` (Java 16+).
- Checked exceptions thrown that the interface does not declare become `UndeclaredThrowableException`.

**Remember:** *Proxy = runtime-generated implementation of interfaces that funnels every call to one handler.*

---

### Q209. Byte streams vs character streams: how is `java.io` designed (Decorator pattern) and why use try-with-resources?

**Say this first:** "Bytes for binary data, characters for text — and always specify the charset when bridging them."

**Answer:**

| | Byte streams | Character streams |
|---|---|---|
| Base classes | `InputStream` / `OutputStream` | `Reader` / `Writer` |
| Unit | 8-bit byte | 16-bit `char` (UTF-16 code unit) |
| Use for | images, zip, network, serialized objects | text |
| Bridge | — | `InputStreamReader(in, charset)` / `OutputStreamWriter(out, charset)` decode/encode |

**Decorator pattern:** each wrapper adds one feature and *is-a* stream itself, so you stack them.
```java
try (var in = new DataInputStream(
         new BufferedInputStream(
             new FileInputStream("data.bin")))) {   // File -> Buffer -> typed reads
    int id = in.readInt();
}
try (var w = new PrintWriter(new BufferedWriter(
        new OutputStreamWriter(new FileOutputStream("a.txt"), StandardCharsets.UTF_8)))) {
    w.println("hello");
}
```
Common decorators: `Buffered*` (fewer system calls), `Data*` (primitives), `Object*` (serialization), `GZIP*`/`Zip*` (compression), `PushbackInputStream`.

**Why buffering?** Every `read()` on a raw `FileInputStream` is a system call. `BufferedInputStream` reads e.g. 8 KB at once and serves bytes from memory. Reading byte-by-byte unbuffered can be orders of magnitude slower.

**try-with-resources (Java 7):** any `AutoCloseable` declared in the `try (...)` is closed automatically in **reverse order**, even on exception. Exceptions thrown by `close()` are attached as **suppressed** exceptions (`getSuppressed()`), so the original is not lost (the old `finally { close() }` could overwrite it). Java 9 allows `try (existingFinalVar)`.

**Pitfalls / follow-up probes:**
- `new FileReader(file)` used the platform default charset before Java 18; Java 11 added `FileReader(file, charset)`. Always pass `StandardCharsets.UTF_8`.
- `InputStream.read()` returns `int` 0–255 or `-1` at EOF (that's why not `byte`).
- `readAllBytes()` (Java 9) and `transferTo(out)` (Java 9) simplify copying.
- Closing the outermost decorator closes the inner ones; flush happens on close of buffered writers.
- `System.out` is a `PrintStream` that **swallows** `IOException` (check `checkError()`).

**Remember:** *Stack decorators (Buffer around Raw), pick bytes vs chars by data, let try-with-resources close.*

---

### Q210. How do you read a very large file efficiently without running out of memory?

**Answer:**
Principle: **stream**, don't load. Memory should be proportional to a line/chunk, not file size.

| Technique | Memory | Notes |
|---|---|---|
| `Files.readAllLines(p)` / `readString` / `readAllBytes` | whole file | only for small files |
| `BufferedReader.readLine()` | one line | classic, easy |
| `Files.lines(p)` / `Files.newBufferedReader(p)` | lazy stream | **must close** (try-with-resources); throws `UncheckedIOException` |
| `InputStream` + `byte[]` chunk loop / `FileChannel` + `ByteBuffer` | one buffer | binary data |
| `FileChannel.map(...)` (memory-mapped) | OS page cache | random access of huge files |
| `FileChannel.transferTo` | none in heap | zero-copy file→socket/file |

```java
try (Stream<String> lines = Files.lines(Path.of("big.log"), StandardCharsets.UTF_8)) {
    long errors = lines.filter(l -> l.contains("ERROR")).count();
}

try (FileChannel ch = FileChannel.open(Path.of("big.bin"), StandardOpenOption.READ)) {
    MappedByteBuffer buf = ch.map(FileChannel.MapMode.READ_ONLY, 0, ch.size()); // max ~2 GB per mapping
    byte b = buf.get(1_000_000);
}
```

**How memory mapping works:** the OS maps file pages into the process address space; pages load lazily on access, and the OS decides which stay in memory. No copying between kernel and Java heap. Downsides: one mapping is limited to `Integer.MAX_VALUE` bytes (the newer FFM `MemorySegment` API lifts this — check JDK docs for your version), unmapping is not deterministic (relies on GC), and page faults give unpredictable latency.

**Other ideas:**
- Parallelise by splitting the file into byte ranges aligned to newline boundaries; `Files.lines(...).parallel()` splits poorly on older JDKs.
- Compressed: wrap with `GZIPInputStream`.
- Write large outputs with `BufferedWriter`; avoid building a giant `String`/`List` first.
- Don't use `Scanner` for huge input — it is slow (regex based).

**Pitfalls / follow-up probes:**
- Forgetting to close `Files.lines` leaks file handles.
- Wrong charset → `MalformedInputException` (`Files.lines` is strict, unlike `new String(bytes)` which silently replaces).
- Also consider: one `readLine()` on a file without newlines is an OOM risk. Guard line length.

**Remember:** *Stream with a buffer; map when you need random access; never `readAll` on unknown sizes.*

---

### Q211. Explain NIO: Buffers, Channels, Selectors — and blocking vs non-blocking vs asynchronous I/O.

**Say this first:** "One thread can watch thousands of connections with a Selector — that is how Netty and Tomcat's NIO connector scale."

**Answer:**

**Classic I/O (`java.io`)** is **stream-oriented and blocking**: `read()` parks the thread until data arrives → one thread per connection → thousands of idle threads.

**NIO (Java 1.4) building blocks:**
- **Buffer** — a block of memory with three indexes: `position`, `limit`, `capacity`. Typical cycle: **write into it → `flip()` → read from it → `clear()`/`compact()`**. `allocateDirect()` creates off-heap memory (avoids an extra copy on I/O; costly to allocate).
- **Channel** — a two-way connection to a file/socket (`FileChannel`, `SocketChannel`, `ServerSocketChannel`). Reads/writes go through buffers. Can be set **non-blocking**.
- **Selector** — one thread asks "which of my registered channels are ready?" (`OP_ACCEPT`, `OP_CONNECT`, `OP_READ`, `OP_WRITE`). Built on OS facilities (epoll on Linux, kqueue on macOS, IOCP/select on Windows).

```java
Selector sel = Selector.open();
ServerSocketChannel server = ServerSocketChannel.open();
server.bind(new InetSocketAddress(8080));
server.configureBlocking(false);
server.register(sel, SelectionKey.OP_ACCEPT);

ByteBuffer buf = ByteBuffer.allocate(1024);
while (true) {
    sel.select();                                   // blocks until something is ready
    var it = sel.selectedKeys().iterator();
    while (it.hasNext()) {
        SelectionKey k = it.next(); it.remove();    // must remove manually
        if (k.isAcceptable()) {
            SocketChannel c = server.accept();
            c.configureBlocking(false);
            c.register(sel, SelectionKey.OP_READ);
        } else if (k.isReadable()) {
            SocketChannel c = (SocketChannel) k.channel();
            buf.clear();
            if (c.read(buf) == -1) { c.close(); continue; }
            buf.flip(); c.write(buf);               // echo
        }
    }
}
```

**Three models compared:**

| Model | Who waits? | Example |
|---|---|---|
| **Blocking** | calling thread sleeps until done | `Socket`, `InputStream.read` |
| **Non-blocking (readiness, multiplexed)** | you poll/`select`; call returns immediately with 0 bytes if not ready; *you* do the read when ready | `Selector` + `SocketChannel` |
| **Asynchronous (completion)** | OS/runtime does the I/O and **notifies you when finished** (callback/Future) | NIO.2 `AsynchronousFileChannel`, `AsynchronousSocketChannel` |

**Real systems:** **Netty** runs a few *event-loop* threads, each with a Selector; handlers must never block them. **Tomcat's NIO connector** uses Selector threads for idle keep-alive connections and worker threads for request processing. Reactive stacks (WebFlux) sit on Netty.

**Pitfalls / follow-up probes:**
- Forgetting `flip()` (reads zero bytes) or `it.remove()` (key reprocessed forever).
- Doing slow work (DB call) on a selector thread starves all connections.
- **Virtual threads (Java 21)** let you write simple blocking code again; the JDK parks the virtual thread on NIO under the hood, so thread-per-request becomes cheap. Selector code is still the right tool for building servers/protocol frameworks.
- Files on Linux are not truly non-blocking via epoll — `FileChannel` is blocking; async file I/O is emulated with a thread pool.

**Remember:** *Buffer = data, Channel = pipe, Selector = one thread watching many pipes.*

---

### Q212. What is NIO.2 (`Path`, `Files`, `WatchService`) and how does it improve on `java.io.File`?

**Answer:**
Java 7 introduced `java.nio.file` to fix `File`'s problems: methods that return `boolean` with no reason for failure, no symbolic-link support, poor metadata, no directory walking.

| Need | Old (`File`) | NIO.2 |
|---|---|---|
| Represent a path | `File` | `Path` (immutable; `Path.of("a","b.txt")`) |
| Copy/move | hand-written loop / `renameTo()` (returns boolean) | `Files.copy`, `Files.move(..., ATOMIC_MOVE)` — throws descriptive exceptions |
| Read/write small files | streams | `Files.readString`, `writeString` (Java 11), `readAllLines`, `write` |
| Walk tree | recursion over `listFiles()` | `Files.walk`, `Files.walkFileTree` + `FileVisitor` |
| Metadata | `lastModified()`… | `Files.readAttributes`, POSIX permissions, `isSymbolicLink` |

```java
Path dir = Path.of("/data/in");
try (Stream<Path> s = Files.walk(dir)) {                    // must close
    s.filter(Files::isRegularFile).filter(p -> p.toString().endsWith(".csv"))
     .forEach(System.out::println);
}
Path tmp = Files.createTempFile("job", ".tmp");
Files.move(tmp, dir.resolve("out.csv"), StandardCopyOption.ATOMIC_MOVE);
Path rel = dir.relativize(dir.resolve("x/y.txt"));          // x/y.txt
```

**`WatchService`:** register a directory for `ENTRY_CREATE/MODIFY/DELETE` events and `take()` them in a loop.
```java
try (WatchService ws = FileSystems.getDefault().newWatchService()) {
    dir.register(ws, StandardWatchEventKinds.ENTRY_CREATE);
    while (true) {
        WatchKey key = ws.take();                           // blocks
        for (WatchEvent<?> e : key.pollEvents())
            System.out.println(e.kind() + " " + e.context());
        if (!key.reset()) break;                            // key invalid -> stop
    }
}
```
Uses OS notifications on Linux (inotify); on macOS the JDK implementation **polls** and can be slow. It is not recursive: register each subfolder. Events can be **coalesced or lost** (`OVERFLOW`), so treat it as a hint and re-scan. A file may be reported while still being written → wait for a "done" marker or atomic move.

**Pitfalls / follow-up probes:**
- `Files.walk`/`list`/`lines` hold a directory handle → close them.
- `Path.resolve` with an absolute argument returns the argument. **Path traversal:** `base.resolve(userInput).normalize()` then check `startsWith(base)` (zip-slip bug).
- `Files.move` fails if the target exists unless `REPLACE_EXISTING`.
- `Files.exists()` followed by an action is racy (TOCTOU) — just try and catch.

**Remember:** *`Path` + `Files` throw useful exceptions, walk trees, and watch folders — `File` does none of that well.*

---

### Q213. How does Java serialization work? Explain `Serializable`, `serialVersionUID`, `transient`, custom `writeObject/readObject` and `Externalizable`.

**Answer:**
**Serialization** converts an object graph into a byte stream (`ObjectOutputStream`); **deserialization** rebuilds it (`ObjectInputStream`). Used historically for RMI, HTTP session replication, caches.

**Rules:**
- Class implements the marker interface `Serializable` (no methods). Otherwise → `NotSerializableException`.
- **All fields** (and the object graph reachable from them) must be serializable, or be `transient`/`static` (not serialized; restored to default `0/null/false`).
- Deserialization **does not call the constructor** of the serializable class. It does call the no-arg constructor of the first **non-serializable superclass** (which must be accessible) — a favourite trick question.
- Shared references and cycles are preserved (object graph, not a tree).

**`serialVersionUID`:** a version stamp stored in the stream. If you don't declare it, the JVM computes one from class structure, so a harmless change (added method) can cause `InvalidClassException` after deploying. Declare `private static final long serialVersionUID = 1L;` and bump it only for *incompatible* changes. Adding fields is compatible (new field gets default).

```java
class Account implements Serializable {
    private static final long serialVersionUID = 1L;
    private String id;
    private transient String password;        // never written
    private transient Instant cachedAt;

    private void writeObject(ObjectOutputStream out) throws IOException {
        out.defaultWriteObject();               // normal fields
        out.writeObject(encrypt(password));     // custom extra
    }
    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
        in.defaultReadObject();
        password = decrypt((String) in.readObject());
    }
    private Object readResolve() { return this; }   // e.g. enforce singleton
}
```
Related hooks: `writeReplace` / `readResolve` (substitute objects; essential for singletons), **serialization proxy pattern** (Effective Java) to protect invariants.

**`Externalizable`:** you implement `writeExternal` / `readExternal` and write **everything yourself**; needs a `public` no-arg constructor, which **is** called. Smaller/faster output possible, more work and more risk.

**Special cases:** `enum`s are serialized by name (safe singletons). **Records** are serialized by their components and reconstructed through the canonical constructor (so validation runs); custom `writeObject` is ignored for them.

**Pitfalls / follow-up probes:**
- Serialization **exposes private state** and creates a hidden public API → every field becomes a compatibility burden.
- `transient` + `final` fields end up default after deserialization.
- A singleton without `readResolve` gets a second instance.
- Prefer not to use Java serialization for new designs (Effective Java: "prefer alternatives"); see the next question (deserialization security and alternatives).

**Remember:** *No constructor runs, `transient` is skipped, `serialVersionUID` guards compatibility.*

---

### Q214. What are the security risks of Java deserialization, and what are the alternatives?

**Say this first:** "Deserializing untrusted bytes is remote code execution waiting to happen — filter it or avoid it."

**Answer:**

**The attack (deserialization gadget chains):**
1. `ObjectInputStream.readObject()` instantiates **any** serializable class on the classpath named in the stream, and runs its `readObject`/`readResolve`/`finalize`-like logic, *before* you can check the type you cast to.
2. An attacker builds a graph of harmless-looking library classes (a "gadget chain", e.g. in Commons Collections) whose combined `readObject` calls end in `Runtime.exec` or a similar sink.
3. Result: arbitrary code execution, or DoS with a deeply nested graph/huge arrays.

**Defences:**
- **Best:** don't deserialize untrusted data with Java serialization.
- **`ObjectInputFilter` (Java 9, JEP 290):** allow-list classes and cap depth/array size/references/bytes.
```java
ObjectInputFilter filter = ObjectInputFilter.Config.createFilter(
    "com.acme.dto.*;java.base/*;maxdepth=10;maxarray=10000;!*");   // allow listed, reject rest
try (ObjectInputStream in = new ObjectInputStream(src)) {
    in.setObjectInputFilter(filter);
    Object o = in.readObject();
}
```
- Process-wide: `-Djdk.serialFilter=...` or `ObjectInputFilter.Config.setSerialFilter`; **Java 17 (JEP 415)** adds filter factories for context-specific filters.
- Validate invariants in `readObject`; use serialization proxy; keep libraries patched.
- Never expose `ObjectInputStream` on a network port or cookie value.

**Alternatives:**

| Format | Strength | Trade-off |
|---|---|---|
| **JSON** (Jackson, Gson) | human-readable, language-neutral | verbose; no schema by default; Jackson *default typing* has its own gadget risks |
| **Protocol Buffers** | compact, fast, schema + field numbers → forward/backward compatible | needs `.proto` and codegen |
| **Avro** | schema stored/registered (Kafka + Schema Registry), good for schema evolution | schema management |
| **Kryo / Hessian** | fast, Java-centric | version/compat and security care |
| Java serialization | zero code for simple cases | insecure, brittle, slow, Java-only |

**Pitfalls / follow-up probes:**
- Where it still hides: Java EE session replication, JMX/RMI, some caching libraries (Redis with `JdkSerializationRedisSerializer`), message queues.
- "How do you evolve a schema?" → with Protobuf/Avro: only add optional fields, never reuse field numbers.
- Log4Shell (2021) was a **JNDI** lookup issue, a related class of "remote class loading" risk — not serialization per se.

**Remember:** *Never `readObject` untrusted input; allow-list with `ObjectInputFilter`; prefer schema-based formats.*

---

### Q215. Why was `java.util.Date`/`Calendar`/`SimpleDateFormat` replaced, and how do `Instant`, `LocalDateTime`, `ZonedDateTime`, `OffsetDateTime` differ?

**Say this first:** "`java.time` is immutable, thread-safe and clear about *time zone*: pick the type by whether you need an instant, a calendar value or a zone."

**Answer:**

**Problems with the legacy API:**
- `Date` is mutable (unsafe to share) and is really an instant, despite its name; year is offset from 1900, month is **0-based** (`new Date(2024, 0, 1)` is year 3924!).
- `Calendar` is mutable, clumsy, and lenient (`set(MONTH, 13)` silently rolls over).
- `SimpleDateFormat` is **not thread-safe** (shared static instance → corrupted output).
- `java.sql.Date/Timestamp` subclass `Date` confusingly; no pure date/time types; poor time-zone support.

**`java.time` (Java 8, JSR-310):** immutable, thread-safe, fluent, ISO-8601, `DateTimeFormatter` is thread-safe.

| Type | Holds | Use for |
|---|---|---|
| `Instant` | point on the UTC timeline (nanoseconds since epoch) | timestamps, audit fields, durations between events |
| `LocalDate` / `LocalTime` / `LocalDateTime` | date/time **without zone** | birthdays, opening hours, "9:00 every day" |
| `ZonedDateTime` | local date-time + **ZoneId** (`Asia/Kolkata`) + offset | user-facing scheduling; follows **DST rules** |
| `OffsetDateTime` | local date-time + fixed offset (`+05:30`) | APIs/DB columns (`timestamptz`), JDBC 4.2 preferred |
| `Duration` / `Period` | time-based (hours, seconds) / date-based (years, months, days) | amounts |

```java
Instant now = Instant.now();
ZonedDateTime ist = now.atZone(ZoneId.of("Asia/Kolkata"));
LocalDate d = LocalDate.of(2025, Month.MARCH, 31).plusMonths(1);   // 2025-04-30 (clamped)
long days = ChronoUnit.DAYS.between(d, LocalDate.of(2025, 12, 25));
DateTimeFormatter f = DateTimeFormatter.ofPattern("dd-MMM-uuuu HH:mm", Locale.ENGLISH);
String s = ist.format(f);
LocalDate parsed = LocalDate.parse("2025-04-30");                  // ISO default
Date legacy = Date.from(now);   Instant back = legacy.toInstant(); // interop
```

**Pitfalls / follow-up probes:**
- `LocalDateTime` has **no zone**; it can't be converted to an instant without supplying a zone/offset.
- `yyyy` vs `YYYY` pattern: `YYYY` is *week-based year* → wrong year around New Year. Use `uuuu`/`yyyy`.
- `Duration.between` on `LocalDate` throws (no time part); use `ChronoUnit` or `Period`.
- `equals` on `ZonedDateTime` compares zone too; use `isEqual` / compare `Instant`s to check the same moment.

**Remember:** *Instant = the moment; Local* = the wall-clock; Zoned/Offset = wall-clock plus where.*

---

### Q216. How should you store timestamps, handle DST pitfalls, and inject a `Clock` for testability?

**Answer:**

**Golden rule: store and transmit in UTC (`Instant` / `timestamptz` / epoch millis); convert to a zone only at the edge (UI).** Why: UTC has no daylight-saving jumps and is unambiguous; ordering and arithmetic are correct. Store the user's **zone ID** (`Asia/Kolkata`, not `IST`) separately if you need local recurrence.

**DST pitfalls:**
- **Gap (spring forward):** `02:30` doesn't exist on that day → `ZonedDateTime.of` shifts it forward.
- **Overlap (fall back):** `01:30` happens twice → `withEarlierOffset/withLaterOffset` chooses.
- `plus` semantics differ:
```java
ZoneId ny = ZoneId.of("America/New_York");
ZonedDateTime t = ZonedDateTime.of(2025, 3, 8, 12, 0, 0, 0, ny);   // day before spring forward
t.plusDays(1);     // 2025-03-09 12:00 (same wall clock; only 23 hours elapsed)
t.plusHours(24);   // 2025-03-09 13:00 (exact elapsed time)
```
  `Period` (days) works on the local timeline; `Duration` (hours) works on the instant timeline.
- A "daily 02:30 job" in a DST zone will skip/duplicate once a year → schedule in UTC or handle gaps.
- Abbreviations like `CST` are ambiguous; use region IDs. Don't use fixed `GMT+5:30` when rules matter.
- Time-zone rules change (governments) → keep JDK/tzdata updated.
- DB: MySQL `DATETIME` has no zone; `TIMESTAMP` converts via session zone → set the connection/JVM zone explicitly (`-Duser.timezone=UTC` is a common server setting but don't rely on it; be explicit in code).

**Clock injection (testability):** `Instant.now()` hides a dependency on global state, so tests can't control it. Pass a `Clock`.
```java
class TokenService {
    private final Clock clock;
    TokenService(Clock clock) { this.clock = clock; }       // prod: Clock.systemUTC()
    boolean isExpired(Instant expiry) { return Instant.now(clock).isAfter(expiry); }
}
// test
Clock fixed = Clock.fixed(Instant.parse("2025-01-01T00:00:00Z"), ZoneOffset.UTC);
assertTrue(new TokenService(fixed).isExpired(Instant.parse("2024-12-31T23:59:59Z")));
```
`Clock.offset(...)` shifts time; `Clock.tick*` truncates. Every `now()` has an overload taking a `Clock`.

**Pitfalls / follow-up probes:**
- `System.currentTimeMillis()` is wall-clock (can jump with NTP); for measuring elapsed time use `System.nanoTime()` (monotonic).
- Comparing `LocalDateTime`s from different zones is meaningless.
- Instant precision: Java 9+ gives microsecond/nanosecond depending on OS; databases may truncate (Postgres µs, MySQL configurable) → equality tests after round trip can fail.

**Remember:** *UTC in storage, zone at the edges, `Clock` in constructors.*

---

### Q217. Explain `Charset`, `Locale` and the UTF-8 default (JEP 400, Java 18) — and common encoding bugs.

**Answer:**

**Charset** maps characters ⇄ bytes. Java `String` is UTF-16 internally (compact Latin-1 representation since Java 9, JEP 254); bytes appear only when you do I/O, `getBytes`, `new String(bytes)`.

**Bug pattern:** relying on the *platform default* charset. Before Java 18, default = OS setting (Windows `windows-1252`, Linux `UTF-8` or `POSIX`/ASCII in minimal containers) → the same code printed `é` correctly on a laptop and as `Ã©` on the server.

```java
byte[] b = "₹".getBytes();                                   // ambiguous (pre-18)
byte[] ok = "₹".getBytes(StandardCharsets.UTF_8);           // always explicit
String s = new String(ok, StandardCharsets.UTF_8);
```

**JEP 400 (Java 18): UTF-8 by default.** The default charset used by `Charset.defaultCharset()`, `FileReader`, `FileWriter`, `InputStreamReader`, `String.getBytes()`, `Scanner`, `PrintStream(file)` etc. is now **UTF-8** on all platforms (`file.encoding` defaults to UTF-8; `-Dfile.encoding=COMPAT` restores old behaviour).
- `native.encoding` system property exposes the OS charset.
- **Not changed:** `System.out`/`System.err` and `Console` use `stdout.encoding`/console's charset (Java 19 added `stdout.encoding`), and `java.nio.file.Files` methods (`readString`, `lines`) were *already* UTF-8.
- Java 17 and below: still set `-Dfile.encoding=UTF-8` or, better, always pass the charset explicitly.

**Other encoding issues:**
- **Malformed input:** `new String(bytes, UTF_8)` silently replaces bad bytes with `\uFFFD`; use `CharsetDecoder` with `REPORT` to fail fast.
- **Surrogate pairs:** emoji take 2 `char`s; `length()` counts UTF-16 units. Use `codePointCount`, `codePoints()`.
- **BOM** at the start of UTF-8 files (Excel exports) shows as `\uFEFF` in the first header.
- HTTP: set `Content-Type: ...; charset=UTF-8`; JDBC/DB connection encoding (`utf8mb4` in MySQL, not `utf8`).
- Properties files: Java 9+ reads `ResourceBundle` `.properties` as UTF-8 (see the Properties/ResourceBundle question); `Properties.load(InputStream)` still uses **ISO-8859-1** — use `load(Reader)`.

**Locale** controls formatting/parsing of numbers, dates, currency, case rules. Pitfalls:
```java
"TITLE".toLowerCase();                       // in Turkish locale -> "tıtle" (dotless i) !
"TITLE".toLowerCase(Locale.ROOT);            // locale-neutral, use for identifiers/keys
String.format("%.2f", 1234.5);               // "1234,50" in a German default locale
String.format(Locale.US, "%.2f", 1234.5);    // "1234.50"
```
Use `Locale.ROOT` for protocol/keys, `Locale.forLanguageTag("en-IN")` for users, and `NumberFormat`/`DateTimeFormatter.ofLocalizedDate` for display. JDK 9+ uses CLDR locale data by default, so formats can differ from Java 8 (`COMPAT`).

**Pitfalls / follow-up probes:**
- Reading and writing with different charsets corrupts text *quietly*.
- Base64/hash outputs are ASCII, but hashing text still needs a defined charset.

**Remember:** *Never trust the default: pass UTF-8 explicitly (and know that Java 18+ finally defaults to it).*

---

### Q218. How do `Properties` and `ResourceBundle` work (configuration and i18n)?

**Answer:**

**`Properties`** — a `Hashtable<Object,Object>` of string key/value pairs, loaded from `.properties` files.
```java
Properties p = new Properties();
try (Reader r = Files.newBufferedReader(Path.of("app.properties"), StandardCharsets.UTF_8)) {
    p.load(r);                                     // load(InputStream) assumes ISO-8859-1
}
int timeout = Integer.parseInt(p.getProperty("timeout", "30"));   // default value
// classpath resource
try (InputStream in = Main.class.getResourceAsStream("/app.properties")) { p.load(in); }
```
- Values are always `String`; convert yourself. Because it extends `Hashtable`, `put(k, nonString)` compiles but breaks `getProperty` → use `setProperty`/`stringPropertyNames()`.
- Layered config: defaults → file → env vars → `-D` system properties (`System.getProperty`). Frameworks (Spring Boot) automate this ordering.
- **Secrets** don't belong in committed properties files.

**`ResourceBundle`** — internationalization (i18n): one bundle per language, same keys.
```
messages.properties        greeting=Hello, {0}!
messages_fr.properties     greeting=Bonjour, {0} !
messages_hi_IN.properties  greeting=नमस्ते, {0}!
```
```java
ResourceBundle rb = ResourceBundle.getBundle("messages", Locale.FRANCE);
String text = MessageFormat.format(rb.getString("greeting"), "Asha");
```
**Lookup chain** for `fr_FR`: `messages_fr_FR` → `messages_fr` → default-locale bundles (`messages_<default>`) → base `messages`. Parent bundles supply missing keys. A missing key → `MissingResourceException`. Bundles are cached.

- Java 9+ reads properties-based bundles as **UTF-8** (JEP 226; falls back to ISO-8859-1 if invalid); on Java 8 you needed `\uXXXX` escapes.
- Alternatives: `ListResourceBundle` (Java class holding `Object[][]`, allows non-String values); `MessageFormat` handles placeholders and plurals (`{0,choice,...}`); ICU4J for rich plural rules.

**Pitfalls / follow-up probes:**
- Locale fallback to the **JVM default** locale can surprise you in tests (set `Locale.setDefault`).
- `Properties.store` writes non-ISO characters escaped and does not preserve comments/order.
- Properties is thread-safe (synchronized `Hashtable`) but coarse; for hot-path config, copy into an immutable object at startup.

**Remember:** *`Properties` = string config with defaults; `ResourceBundle` = locale-keyed message lookup with a fallback chain.*

---

### Q219. `java.util.Random` vs `ThreadLocalRandom` vs `SecureRandom` — which one and when?

**Answer:**

| | `Random` | `ThreadLocalRandom` | `SecureRandom` |
|---|---|---|---|
| Algorithm | 48-bit linear congruential generator (predictable) | faster non-crypto generator | cryptographically strong (OS entropy + DRBG) |
| Thread-safety | safe, but threads **contend** on one atomic seed | per-thread instance, no contention | safe, may synchronize |
| Seedable/reproducible | yes (`new Random(42)`) | **no** | not meant to be |
| Use for | simulations, tests needing determinism | parallel/concurrent non-security randomness | tokens, session IDs, passwords, keys, salts, nonces |

```java
int roll = ThreadLocalRandom.current().nextInt(1, 7);       // [1,7)
Random seeded = new Random(42);                             // deterministic in tests

SecureRandom sr = new SecureRandom();
byte[] token = new byte[32];
sr.nextBytes(token);
String id = Base64.getUrlEncoder().withoutPadding().encodeToString(token);
```

**Why `Random` is insecure:** from a few outputs an attacker can recover the 48-bit seed and predict all future values. Never use `Random` or `Math.random()` for security values. (`UUID.randomUUID()` is built on `SecureRandom`, but it is meant as an identifier, not a secret key.)

**Notes:**
- `ThreadLocalRandom.current()` — call it in the thread using it; don't cache it in a field shared between threads.
- `SecureRandom.getInstanceStrong()` may **block** on some systems waiting for entropy; the default constructor/`new SecureRandom()` is normally fine and non-blocking on modern Linux.
- Java 17 (JEP 356) added the `RandomGenerator` interface and `RandomGeneratorFactory` (algorithms like `L64X128MixRandom`, `SplittableRandom`) — good for parallel streams and reproducible splitting.
- `Collections.shuffle(list, rnd)` and `random.ints(...)` streams accept a generator.

**Pitfalls / follow-up probes:**
- `Math.abs(random.nextInt()) % n` — `abs(Integer.MIN_VALUE)` is negative; use `nextInt(n)`.
- Reusing one seeded `Random` in parallel tests breaks determinism.
- Modulo bias when reducing random bytes to a range.

**Remember:** *Security → `SecureRandom`; concurrency → `ThreadLocalRandom`; reproducible → seeded `Random`/`SplittableRandom`.*

---

### Q220. How do you use `Objects` utilities and `Optional` properly?

**Answer:**

**`java.util.Objects`** — null-safe helpers:
| Method | Use |
|---|---|
| `requireNonNull(x, "msg")` | fail fast in constructors/params (throws NPE with message) |
| `requireNonNullElse(x, def)` / `requireNonNullElseGet` | default value (Java 9) |
| `equals(a, b)` | null-safe equality |
| `hash(a, b, c)` | `hashCode` for `hashCode()` implementation (varargs → boxing, fine for most) |
| `toString(o, "n/a")` | null-safe string |
| `isNull`, `nonNull` | method-reference friendly in streams |
| `checkIndex(i, len)` | index validation |

**`Optional<T>`** — a return-type container saying "a value may be absent", forcing the caller to handle it. Not a general null replacement.
```java
Optional<User> find(String id) { return Optional.ofNullable(map.get(id)); }

String city = find("42")
    .map(User::address)                 // null-returning mapper -> empty
    .map(Address::city)
    .filter(c -> !c.isBlank())
    .orElse("Unknown");

find("42").ifPresentOrElse(this::greet, () -> log.warn("no user"));   // Java 9
User u = find("42").orElseThrow(() -> new NotFoundException("42"));    // Java 10: no-arg orElseThrow()
```
Methods: `of` (NPE on null), `ofNullable`, `empty`, `map`, `flatMap`, `filter`, `or` (Java 9, lazy alternative Optional), `stream` (Java 9), `isEmpty` (Java 11).

**Do / don't:**
- ✅ Use as a **return type** for "might not find".
- ❌ Don't use for fields, method parameters, collections elements (use empty collection, never `Optional<List>`), or in DTOs/JSON entities. `Optional` is not `Serializable`.
- ❌ `opt.get()` without check (use `orElseThrow`); `if (opt.isPresent()) opt.get()` is just a verbose null check.
- **`orElse` vs `orElseGet`:** `orElse(expensive())` evaluates the argument **always**; `orElseGet(() -> expensive())` only when empty.
- Don't return `null` from a method returning `Optional`.
- Primitive variants `OptionalInt/Long/Double` avoid boxing.

**Pitfalls / follow-up probes:**
- `Optional.map(f)` where `f` returns `Optional` → `Optional<Optional<X>>`; use `flatMap`.
- Using `==` on Optionals is meaningless (identity-based, "value-based class").
- Performance: an extra object per call; fine outside the hottest loops.

**Remember:** *Optional is for return values only; `orElseGet` for costly defaults; `Objects.requireNonNull` at the door.*

---

### Q221. `java.util.logging` vs SLF4J/Logback: log levels, parameterized logging and MDC.

**Answer:**

**Facade + implementation:** **SLF4J** is a *facade* (API only) — your code depends on it; the actual logging engine (**Logback**, Log4j2, `java.util.logging` via bridge) is chosen by what is on the classpath. This lets libraries log without forcing an engine on the app. **JUL** (`java.util.logging`) is built into the JDK: no dependency, but clumsy configuration, fewer features, weaker performance and different level names. Java 9 added `System.Logger` as a minimal facade for the JDK itself.

| SLF4J level | JUL level | When |
|---|---|---|
| ERROR | SEVERE | operation failed, needs attention |
| WARN | WARNING | unexpected but handled |
| INFO | INFO | business milestones (startup, order placed) |
| DEBUG | FINE/FINER | developer detail, off in prod |
| TRACE | FINEST | very detailed |

```java
private static final Logger log = LoggerFactory.getLogger(OrderService.class);

log.info("Order {} placed by user {}", orderId, userId);      // parameterized: no string concat
log.error("Payment failed for order {}", orderId, ex);        // exception LAST -> full stack trace
if (log.isDebugEnabled()) log.debug("payload={}", expensiveJson());   // guard costly args
log.atDebug().addArgument(() -> expensiveJson()).log("payload={}");   // SLF4J 2 lazy alternative
```
**Why parameterized?** `"a" + b + c` builds the string even when the level is disabled; `{}` formats only if the level is enabled (and avoids many allocations).

**MDC (Mapped Diagnostic Context):** a **thread-local map** of key/values automatically added to every log line from that thread — perfect for **correlation IDs**.
```java
MDC.put("traceId", traceId);
try { handle(request); }                 // all logs in here include %X{traceId}
finally { MDC.clear(); }                 // threads are pooled! always clean
```
Pattern: `%d %-5level [%thread] %X{traceId} %logger{36} - %msg%n`. MDC does **not** propagate to other threads automatically → copy it into async tasks (`MDC.getCopyOfContextMap()`), and note virtual threads/reactive pipelines need explicit propagation.

**Best practices:** log at the boundary once (don't log-and-rethrow → duplicates); never log secrets/PII/full card numbers; structured (JSON) logging for aggregators (ELK); use async appenders for throughput; keep one `static final` logger per class.

**Pitfalls / follow-up probes:**
- Two bindings on the classpath → SLF4J warns "multiple bindings"; also avoid both `log4j-over-slf4j` and `slf4j-log4j` (loop).
- `log.error("failed: " + e)` loses the stack trace.
- Log4Shell (CVE-2021-44228) was a Log4j2 JNDI-lookup flaw: keep logging libs updated.

**Remember:** *Code against SLF4J, use `{}` placeholders, exception last, MDC for correlation IDs.*

---

### Q222. How does Java regex work? Greedy vs lazy vs possessive, and what is catastrophic backtracking (ReDoS)?

**Answer:**

**API:** `Pattern` (compiled, immutable, **thread-safe**) + `Matcher` (stateful, **not** thread-safe).
```java
private static final Pattern EMAIL = Pattern.compile("^[\\w.+-]+@[\\w-]+(\\.[\\w-]+)+$");  // compile once
Matcher m = Pattern.compile("(?<year>\\d{4})-(?<mon>\\d{2})").matcher("on 2025-03-14");
if (m.find()) { m.group("year"); m.start(); }    // matches() = whole string, find() = next substring, lookingAt() = prefix
"a1b22".replaceAll("\\d+", "#");                  // a#b#
```
`String.matches/split/replaceAll` recompile the pattern each time → in hot paths use a precompiled `Pattern`. Features: named groups, backreferences (`\\1`), lookahead/lookbehind (`(?=…)`, `(?<=…)`), flags (`CASE_INSENSITIVE`, `MULTILINE`, `DOTALL`), `Pattern.quote` for literals.

**Quantifier modes** — on input `<a><b>`:
| Mode | Syntax | Behaviour | `<.+>` style result |
|---|---|---|---|
| **Greedy** (default) | `.+` | take as much as possible, back off if needed | `<.+>` → `<a><b>` |
| **Lazy / reluctant** | `.+?` | take as little as possible, expand if needed | `<.+?>` → `<a>` |
| **Possessive** | `.++` | take as much as possible, **never give back** (no backtracking) | fast fail |

**How matching works:** Java's engine is a **backtracking** NFA. At each choice it tries one path; if later parts fail it **backs up** and tries another.

**ReDoS (Regular-expression Denial of Service):** nested/overlapping quantifiers cause exponential backtracking on failing input.
```java
Pattern.compile("^(a+)+$").matcher("aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa!").matches();
// each extra 'a' roughly doubles the time -> CPU pinned for minutes
```
Dangerous shapes: `(a+)+`, `(a*)*`, `(a|a)+`, `(.*a){n}`.

**Defences:**
- Avoid nested quantifiers and ambiguous alternations; make branches mutually exclusive.
- Use **possessive quantifiers / atomic groups** `(?>...)` where backtracking is pointless.
- **Limit input length** before matching; anchor patterns.
- Wrap the input in a `CharSequence` that checks a deadline (or interrupt flag) in `charAt` to enforce a timeout — Java's regex has no built-in timeout.
- Don't accept user-supplied regexes; for simple jobs use `String` methods, `indexOf`, or a parser.
- Test risky patterns with fuzzing / ReDoS scanners and long failing inputs.

**Pitfalls / follow-up probes:**
- Backslashes in strings: `"\\d"` (Java text blocks help).
- `split(".")` splits on any char (returns empty) — use `split("\\.")`.
- `Matcher.reset`/`appendReplacement` with `$` in replacement → use `Matcher.quoteReplacement`.
- Email/HTML parsing with regex is a trap — use a real parser or validation library.

**Remember:** *Compile once, prefer lazy/possessive, never nest quantifiers on untrusted input.*

---

### Q223. Networking basics: `Socket`/`ServerSocket`, `HttpURLConnection` vs `java.net.http.HttpClient`.

**Answer:**

**TCP basics (`java.net`):**
- `ServerSocket(port)` listens; `accept()` **blocks** until a client connects and returns a `Socket`. Each `Socket` gives `getInputStream()/getOutputStream()`.
- Classic server = **thread per connection** (or a pool) — simple, but limited by threads (see the NIO/Selector question; virtual threads in Java 21 make this model scalable again).
```java
try (ServerSocket ss = new ServerSocket(9000)) {
    while (true) {
        Socket s = ss.accept();
        pool.submit(() -> {
            try (s;
                 var in = new BufferedReader(new InputStreamReader(s.getInputStream(), UTF_8));
                 var out = new PrintWriter(new OutputStreamWriter(s.getOutputStream(), UTF_8), true)) {
                out.println("echo: " + in.readLine());
            } catch (IOException e) { /* log */ }
        });
    }
}
```
- **Always set timeouts:** `socket.connect(addr, connectTimeoutMs)` and `setSoTimeout(readTimeoutMs)`; otherwise a dead peer hangs a thread forever.
- Key terms: TCP is a byte **stream** — message boundaries are your job (length prefix or delimiter); `setTcpNoDelay` (Nagle), `SO_REUSEADDR`, `backlog`; UDP uses `DatagramSocket`.

**HTTP clients:**

| | `HttpURLConnection` (JDK 1.1) | `java.net.http.HttpClient` (standard since Java 11) |
|---|---|---|
| API | awkward, mutable, verbose | fluent builder, immutable client |
| HTTP/2 | no | yes (auto-negotiation), also WebSocket |
| Async | no | `sendAsync` → `CompletableFuture` |
| Timeouts | `setConnectTimeout/ReadTimeout` | `connectTimeout`, per-request `timeout` |
| Notes | error stream quirks, hard to pool | reuse one client (connection pool, executor) |

```java
HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(3))
    .version(HttpClient.Version.HTTP_2).build();                 // create once, reuse
HttpRequest req = HttpRequest.newBuilder(URI.create("https://api.example.com/items"))
    .timeout(Duration.ofSeconds(5)).header("Accept", "application/json").GET().build();

HttpResponse<String> res = client.send(req, HttpResponse.BodyHandlers.ofString());
if (res.statusCode() == 200) System.out.println(res.body());

client.sendAsync(req, HttpResponse.BodyHandlers.ofString())
      .thenApply(HttpResponse::body).thenAccept(System.out::println);
```

**Pitfalls / follow-up probes:**
- No automatic retries or circuit-breaking: add them yourself (or use a library; Spring/OkHttp/Apache have more features).
- Default redirect policy is `NEVER`; configure `followRedirects`.
- `InputStream.read` on a socket may return fewer bytes than asked; loop until you have a full message (`readNBytes`).
- DNS caching: JVM caches lookups (`networkaddress.cache.ttl`); stale IPs in cloud environments.
- Validate URLs from users (SSRF) — block internal addresses.

**Remember:** *ServerSocket accepts, Socket reads/writes streams; set timeouts; reuse one `HttpClient`.*

---

### Q224. JDBC fundamentals: `Connection`, `PreparedStatement`, `ResultSet`, SQL injection, batching, transactions and pooling.

**Answer:**

**JDBC** is the standard Java API to talk to relational databases; a **driver** (vendor JAR) implements it. JPA/Hibernate/Spring `JdbcTemplate` all sit on top.

**Flow:** get `Connection` → create `PreparedStatement` → bind parameters → execute → iterate `ResultSet` → close everything (try-with-resources).
```java
String sql = "SELECT id, name FROM users WHERE email = ?";
try (Connection c = dataSource.getConnection();
     PreparedStatement ps = c.prepareStatement(sql)) {
    ps.setString(1, email);                       // bound as data, never concatenated
    try (ResultSet rs = ps.executeQuery()) {
        while (rs.next()) { long id = rs.getLong("id"); String n = rs.getString("name"); }
    }
}
```

**SQL injection:** building SQL by string concatenation (`"... WHERE email='" + email + "'"`) lets input like `' OR '1'='1` change the query. **`PreparedStatement`** sends the SQL structure and the values separately, so the values can never become SQL. Caveats: you can't bind table/column names or `ORDER BY` direction → allow-list them; `Statement` is fine only for constant SQL.

**Batching** (cuts network round trips):
```java
c.setAutoCommit(false);
try (PreparedStatement ps = c.prepareStatement("INSERT INTO t(a,b) VALUES (?,?)")) {
    for (Row r : rows) { ps.setInt(1, r.a); ps.setString(2, r.b); ps.addBatch();
        if (++n % 1000 == 0) ps.executeBatch(); }
    ps.executeBatch();
    c.commit();
} catch (SQLException e) { c.rollback(); throw e; }
```
Some drivers need a flag to really rewrite batches (MySQL `rewriteBatchedStatements=true`, PostgreSQL `reWriteBatchedInserts=true`) — check the driver docs.

**Transactions:** by default **auto-commit** is on (each statement commits). For a unit of work: `setAutoCommit(false)` → statements → `commit()` or `rollback()`. Isolation via `setTransactionIsolation(...)`. `@Transactional` in Spring does exactly this around your method.

**Connection pooling:** opening a connection (TCP + auth + session setup) is expensive. A **pool** (HikariCP is the common default) keeps N open connections, lends one out through `DataSource.getConnection()`, and `close()` merely **returns it to the pool**. Key settings: `maximumPoolSize` (small! often ~10–20; more is not faster, DB cores are the limit), `connectionTimeout`, `maxLifetime` (shorter than DB/firewall idle timeout), `idleTimeout`, leak detection.

**Pitfalls / follow-up probes:**
- **Connection leak** (not closing) → pool exhaustion → request timeouts. try-with-resources prevents it.
- `ResultSet` default fetches may load the entire result into memory (some drivers); use `setFetchSize` (PostgreSQL also needs autocommit off) for streaming big results.
- `getInt` on `NULL` returns 0 → check `rs.wasNull()` or `getObject(..., Integer.class)`.
- Use `java.time` with JDBC 4.2: `ps.setObject(1, offsetDateTime)`.
- N+1 queries: many small queries in a loop — use a join or batch `IN`.

**Remember:** *`PreparedStatement` for safety, batch + one transaction for speed, pool connections and always close.*

---

### Q225. What is JPMS (Java modules, `module-info.java`)? Why does strong encapsulation make `--add-opens` appear?

**Answer:**

**Problem before Java 9:** the classpath is a flat bag of JARs; `public` meant public to **everyone** (including JDK internals like `sun.misc.Unsafe`), no dependency declaration, split packages, "JAR hell".

**JPMS (Java Platform Module System, Java 9, Project Jigsaw)** adds a layer above packages: a **module** declares what it needs and what it exposes.
```java
module com.acme.orders {
    requires java.sql;                       // dependency
    requires transitive com.acme.common;     // dependents also get this
    exports com.acme.orders.api;             // public API: public types accessible at compile and run time
    exports com.acme.orders.spi to com.acme.audit;   // qualified export
    opens com.acme.orders.model;             // allow DEEP REFLECTION at runtime (private members)
    uses com.acme.orders.spi.Notifier;
    provides com.acme.orders.spi.Notifier with com.acme.orders.EmailNotifier;   // ServiceLoader
}
```

| Keyword | Compile-time access | Runtime reflection on non-public members |
|---|---|---|
| `exports pkg` | public types | no |
| `opens pkg` | no | **yes** |
| `open module m {}` | — | opens everything |

**Benefits:** **strong encapsulation** (non-exported packages are truly hidden), reliable configuration (missing modules found at startup), smaller runtime images via `jlink`, a modularized JDK (`java --list-modules`, `java.base`).

**Why `--add-opens` appears:** frameworks (Hibernate, Jackson, Mockito, Spring, serialization libs) use reflection `setAccessible(true)` on JDK internals or on your classes in named modules. Java 9–15 allowed it with warnings; **Java 16 denied by default, Java 17 removed `--illegal-access`** (JEP 396/403). Now you get `InaccessibleObjectException: module java.base does not "opens java.lang" to unnamed module`. Fix:
```
--add-opens java.base/java.lang=ALL-UNNAMED     # JVM flag
--add-exports java.base/sun.nio.ch=ALL-UNNAMED  # compile/run access to non-exported API (no deep reflection)
```
Better: upgrade the library to a version that doesn't need it. For your own modules use `opens` in `module-info`.

**Unnamed & automatic modules:** code on the classpath lives in the **unnamed module** (can read everything; exports everything); a plain JAR on the *module path* becomes an **automatic module** (name from `Automatic-Module-Name` or file name). That is how most apps (including Spring Boot fat JARs) still run on the classpath and ignore JPMS.

**Pitfalls / follow-up probes:**
- Split packages (same package in two modules) are forbidden.
- Cyclic dependencies between modules are not allowed.
- Reality check: application teams rarely write `module-info`; knowing JPMS mostly helps with upgrade errors and `jlink`.
- `ServiceLoader` (`META-INF/services` or `provides ... with`) is the plug-in mechanism.

**Remember:** *`exports` = compile-time API; `opens` = runtime reflection; `--add-opens` is the escape hatch for old libraries.*

---

### Q226. JNI vs the Foreign Function & Memory (FFM) API: how does Java call native code?

**Answer:**

**Why call native code?** Reuse C/C++ libraries (OpenSSL, BLAS, OS APIs), hardware access, or performance-critical routines; also manage off-heap memory outside the GC.

**JNI (Java Native Interface, since Java 1.1):**
1. Declare `public native int compute(int x);` in Java.
2. Generate header (`javac -h`), write C/C++ using `JNIEnv*`, compile into `.so/.dll`.
3. `System.loadLibrary("name")` loads it; JVM binds the call.
```java
class Native { static { System.loadLibrary("fast"); } native int compute(int x); }
```
Problems: lots of C boilerplate, platform-specific builds, easy to crash the **whole JVM** (segfault, bad pointers), manual reference management, hard to test, call overhead, and no safety net.

**FFM API (Project Panama):** pure-Java, safer replacement.
- **Foreign memory:** `Arena` (scoped lifetime) + `MemorySegment` (bounds-checked off-heap memory) — safe alternative to `ByteBuffer.allocateDirect` and `Unsafe`, with deterministic deallocation and 64-bit sizes.
- **Foreign functions:** `Linker` + `FunctionDescriptor` + `MethodHandle` call C functions directly, no C glue code. The `jextract` tool generates Java bindings from C headers.
```java
Linker linker = Linker.nativeLinker();
MethodHandle strlen = linker.downcallHandle(
    linker.defaultLookup().find("strlen").orElseThrow(),
    FunctionDescriptor.of(ValueLayout.JAVA_LONG, ValueLayout.ADDRESS));
try (Arena arena = Arena.ofConfined()) {
    MemorySegment s = arena.allocateUtf8String("hello");   // method names changed across versions
    long n = (long) strlen.invokeExact(s);                  // 5
}
```
**Status:** the API evolved through incubator/preview releases (Java 14–21) and was finalized in **Java 22 (JEP 454)**; method names changed between previews (e.g. `allocateUtf8String` → `allocateFrom`). *Check the JDK docs for your version* before using it. Access still needs `--enable-native-access` (warnings become errors in future versions).

| | JNI | FFM |
|---|---|---|
| C glue code | required | none |
| Memory safety | none | bounds + lifetime checked |
| Memory model | `GetByteArrayElements`, pinning | `MemorySegment` off-heap |
| Maintenance | hard | pure Java + jextract |

**Pitfalls / follow-up probes:**
- Native calls make the JVM unable to give stack traces/GC help inside that code; native memory leaks do not show in the heap — use **NMT** (`-XX:NativeMemoryTracking`) to diagnose.
- Both are "restricted": unsafe usage can crash the JVM.
- Related: `sun.misc.Unsafe` — internal, being phased out; `VarHandle` and FFM replace its uses.
- JNA (library) is a convenience layer over JNI; slower but easy.

**Remember:** *JNI = C glue and crash risk; FFM = Java-only, bounds-checked access to native memory and functions.*

---

### Q227. Which JDK command-line tools should a senior developer know (`javac`, `java`, `jar`, `javap`, `jshell`, `jlink`, `jpackage`, `jcmd`)?

**Answer:**

| Tool | What it does | Typical use |
|---|---|---|
| `javac` | compiles `.java` → `.class` | `javac -d out --release 17 src/**/*.java`; `-parameters` keeps parameter names for reflection; `-Xlint:all` |
| `java` | launches the JVM | `java -cp out Main`; `java -jar app.jar`; **Java 11+ single-file**: `java Hello.java`; `-Xmx`, `-XX:+HeapDumpOnOutOfMemoryError`, `--add-opens` |
| `jar` | creates/inspects JAR archives | `jar cfe app.jar Main -C out .`; `jar tf app.jar` (list) |
| `javap` | **disassembler** — see class members and bytecode | `javap -c -p Foo` (bytecode incl. private), `-v` constants/stack maps; spot bridge methods, lambdas, string concat (`invokedynamic`) |
| `jshell` | REPL (Java 9) | try an API/regex/`java.time` call quickly without a project |
| `jdeps` | dependency analysis | `jdeps --print-module-deps app.jar` before `jlink` |
| `jlink` | builds a **custom minimal runtime image** with only the modules you need | `jlink --add-modules java.base,java.sql --output rt` (smaller Docker images) |
| `jpackage` | native installers (`.msi/.exe/.dmg/.deb`) bundling a runtime (final in Java 16) | desktop/CLI distribution |
| `jcmd` | **diagnostics Swiss-army knife** for a running JVM by PID | `jcmd <pid> Thread.print` (thread dump), `GC.heap_dump file.hprof`, `GC.class_histogram`, `VM.flags`, `VM.native_memory`, `JFR.start` |
| `jstack`/`jmap`/`jstat` | older single-purpose tools (thread dump / heap / GC stats) | `jcmd` covers them |
| `jfr` + JDK Flight Recorder | low-overhead profiling events | `jcmd <pid> JFR.start duration=60s filename=rec.jfr`, open in JDK Mission Control |

**Example workflows:**
```
# 1. "Why is my service stuck?"
jcmd                       # list JVMs with PIDs
jcmd 4242 Thread.print     # look for BLOCKED / waiting threads, deadlock report

# 2. "Is this lambda/bridge really generated?"
javap -c -p -cp out com.acme.Foo

# 3. "Ship a small runtime"
jdeps --print-module-deps --ignore-missing-deps app.jar
jlink --add-modules <list> --strip-debug --no-header-files --output runtime
```

**Pitfalls / follow-up probes:**
- Use `--release N` (not just `-source/-target`) so the compiler also checks against that JDK's API.
- `jcmd` heap dumps pause the app and are large; prefer `-XX:+HeapDumpOnOutOfMemoryError` for post-mortems.
- Tools must run as the **same OS user** as the target JVM; in containers use `jcmd 1 ...` or `kubectl exec`.
- `jlink` needs modular JDK; automatic-module JARs may need `jdeps`/`--add-modules ALL-MODULE-PATH`.
- `jps` lists Java processes; `jinfo` shows/changes flags.

**Remember:** *`javac`/`java`/`jar` to build and run, `javap` to peek inside, `jshell` to experiment, `jcmd` to diagnose, `jlink`/`jpackage` to ship.*

---

# 9. Unit Testing in Java (JUnit 5 & Mockito)

### Q228. What is the testing pyramid, and how do unit, integration, contract and end-to-end tests differ?

**Say this first:** "I want most of my confidence from fast, cheap tests and a thin layer of slow, realistic ones. The pyramid is about *cost and feedback speed*, not a religion."

**Answer:**
A test is only useful if it is **fast enough to run on every change** and **trustworthy enough that a failure means a real bug**. Those two goals pull in opposite directions, so we mix test types.

| Level | What it exercises | Speed | Typical tools (plain Java) | Main risk it catches |
|---|---|---|---|---|
| **Unit** | One class/method, collaborators faked | ms | JUnit 5, Mockito, AssertJ | Logic bugs, edge cases |
| **Integration** | Your code + a *real* dependency (DB, file system, HTTP server, queue) | 100 ms – seconds | JUnit 5 + Testcontainers, embedded HTTP server (e.g. WireMock) | Wrong SQL, wrong serialization, config/wiring errors |
| **Contract** | The *agreement* (request/response shape) between a consumer and a provider | seconds | Pact (consumer-driven), JSON-schema checks | Service A and B drifting apart without noticing |
| **End-to-end (E2E)** | The whole deployed system through its public entry point | seconds – minutes | Test harness hitting a real environment | Config, deployment, cross-service flow issues |

```
        /\        few  - E2E (slow, flaky, expensive)
       /--\       some - integration / contract
      /----\      many - unit (fast, precise, cheap)
```

**Why the shape matters:**
- A failing unit test points at *one class*. A failing E2E test says "something, somewhere, broke" and costs an hour to diagnose.
- E2E suites grow flaky (network, timing, shared data), people stop trusting them → the "ice-cream cone" anti-pattern (many E2E, few unit tests).
- Some teams prefer a **"testing trophy"** (more integration tests, because real-DB tests catch more real bugs than mock-heavy unit tests). The principle is the same: *choose the cheapest test that can actually catch the bug*.

**Contract tests in one sentence:** the consumer writes down "I call `GET /orders/1` and I need fields `id` and `status`"; the provider's build replays that and fails if it breaks the promise. This replaces a slow shared E2E environment for checking compatibility.

**Pitfalls / follow-up probes:**
- "Is 100% unit coverage enough?" No - units can all pass while the SQL or JSON mapping is wrong; you need some integration tests.
- "What is a *smoke test*?" A tiny post-deploy check ("app starts, health endpoint OK, one critical path works").
- "Where do you test business rules?" In unit tests, on plain objects with no I/O - that is exactly what makes them fast.

**Remember:** *Many fast unit tests, some realistic integration/contract tests, very few E2E - pick the cheapest test that can catch the bug.*

---

### Q229. Explain JUnit 5 architecture and lifecycle. How does it differ from JUnit 4?

**Say this first:** "JUnit 5 is three things: a **Platform** that launches tests, **Jupiter** that is the new programming model, and **Vintage** that runs old JUnit 3/4 tests."

**Answer:**
```
JUnit 5 = JUnit Platform  +  JUnit Jupiter  +  JUnit Vintage
          (launcher, TestEngine API; IDE/build integration)
                            (new annotations + extension model)
                                              (engine that runs JUnit 3/4 tests)
```
- **Platform** - defines the `TestEngine` API and the launcher that IDEs, Maven and Gradle call. Other frameworks (jqwik, Cucumber, Spock) plug in as engines.
- **Jupiter** - the `org.junit.jupiter.*` API you write tests with, plus its own engine.
- **Vintage** - lets you keep running old JUnit 4 tests in the same build while you migrate.

**Lifecycle (one new test-class instance per test method by default):**
```java
class AccountTest {
    @BeforeAll  static void startOnce()   { /* expensive shared setup */ }
    @BeforeEach void freshState()         { /* runs before EVERY test */ }
    @Test       void deposit() { ... }
    @Test       void withdraw() { ... }
    @AfterEach  void cleanUp()            { }
    @AfterAll   static void stopOnce()    { }
}
```
Because each test gets a **new instance**, instance fields are not shared between tests - this prevents hidden test-order dependencies. `@BeforeAll`/`@AfterAll` must be `static` unless you use `@TestInstance(Lifecycle.PER_CLASS)`.

**JUnit 4 vs 5 - the differences interviewers want:**

| Topic | JUnit 4 | JUnit 5 |
|---|---|---|
| Packages | `org.junit` | `org.junit.jupiter.api` |
| Setup | `@Before`, `@BeforeClass` | `@BeforeEach`, `@BeforeAll` |
| Skip | `@Ignore` | `@Disabled` |
| Expect exception | `@Test(expected=...)` or `@Rule ExpectedException` | `assertThrows(...)` |
| Extending | `@RunWith`, `@Rule` (two competing mechanisms) | One **Extension** model (`@ExtendWith`) |
| Grouping | `@Category` | `@Tag` |
| Visibility | test methods must be `public` | package-private is fine |
| Java version | Java 5+ | Java 8+ (lambdas in assertions) |
| Architecture | one monolithic jar | modular (Platform/Jupiter/Vintage) |
| Extra features | - | `@Nested`, `@ParameterizedTest`, `@RepeatedTest`, `@TestFactory`, `@TempDir`, `@Timeout`, `@DisplayName` |

**Pitfalls / follow-up probes:**
- Mixing `org.junit.Test` (4) with Jupiter annotations: the test silently does not run - a classic migration bug. Check imports.
- `@BeforeAll` non-static without `PER_CLASS` → `JUnitException` at runtime.
- Test order is deterministic but intentionally not obvious; never depend on it (use `@TestMethodOrder` only for genuine reasons).

**Remember:** *Platform launches, Jupiter writes, Vintage keeps legacy alive - and a fresh instance per test keeps tests independent.*

---

### Q230. Which JUnit 5 features do you use day to day - parameterized tests, `@Nested`, `assertAll`, `assertThrows`, extensions?

**Say this first:** "They remove duplication and make failures readable: one test method, many inputs; one group per scenario; all assertion failures reported at once."

**Answer:**

**1. `@ParameterizedTest`** (needs the `junit-jupiter-params` artifact) - run one test with many inputs.

| Source annotation | Supplies |
|---|---|
| `@ValueSource(ints = {1, 2, 3})` | A single literal argument per run |
| `@NullSource`, `@EmptySource`, `@NullAndEmptySource` | `null` / empty string-collection-array - great for edge cases |
| `@EnumSource(Status.class)` | Enum constants (can include/exclude) |
| `@CsvSource({"1,2,3", "5,5,10"})` / `@CsvFileSource` | Several columns per run |
| `@MethodSource("cases")` | A static method returning `Stream<Arguments>` - any objects |
| `@ArgumentsSource(MyProvider.class)` | Fully custom provider |

```java
@ParameterizedTest(name = "{0} + {1} = {2}")
@CsvSource({"1,2,3", "0,0,0", "-1,1,0"})
void adds(int a, int b, int expected) {
    assertEquals(expected, Calculator.add(a, b));
}

static Stream<Arguments> invalidEmails() {
    return Stream.of(Arguments.of("a@"), Arguments.of("@b.com"), Arguments.of(""));
}
@ParameterizedTest @MethodSource("invalidEmails")
void rejectsInvalidEmail(String email) { assertFalse(Email.isValid(email)); }
```

**2. `@Nested`** - group tests by scenario inside a **non-static inner class**; each inner class can have its own `@BeforeEach`.
```java
class StackTest {
    Stack<Integer> stack;
    @Nested class WhenEmpty {
        @BeforeEach void init() { stack = new Stack<>(); }
        @Test void popThrows() { assertThrows(EmptyStackException.class, stack::pop); }
    }
    @Nested class WhenHasOneItem { /* push(1) in @BeforeEach, then test pop/peek */ }
}
```

**3. `assertAll` and `assertThrows`**
```java
assertAll("customer",                       // every assertion runs; ALL failures reported together
    () -> assertEquals("Ann", c.name()),
    () -> assertEquals(30, c.age()));

IllegalArgumentException ex =
    assertThrows(IllegalArgumentException.class, () -> new Age(-1));  // returns the exception
assertEquals("age must be >= 0", ex.getMessage());
```
`assertThrows` accepts **subclasses**; `assertThrowsExactly` demands the exact type.

**4. Extensions** replace JUnit 4 runners and rules. Register with `@ExtendWith(MyExt.class)` (declarative), `@RegisterExtension` (programmatic) or auto-detection. You implement callback interfaces such as `BeforeEachCallback`, `AfterEachCallback`, `ParameterResolver` (inject custom parameters into test methods), `TestExecutionExceptionHandler`, `ExecutionCondition` (enable/disable tests). Mockito's `MockitoExtension` and Testcontainers' `@Testcontainers` are built this way. Also useful: `@TempDir` (auto-deleted temp folder), `@Timeout`, `@RepeatedTest(5)`, `@TestFactory` (dynamic tests), `@Tag("slow")`.

**Pitfalls / follow-up probes:**
- A `@MethodSource` factory must be `static` (unless `PER_CLASS`).
- Do not put logic (`if`, loops) in a parameterized test to cater for special rows - split into two tests.
- Interview twist: "Why is `assertAll` better than five separate asserts?" - the first failing plain assert stops the test and hides the rest.

**Remember:** *Parameterize inputs, nest scenarios, assertAll for complete failure reports, extensions instead of runners.*

---

### Q231. Mockito: mock vs spy vs stub - how do you stub, verify and capture arguments?

**Say this first:** "A mock replaces a collaborator with a programmable fake; I stub *what it returns* and verify *only the interactions that matter*."

**Answer:**
- **Mock** - every method returns a default (`null`, `0`, `false`, empty collection/`Optional`) until you stub it.
- **Stub** - a canned answer you configure on a mock (`when(...).thenReturn(...)`). "Stub" is the *role*, "mock" is the Mockito *object*.
- **Spy** - wraps a **real** object; un-stubbed methods run the real code. Use sparingly (legacy code you cannot refactor).

```java
@ExtendWith(MockitoExtension.class)                 // initialises mocks + strict stubs
class CheckoutTest {
    @Mock PriceService prices;
    @Mock OrderRepository repo;
    @Captor ArgumentCaptor<Order> orderCaptor;
    @InjectMocks Checkout checkout;                 // constructor injection of the mocks above

    @Test void savesOrderWithTotal() {
        when(prices.priceOf("A")).thenReturn(new BigDecimal("10"));   // stub
        checkout.place("A", 2);
        verify(repo).save(orderCaptor.capture());                      // verify + capture
        assertThat(orderCaptor.getValue().total()).isEqualByComparingTo("20");
        verify(repo, never()).delete(any());
    }
}
```

**Useful API:**
- `thenThrow(...)`, `thenAnswer(inv -> ...)`, consecutive calls `thenReturn(a, b)`.
- `verify(m, times(2))`, `atLeastOnce()`, `never()`, `verifyNoMoreInteractions(m)`, `InOrder`.
- Void method stubbing: `doThrow(new IOException()).when(m).close();`
- **Spy gotcha:** `when(spy.m())` calls the *real* `m()` while stubbing. Use `doReturn(x).when(spy).m()`.
- **Matchers rule:** if one argument uses a matcher (`any()`, `eq()`), **all** arguments must.
  `verify(repo).find(eq("A"), anyInt())` - not `find("A", anyInt())`.
- **Strict stubs** (default with `MockitoExtension`): an unused stub fails the test with `UnnecessaryStubbingException`; this keeps tests honest. Use `lenient()` only for shared setup.
- **BDDMockito** reads like Given-When-Then: `given(prices.priceOf("A")).willReturn(ten);` ... `then(repo).should().save(any());`

**Pitfalls / follow-up probes:**
- Mockito cannot mock `equals`/`hashCode`, cannot stub constructors (use `mockConstruction`), and cannot see `private` methods.
- Over-verifying every call couples tests to implementation: refactoring breaks them without any behaviour change. Prefer asserting the **result/state**; verify only side effects that *are* the behaviour (e.g. "email sent").
- `@InjectMocks` silently leaves a field null if no mock matches - prefer plain `new Checkout(prices, repo)` in the test.

**Remember:** *Stub inputs, verify important outputs, capture what was passed, and let strict stubs catch dead setup.*

---

### Q232. What should you NOT mock, and how do you handle static and final code?

**Say this first:** "I mock **boundaries I own** (ports to DB, HTTP, clock), not value objects, not the class under test, and not types I do not own."

**Answer:**

**Do NOT mock:**
1. **Value objects / DTOs / records** - construct the real thing (`new Money("10", "USD")`). Mocking them only tests Mockito.
2. **The class under test** (or a partial spy of it) - you are no longer testing real code.
3. **Simple collections, `String`, `Optional`** - use real instances.
4. **Third-party types you don't own** (e.g. `Connection`, `HttpClient`) - wrap them in a small interface you own (an *adapter*), mock that, and test the adapter itself with an integration test.
5. **Things whose real behaviour is the point** - SQL, serialization, regex: a mock will happily "pass" with a wrong query. Use a real DB (Testcontainers).
6. **Too many collaborators** - if a test needs eight mocks, the class has too many responsibilities (design smell).

**Prefer fakes for stateful collaborators:** a hand-written `InMemoryOrderRepository` (a `HashMap` behind the interface) is often clearer than 10 `when(...)` lines and survives refactoring.

**Static and final:**
- Mockito **5** uses the *inline* mock maker by default, so **final classes/methods can be mocked**. On Mockito 2-4 you needed the `mockito-inline` artifact.
- Static methods: `Mockito.mockStatic(...)` - it is **thread-local** and must be closed.
```java
try (MockedStatic<UUID> uuid = Mockito.mockStatic(UUID.class)) {
    uuid.when(UUID::randomUUID).thenReturn(FIXED);
    assertEquals(FIXED, service.newId());
}   // static mock is removed here - forgetting try-with-resources breaks other tests
```
- **But the better answer is to remove the static dependency:** inject a `Supplier<UUID>`, a `Clock`, a `Random`. Statics are hidden inputs; injecting them makes the dependency visible and the test trivial.
- Constructors: `mockConstruction(Foo.class)` exists, but `new Foo()` buried inside a method is itself a testability smell (see the testable-design question).

**Pitfalls / follow-up probes:**
- "Mock vs real for `LocalDateTime.now()`?" Neither - inject `Clock`.
- Mocking `private`/static helper methods to "get coverage" is a sign the helper should be its own class.
- `mockStatic` on a heavily used JDK class (like `System`) can break the test framework itself - avoid.

**Remember:** *Mock the edges you own; use real objects for values; replace statics with injected collaborators.*

---

### Q233. Test doubles vocabulary (dummy, stub, fake, mock, spy) - and AssertJ vs Hamcrest for assertions?

**Say this first:** "Test double is the umbrella term, like a stunt double: anything that stands in for a real collaborator during a test."

**Answer:**

| Double | What it does | Example |
|---|---|---|
| **Dummy** | Passed only to satisfy a signature; never used | `new Checkout(null, dummyLogger)` |
| **Stub** | Returns canned answers; no checks on how it was called | `taxService` always returns `0.18` |
| **Fake** | A *working* but simplified implementation | `InMemoryUserRepository`, an in-memory queue, H2 in place of a prod DB |
| **Mock** | Pre-programmed with expectations; the test **verifies interactions** | "`mailer.send()` was called once" |
| **Spy** | A real object that **records** calls (and can be partly stubbed) | Real `TaxCalculator` where we check `rate()` was called |

Rule of thumb: **state verification** (assert on the result) works with stubs/fakes; **behaviour verification** (assert on calls) needs mocks/spies. Prefer state verification - it survives refactoring.

**AssertJ (fluent, my default):**
```java
assertThat(orders)
    .hasSize(2)
    .extracting(Order::status)
    .containsExactly(NEW, PAID);

assertThat(actualUser).usingRecursiveComparison().isEqualTo(expectedUser);
assertThatThrownBy(() -> service.pay(null))
    .isInstanceOf(IllegalArgumentException.class)
    .hasMessageContaining("order");
assertThat(price).isEqualByComparingTo("10.00");   // BigDecimal ignoring scale
```
Benefits: IDE auto-complete discovers assertions, very readable failure messages, `SoftAssertions` to collect many failures.

**Hamcrest:** matcher-based - `assertThat(list, hasItems("a", "b"))`, `containsString`, `is`, `not`. Matchers are composable and reusable in other libraries (e.g. HTTP-assertion tools, Mockito's `argThat`). Less discoverable in the IDE, messages are fine but chaining is less natural.

**JUnit's own assertions** (`assertEquals(expected, actual)`) are fine for simple cases - remember **expected comes first**, a very common argument-order mistake that produces misleading failure messages.

**Pitfalls / follow-up probes:**
- `assertEquals(1.1, 1.0 + 0.1)` fails on doubles - pass a delta (`assertEquals(a, b, 1e-9)`) or use `BigDecimal`.
- "Is H2 a fake?" Yes - and a risky one: its SQL dialect differs from production; prefer the real DB in a container.
- Don't assert on `toString()` of objects unless the string is the contract.

**Remember:** *Dummy fills, stub answers, fake works, mock verifies, spy records - assert state first, behaviour only when it is the point.*

---

### Q234. How do you design code to be testable (dependency injection, `Clock`, no statics / hidden `new`)?

**Say this first:** "Hard-to-test code is usually hard-to-change code. Testability is a design signal: make every hidden input explicit."

**Answer:**
A method is easy to test when **all its inputs come in through parameters/constructor** and **all its outputs come out through return values or an injected collaborator**. Anything else (current time, random numbers, singletons, `new HttpClient()` inside the method) is a **hidden dependency**.

**Before - untestable:**
```java
class InvoiceService {
    Invoice create(Order o) {
        PriceClient client = new PriceClient("https://prices.internal");   // hidden new
        BigDecimal p = client.fetch(o.sku());                                // real network
        LocalDate due = LocalDate.now().plusDays(30);                        // hidden clock
        return new Invoice(UUID.randomUUID(), p, due);                       // hidden randomness
    }
}
```

**After - plain constructor injection, no framework needed:**
```java
class InvoiceService {
    private final PriceClient prices;
    private final Clock clock;
    private final Supplier<UUID> ids;

    InvoiceService(PriceClient prices, Clock clock, Supplier<UUID> ids) {
        this.prices = prices; this.clock = clock; this.ids = ids;
    }
    Invoice create(Order o) {
        return new Invoice(ids.get(), prices.fetch(o.sku()), LocalDate.now(clock).plusDays(30));
    }
}

// test: deterministic, no network
Clock fixed = Clock.fixed(Instant.parse("2025-01-31T10:00:00Z"), ZoneOffset.UTC);
var svc = new InvoiceService(sku -> BigDecimal.TEN, fixed, () -> KNOWN_UUID);
assertEquals(LocalDate.of(2025, 3, 2), svc.create(order).dueDate());
```

**Guidelines:**
- **Depend on interfaces (ports)** for I/O; keep the business logic in pure functions/objects.
- **Constructor injection** over field injection - dependencies are obvious, fields can be `final`, tests don't need reflection.
- **Inject `Clock`**: `LocalDate.now(clock)`, `Instant.now(clock)`. Use `Clock.fixed` for a frozen time, or a small mutable test clock to "advance" time.
- Avoid **static mutable state and singletons**; if unavoidable, hide behind an instance.
- **Law of Demeter / small classes**: `a.getB().getC().doIt()` forces mocks of mocks.
- **Separate decisions from effects** ("functional core, imperative shell"): compute *what to do* in pure code, then execute it at the edge.
- Prefer **immutable objects** - no setup/teardown state to leak between tests.

**Pitfalls / follow-up probes:**
- Over-engineering: do not create an interface for every class "for mocking". Create one when there is a real boundary (I/O, time, randomness) or multiple implementations.
- A constructor with 7+ parameters means too many responsibilities, not "needs a DI container".
- Interview question: "What is the difference between DI and IoC?" - IoC is the principle (something else controls creation/wiring); DI is the technique of handing dependencies to the object. You can do DI by hand with `new` in `main`.

**Remember:** *Pass time, randomness and I/O in; the code becomes deterministic and tests become boring - which is the goal.*

---

### Q235. What are TDD and BDD, and when do they actually help?

**Say this first:** "TDD is a *design* technique that happens to leave tests behind; BDD is about *communicating behaviour* in business language."

**Answer:**

**TDD - Red → Green → Refactor:**
1. **Red** - write one small failing test that describes the next behaviour.
2. **Green** - write the simplest code that makes it pass (even hard-coded).
3. **Refactor** - clean up with the safety net of green tests.
Repeat in tiny steps (minutes, not hours).

```java
@Test void emptyStringSumsToZero() { assertEquals(0, StringCalc.add("")); }          // red
// implement: if (s.isEmpty()) return 0;                                              // green
@Test void twoNumbersAreAdded()    { assertEquals(3, StringCalc.add("1,2")); }       // red again
```
**Why it helps:** forces you to think about the API from the caller's side, produces small testable units, gives fast feedback, and avoids untested code. It works best for **algorithmic/business logic** with clear rules; it is less natural for exploratory spikes, UI layout, or when you do not yet know what you are building (do a throw-away *spike*, then TDD the real thing).

**BDD - Behaviour-Driven Development:**
- Describe behaviour as **Given / When / Then** scenarios that business, QA and developers all understand:
  *Given* an account with balance 100, *When* the user withdraws 30, *Then* the balance is 70.
- Tools like Cucumber map plain-text scenarios to Java step methods. Without such a tool you can still write BDD-style test names and `BDDMockito` (`given/when/then`).
- Value is mainly in the **conversation** (shared understanding, "three amigos"); if only developers read the scenario files, it is just extra indirection.

| | TDD | BDD |
|---|---|---|
| Audience | Developers | Whole team |
| Level | Unit/design level | Feature/behaviour level |
| Artifact | Tests in code | Readable scenarios + glue code |

**Pitfalls / follow-up probes:**
- "100% TDD always?" No - be pragmatic; use it where it pays (complex logic, bug fixes: *first write a test that reproduces the bug*).
- Chicago/classic style (state-based, real collaborators) vs London/mockist style (interaction-based, mocks everywhere). Mention you lean classic to avoid brittle tests.
- TDD does not replace integration tests or design thinking.

**Remember:** *TDD = red, green, refactor in small steps; BDD = shared Given/When/Then language - both aim at behaviour, not implementation.*

---

### Q236. What does code coverage really tell you? JaCoCo, its limits, and mutation testing with PIT.

**Say this first:** "Coverage tells me what was *executed*, never what was *verified*. Low coverage proves risk; high coverage proves nothing."

**Answer:**
**JaCoCo** instruments bytecode and reports, per class/method:
- **Line / instruction coverage** - was this code run?
- **Branch coverage** - were both outcomes of each `if`/`switch` taken?
- Also cyclomatic complexity, method and class coverage.

**Why 100% coverage can still hide bugs:**
```java
int discount(int qty) { return qty > 10 ? 20 : 0; }

@Test void runsIt() { discount(11); discount(5); }   // 100% line + branch coverage, ZERO assertions!
```
The tests executed every branch but would pass even if the method returned garbage. Coverage also says nothing about missing code (a forgotten requirement), concurrency bugs, or integration problems.

**Mutation testing (PIT / Pitest)** measures test *quality*:
1. PIT makes small changes ("mutants") to your bytecode: flip `>` to `>=`, negate a condition, return `null`, remove a method call.
2. It re-runs the tests for each mutant.
3. If a test fails, the mutant is **killed** (good). If all tests pass, the mutant **survived** → your tests don't notice that behaviour changing.
**Mutation score** = killed / total. For the example above, mutating `>` to `>=` survives, exposing the missing boundary test for `qty == 10`.

```xml
<!-- Maven: org.pitest:pitest-maven plus the JUnit 5 plugin dependency -->
mvn org.pitest:pitest-maven:mutationCoverage
```
(JUnit 5 needs the `pitest-junit5-plugin`.) It is slower than JaCoCo, so run it on changed code or nightly, and on core domain logic rather than getters/DTOs.

**Practical policy I would state in the interview:**
- Use JaCoCo to find **untested areas**, not as a target to game. A team-wide gate like "new code ≥ 80% line and no drop" is reasonable; "100% or the build fails" encourages assertion-free tests.
- Add mutation testing to critical modules (pricing, security, money).
- Exclude generated code, DTOs, `main` bootstrap from the numbers.

**Pitfalls / follow-up probes:**
- Goodhart's law: when a measure becomes a target it stops being a good measure.
- Branch coverage > line coverage; a single line `a ? b : c` can be 100% line but 50% branch.
- Equivalent mutants (mutation changes nothing observable) can never be killed - don't chase 100% mutation score either.

**Remember:** *Coverage shows what ran; mutation score shows whether the tests would notice a bug.*

---

### Q237. How do you test concurrent / asynchronous code and eliminate flaky tests?

**Say this first:** "A flaky test is a bug report with the wrong label: either the test or the code has a hidden dependency on timing, order, or shared state."

**Answer:**

**Why tests become flaky:**
- `Thread.sleep(500)` hoping the work is done (too short on a slow CI, wasted time elsewhere).
- Shared mutable state / static fields / shared DB rows between tests; dependence on test order.
- Real time (`now()`), randomness, timezone, hash-iteration order, fixed ports, real network.
- Genuine race conditions in the code under test.

**Techniques:**
1. **Make the code deterministic by injection** - pass the `Executor` in. In tests use a *direct* executor so "async" work runs on the calling thread:
```java
Executor direct = Runnable::run;
var service = new ReportService(direct, repo);
service.generateAsync(id);
assertTrue(repo.isSaved(id));   // no waiting needed
```
   For scheduled work, inject a `ScheduledExecutorService`/`Clock` you can advance manually.
2. **Wait for a condition, not a duration - Awaitility:**
```java
await().atMost(Duration.ofSeconds(5)).pollInterval(Duration.ofMillis(50))
       .untilAsserted(() -> assertThat(repo.count()).isEqualTo(3));
```
3. **Synchronise with latches** - `CountDownLatch` the code counts down when done; the test calls `latch.await(2, TimeUnit.SECONDS)` (always with a timeout).
4. **Futures:** `future.get(2, TimeUnit.SECONDS)`; for `CompletableFuture` use `orTimeout`/`join` carefully. Add `@Timeout(5)` so a deadlock fails instead of hanging CI.
5. **Stress the race on purpose:** start N threads together with a `CyclicBarrier`/`CountDownLatch`, hammer the code, then assert an invariant (e.g. final counter equals N*iterations). Repeat many times (`@RepeatedTest(100)`). For memory-model-level bugs the OpenJDK **jcstress** tool exists.
6. **Prove thread-safety by design** where possible: immutable objects, `java.util.concurrent` classes, confinement; then the test becomes small.

**Handling flaky tests as a team:** reproduce (run in loop, same seed/timezone), fix the root cause, never "just rerun". If you must, **quarantine** it (`@Tag("flaky")`, separate job) with an owner and a deadline - an ignored-forever flaky test is a deleted test. Log random seeds so failures can be replayed.

**Pitfalls / follow-up probes:**
- A passing concurrency test does not *prove* correctness; absence of failure in 1000 runs is evidence, not proof.
- Assertions inside other threads are swallowed - capture the exception/`Future` and assert on the test thread.
- `parallel` test execution requires tests that share nothing.

**Remember:** *Inject executors, wait for conditions (Awaitility/latches) never for time, and treat flakiness as a real defect.*

---

### Q238. What is Testcontainers, and what is property-based testing?

**Say this first:** "Testcontainers gives me a *real* dependency in a disposable Docker container; property-based testing lets the computer invent the inputs while I state the rules."

**Answer:**

**Testcontainers** is a Java library that starts throw-away Docker containers (PostgreSQL, Kafka, Redis, a custom image) from your test, waits until they are ready, and removes them afterwards.

**Why:** in-memory substitutes (H2 instead of Postgres) differ in SQL dialect, locking, JSON types, and index behaviour - tests pass, production fails. A real DB version in a container removes that gap, and every developer/CI run gets the same clean environment.

```java
@Testcontainers
class UserDaoIT {
    @Container
    static PostgreSQLContainer<?> db = new PostgreSQLContainer<>("postgres:16");   // static = one container per class

    @Test void insertsAndReadsUser() throws Exception {
        try (Connection c = DriverManager.getConnection(db.getJdbcUrl(), db.getUsername(), db.getPassword())) {
            var dao = new UserDao(c);
            dao.insert(new User("ann"));
            assertEquals("ann", dao.findByName("ann").name());
        }
    }
}
```
Tips: reuse containers across a class/suite (starting is the slow part), pin image versions, requires Docker on CI, name your slow tests `*IT` so unit and integration phases can run separately. A resource reaper container cleans up if the JVM dies.

**Property-based testing (jqwik, junit-quickcheck):** instead of hand-picking examples, state a **property** that must hold for *all* valid inputs; the framework generates hundreds of random cases, and on failure **shrinks** the input to the smallest counter-example.
```java
@Property
boolean reversingTwiceGivesOriginal(@ForAll List<Integer> list) {
    List<Integer> copy = new ArrayList<>(list);
    Collections.reverse(copy); Collections.reverse(copy);
    return copy.equals(list);
}
```
Good properties: round-trip (`decode(encode(x)) == x`), idempotence (`sort(sort(x)) == sort(x)`), invariants (result of `merge` is sorted and same size), "never throws for valid input", comparing against a slow-but-obvious reference implementation.

**Pitfalls / follow-up probes:**
- Property tests find edge cases you would never write (empty, huge, negative, unicode), but they need *good properties* - a re-implementation of the code as the "property" proves nothing.
- Keep the failing seed so the case is replayable; promote shrunk failures into normal example tests.
- Testcontainers tests are slower - keep them focused on the integration seam (SQL, serialization), not business logic.

**Remember:** *Testcontainers = real dependencies, disposable; property-based = state the rule, let the machine hunt for counter-examples.*

---

### Q239. How do you test private methods? (and what about exceptions and time?)

**Say this first:** "I don't test private methods directly - they are implementation detail. I test the behaviour through the public API, and if a private method is complex enough to deserve its own test, that is a design hint."

**Answer:**

**Why not test private methods:** tests should pin down **behaviour**, not structure. If you test a private helper, every refactor (rename, inline, split) breaks tests although nothing observable changed. Private code is already covered when you exercise the public methods that call it. If you cannot reach it that way, the code is probably **dead**.

**What to do when a private method feels important:**
1. **Extract it into its own class** with a public (or package-private) method and test that - this is usually the right answer (single responsibility).
2. Make it **package-private** and put the test in the same package (`@VisibleForTesting`-style comment) - an acceptable compromise.
3. Reflection (`setAccessible(true)`) works but is brittle and breaks with module encapsulation (Java 9+ `--add-opens`) - avoid.

**Test naming and structure (AAA / Given-When-Then):**
```java
@Test
@DisplayName("withdraw rejects amount greater than balance")
void withdraw_throwsInsufficientFunds_whenAmountExceedsBalance() {
    // Arrange (Given)
    Account account = new Account(new BigDecimal("50"));
    // Act (When) + Assert (Then)
    InsufficientFundsException ex = assertThrows(InsufficientFundsException.class,
            () -> account.withdraw(new BigDecimal("80")));
    assertEquals(new BigDecimal("30"), ex.shortfall());
}
```
Name = *unit + scenario + expected outcome*. One logical reason to fail per test. Keep tests independent, readable without scrolling, and free of `if`/loops.

**Testing exceptions:** assert the **type**, the important part of the **message/fields**, and that **no side effect happened** (`verify(repo, never()).save(any())`). Use `assertThrows`/`assertThatThrownBy`, not `try/catch` + `fail()` (a missing throw would silently pass if you forget `fail()`). Also test that the original **cause** is preserved (`ex.getCause()`).

**Testing time:**
- Never call `Thread.sleep` or `LocalDateTime.now()` in logic you need to test; inject `Clock` (see the testable-design question).
- Boundary tests: exactly at expiry, one millisecond before/after, month-end, leap day, DST transition, different time zones.
- For timeouts/retries inject a scheduler or a `Sleeper` interface and record the requested delays instead of waiting.
```java
MutableClock clock = new MutableClock(Instant.parse("2025-01-01T00:00:00Z"));
Token t = new TokenService(clock).issue(Duration.ofMinutes(5));
clock.advance(Duration.ofMinutes(5));
assertTrue(t.isExpired());
```

**Pitfalls / follow-up probes:**
- "But I need coverage of that private branch!" - then exercise it through a public input that reaches it; if impossible, delete it.
- Test names that describe implementation (`test_calls_helper`) rot quickly.
- A test with 15 asserts is a script, not a specification - split it.

**Remember:** *Test behaviour through the public door; extract, don't reflect; inject the clock; name tests as scenario + outcome.*

---

# 10. Production Debugging & Scenario-Based Questions

### Q240. The app gets slower and slower over hours or days, then restarts fix it. How do you find a memory leak?

**Situation:** Latency creeps up, GC runs more and more often, eventually `OutOfMemoryError` or a restart. After restart everything is fast again.

**Say this first:** "Gradual slowdown plus 'restart fixes it' is the classic signature of a leak: objects stay *reachable* (something still points to them), so the garbage collector (GC) cannot free them."

**How you investigate:**
1. **Confirm it is a leak, not just a big heap.** Look at *heap used after GC* over time. A healthy app draws a saw-tooth whose *bottom* stays flat. A leak = the bottom keeps climbing.
```bash
jstat -gcutil <pid> 5000          # O (old gen %) keeps rising; FGC count rising, little freed
# Java 9+: enable GC logs at startup
-Xlog:gc*:file=gc.log:time,uptime:filecount=5,filesize=20m
```
2. **Find *what* is growing** - take two class histograms ~10-30 min apart and diff them:
```bash
jcmd <pid> GC.class_histogram > h1.txt      # note: forces a full GC unless you add -all
# ...wait...
jcmd <pid> GC.class_histogram > h2.txt      # which class's instance count/bytes grew a lot?
```
3. **Find *who holds them*** - take a heap dump (see the heap-dump question) and open it in Eclipse MAT: *Leak Suspects* report, then the *Dominator Tree* (biggest retained sizes), then right-click an instance → *Path to GC Roots → exclude weak/soft references*. The chain `static field → HashMap → ... → your object` is the culprit.
4. Optionally record allocations with **JFR** (`jcmd <pid> JFR.start duration=120s filename=leak.jfr`) and look at "old object sample" events.

**Root cause (usual suspects):** unbounded cache/static map, listeners never deregistered, `ThreadLocal` in pooled threads, sessions/requests kept in a collection, a queue whose consumer is slower than the producer, classloader leak (see the leak-sources question).

**Fix:** remove the strong reference (bound the cache with eviction, `remove()` in `finally`, unregister listeners, close resources). Don't "fix" it by raising `-Xmx` - that only delays the crash.

**Prevention:**
- Always run with `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps`.
- Alert on *old-gen occupancy after GC* trending upward, not on raw heap usage.
- Soak/load tests of several hours before release; review every `static` collection and cache for a size bound.

**Pitfalls / follow-up probes:**
- "Java has GC, how can it leak?" - GC frees *unreachable* objects only; a forgotten reference is still reachable, so it's a logical leak.
- A heap dump taken right after a full GC shows only live objects; that is what you want.
- Memory growth that is **not** on the heap (threads, direct buffers, metaspace) won't show in the heap dump - check native memory (`-XX:NativeMemoryTracking=summary`, `jcmd <pid> VM.native_memory summary`).

**Remember:** *Rising floor after GC = leak; diff histograms to find the class, dominator tree + path-to-GC-roots to find the holder.*

---

### Q241. `OutOfMemoryError: Java heap space` and `GC overhead limit exceeded` - how do you diagnose and fix?

**Situation:** Requests start failing, logs show `java.lang.OutOfMemoryError: Java heap space` (or `GC overhead limit exceeded`), the process may die or limp.

**Say this first:** "First decide: is it a **leak** (heap fills slowly, over time), **under-sizing** (the working set really needs more), or **one huge request** (a single call loads millions of rows or a huge file)? The fix is different for each."

**How you investigate:**
1. The dump: if the JVM was started with `-XX:+HeapDumpOnOutOfMemoryError`, you already have an `.hprof` file. If not, you only have the stack trace - the thread that *hit* the error is often just the victim, not the culprit.
2. Open the dump in MAT / VisualVM: which objects retain most memory? One `byte[]`/`ArrayList` of 2 GB → a big request. Millions of small objects under one cache → leak.
3. Read the GC log around the crash: does the heap after full GC stay near 100% (real shortage) or is it one sudden spike (giant allocation)?
4. Check recent changes: a new report endpoint, `SELECT *` without a limit, `readAllBytes()` on an upload, a larger batch size.

**`GC overhead limit exceeded`** = the JVM (Parallel GC) spent more than ~98% of time in GC while recovering less than ~2% of the heap. It's the heap-full problem seen from the "dying slowly" side; the diagnosis is identical. (Flag `-XX:-UseGCOverheadLimit` merely hides the warning - don't.)

**Root cause examples:**
```java
List<Row> all = jdbc.queryAll("select * from events");     // loads the whole table
byte[] data = Files.readAllBytes(path);                     // 4 GB file
Map<String, Session> cache = new HashMap<>();               // never evicted
```
**Fix:**
- **Stream/paginate** instead of loading all: process in chunks, `fetchSize` on JDBC, `Files.lines` in try-with-resources, `InputStream` copy.
- **Bound everything**: max upload size, page size limit, cache with maximum entries.
- If the data really needs it, **right-size the heap** (`-Xmx`) *and* check container limits (see the "works on my machine" question).
- Fail fast on the one bad request rather than killing the whole JVM.

**Prevention:**
- `-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=...` plus `-XX:+ExitOnOutOfMemoryError` (or `CrashOnOutOfMemoryError`) so an orchestrator restarts a *clean* process instead of a half-broken one.
- Load-test with realistic data volumes; reject unbounded queries in code review.

**Pitfalls / follow-up probes:**
- After an OOME the JVM state is unreliable (other threads may have died inside `finally`); restart rather than continue.
- Catching `OutOfMemoryError` and "carrying on" is almost always wrong.
- Heap dumps contain customer data - protect them (see dump-safety question).

**Remember:** *Leak, undersized, or one giant request? Dump + GC log decide it; stream and bound instead of just raising -Xmx.*

---

### Q242. `OutOfMemoryError: Metaspace`, `unable to create native thread`, `Direct buffer memory` - what are these and how do you fix them?

**Situation:** The heap looks fine (plenty free) yet the JVM throws an OOME with a *different* message, or the container is killed.

**Say this first:** "The heap is only one of the JVM's memory areas. These errors come from *non-heap* memory: class metadata, thread stacks, and off-heap buffers."

**1) `OutOfMemoryError: Metaspace`** - Metaspace stores class metadata (native memory, grows until `-XX:MaxMetaspaceSize` if set).
- *Investigate:* `jstat -gc <pid>` (columns MC/MU = metaspace capacity/used) growing steadily; `jcmd <pid> GC.class_histogram`, `jcmd <pid> VM.classloader_stats`; `-Xlog:class+load` to see what keeps loading.
- *Root cause:* **classes generated at runtime without bound** (dynamic proxies, bytecode generation, scripting engines creating a new class per call, reflection accessors) or a **classloader leak** on redeploy (old loader never collected → all its classes stay).
- *Fix:* reuse generated classes/proxies (cache them), fix the leak that pins the old classloader, set a sane `-XX:MaxMetaspaceSize` so a leak fails loudly instead of eating the machine.

**2) `OutOfMemoryError: unable to create native thread`** - the OS refused a new thread.
- *Investigate:*
```bash
ls /proc/<pid>/task | wc -l             # or: ps -o nlwp= -p <pid>
jcmd <pid> Thread.print | grep -c '^"'  # thread count + names reveal the leaking pool
ulimit -u                                # per-user process/thread limit
cat /sys/fs/cgroup/pids.max              # container pid limit (cgroup v2 path)
```
- *Root cause:* a thread **per request/task** (`new Thread`, `Executors.newCachedThreadPool()` under load), pools created in a loop and never shut down, big `-Xss` × thousands of threads, OS/container limit too low.
- *Fix:* use **bounded pools** (create once, reuse, `shutdown()` on stop), lower `-Xss` only if truly safe, raise limits only after fixing the leak. Virtual threads (Java 21+) make thread-per-task cheap but don't remove the need for limits on downstream resources.

**3) `OutOfMemoryError: Direct buffer memory`** - NIO `ByteBuffer.allocateDirect` memory is off-heap, capped by `-XX:MaxDirectMemorySize` (defaults to roughly max heap). It's freed only when the small heap object that owns it is garbage-collected, so a quiet heap can leave direct memory unreleased. *Fix:* pool/reuse buffers, release explicitly in libraries that support it, cap the number of concurrent big I/O operations.

**Finding unexplained total memory:**
```bash
-XX:NativeMemoryTracking=summary
jcmd <pid> VM.native_memory summary     # heap, class, thread, code cache, GC, internal...
```

**Prevention:** monitor thread count, metaspace used, and direct-buffer usage (JMX `BufferPoolMXBean`); alert on steady growth; load-test with redeploy/hot-reload scenarios if you have them.

**Pitfalls / follow-up probes:**
- Container "OOMKilled" (exit code 137) is the **kernel** killing the process for exceeding the container limit - not a Java `OutOfMemoryError`; the cause can be any area above, not only the heap.
- Raising `-Xmx` makes the native-thread case *worse* (less memory left for stacks).

**Remember:** *Heap fine but OOME? Look at class metadata, thread stacks, direct buffers - count classes, threads, buffers.*

---

### Q243. A Java process is at 100% CPU. How do you find the exact code responsible?

**Situation:** `top` shows the Java process at 100%+ CPU (400% on a 4-core box). Latency is terrible. Nothing was deployed... or something was.

**Say this first:** "I go *process → thread → stack line*: find the hottest **OS thread**, convert its id to hex, and match it to the `nid` in a thread dump."

**How you investigate:**
```bash
# 1. Which threads of the process burn CPU? (-H = show threads)
top -H -p <pid>                       # note the PID column of the top thread, e.g. 21480

# 2. Convert that thread id to hex (thread dumps print it as nid=0x....)
printf '%x\n' 21480                   # -> 53e8

# 3. Take a thread dump (a few, 3-5 s apart) and find that thread
jstack <pid> > t1.txt                 # or: jcmd <pid> Thread.print > t1.txt
grep -A 25 'nid=0x53e8' t1.txt
```
Typical output - the top stack frame is your hot spot:
```
"pool-3-thread-7" #42 prio=5 os_prio=0 cpu=913452ms tid=0x00007f.. nid=0x53e8 runnable
   java.lang.Thread.State: RUNNABLE
        at java.util.regex.Pattern$Loop.match(Pattern.java:4817)
        at com.acme.Validator.isValid(Validator.java:31)
```
Take **several** dumps - if the same thread sits on the same frame each time, it's the loop. One dump is a single snapshot and can mislead.

**What the hot thread's name tells you:**
- Your own thread in `RUNNABLE` in your code → infinite/tight loop, busy-wait (`while(!flag){}`), expensive computation, or **catastrophic regex backtracking** (nested quantifiers like `(a+)+b` on long input).
- `GC Thread` / `G1 ...` / `VM Thread` → it's the garbage collector (heap too small or leak) - go to the OOM/GC questions.
- `C2 CompilerThread` → JIT compiling; normal for a few minutes after start/deploy.
- A `HashMap` used by multiple threads in Java 7 can spin forever (see the HashMap question).

**Fix:** correct the loop/exit condition; replace the busy-wait with blocking (`BlockingQueue`, `Condition`, `Future`); rewrite the regex (possessive quantifiers/atomic groups, limit input length, or avoid regex); cache the expensive result; add a back-off to retry loops.

**Prevention:**
- Input length limits; timeouts on all loops that depend on external state.
- Continuous low-overhead profiling (**JFR**: `jcmd <pid> JFR.start duration=60s filename=cpu.jfr settings=profile`, open in JDK Mission Control) - also shows hot methods without hex gymnastics. async-profiler is a popular external alternative.
- Alert on process CPU, and name your threads (custom `ThreadFactory`) so dumps are readable.

**Pitfalls / follow-up probes:**
- Why hex? `top` shows decimal thread ids, `jstack` prints `nid` in hex.
- High CPU + low throughput and many `RUNNABLE` threads in `socketRead` is *not* CPU - those are waiting on I/O.
- Container CPU limit throttling looks like slowness at *low* CPU (see the p99 question).

**Remember:** *top -H → printf '%x' → jstack | grep nid → read the top frame; several dumps, not one.*

---

### Q244. The application hangs - no errors, requests just wait forever. How do you diagnose deadlock vs thread-pool exhaustion?

**Situation:** Health endpoint times out or requests never complete. CPU is **low**. Logs go quiet.

**Say this first:** "Low CPU plus no progress means threads are *waiting*. A thread dump shows exactly what each thread waits for - take three, 5-10 seconds apart."

**How you investigate:**
```bash
jcmd <pid> Thread.print -l > dump1.txt        # -l adds lock info (java.util.concurrent locks)
# or: jstack -l <pid>   |   kill -3 <pid>  (prints to the JVM's stdout/log)
```
Then read it with these three patterns:

**A) Deadlock** - the dump ends with a section the JVM computes for you:
```
Found one Java-level deadlock:
"T-1": waiting to lock monitor 0x..a (object ..., a Account), which is held by "T-2"
"T-2": waiting to lock monitor 0x..b (object ..., a Account), which is held by "T-1"
```
*Cause:* two threads take the same locks in **opposite order** (`transfer(a,b)` vs `transfer(b,a)`). *Fix:* a global lock order (e.g. by account id), or `tryLock(timeout)`, or one coarser lock. Programmatic detection: `ThreadMXBean.findDeadlockedThreads()`.

**B) Thread-pool exhaustion / starvation** - all pool threads are `WAITING` or `TIMED_WAITING`:
```
"worker-1" WAITING  at java.util.concurrent.FutureTask.get
"worker-2" WAITING  at java.util.concurrent.FutureTask.get   (all N workers!)
```
*Cause:* tasks running in a pool **wait for other tasks submitted to the same pool** - everyone holds a thread, nobody is left to run the sub-task. Or all threads block on a slow downstream / a full connection pool (`getConnection` in the stack, `parking to wait for`). *Fix:* separate pools per kind of work (bulkheads), never block a pool thread on a task of the same pool, add **timeouts** (`future.get(2, SECONDS)`, connect/read timeouts), size pools sensibly.

**C) Everyone is blocked on one lock** - many threads `BLOCKED` on the same monitor owned by one thread that is itself stuck in I/O (e.g. synchronized method calling a remote service). *Fix:* shrink the critical section; never do I/O while holding a lock.

**Prevention:**
- Timeouts everywhere (HTTP, JDBC, queue polls, `Future.get`); no unbounded waits.
- Consistent lock ordering documented in code; prefer immutable/concurrent structures over hand-rolled locking.
- A watchdog/alert on "pool active == max for N minutes", and name threads.

**Pitfalls / follow-up probes:**
- A **livelock** (threads busy retrying, CPU high) does not appear as a deadlock section.
- `jstack` deadlock detection covers `synchronized` and, with `-l`, `ReentrantLock`; it can't see "waiting for a message that never arrives".
- Compare dumps: a thread in the same place in all three dumps is stuck, not just slow.

**Remember:** *Low CPU + hang → thread dumps ×3: look for "Found deadlock", all-WAITING pools, or one lock owner stuck in I/O.*

---

### Q245. Users complain about long pauses. How do you diagnose and fix long GC pauses?

**Situation:** Every few minutes all requests freeze for 1-5 seconds, then continue. CPU is normal outside the freeze.

**Say this first:** "A *stop-the-world* (STW) GC pause freezes all application threads. The GC log tells me which collection, how long, and why."

**How you investigate:**
1. Make sure GC logging is on (it is cheap): Java 9+ `-Xlog:gc*:file=gc.log:time,uptime,level,tags`; Java 8 `-XX:+PrintGCDetails -XX:+PrintGCDateStamps -Xloggc:gc.log`.
2. Match pause timestamps to the freezes. Read the lines:
```
[gc] GC(812) Pause Young (Normal) (G1 Evacuation Pause) 900M->410M(1024M) 18.4ms
[gc] GC(813) Pause Full (Allocation Failure)  1020M->980M(1024M) 3120.5ms   <- the problem
```
3. Look at the **cause** and the **heap after GC**:
   - `Pause Full` with heap still ~full afterwards → heap too small *or* a leak.
   - Many **to-space exhausted / evacuation failure** (G1) → no room to copy survivors; increase heap or reduce allocation.
   - **Humongous allocations** (G1: objects bigger than half a region) → giant arrays/strings allocated in bursts; chunk them.
   - Long pause with **`sys` time ≫ `user` time** → the OS is swapping or the container is being throttled - GC isn't the real problem.
   - Pause is long but GC work is tiny → a **safepoint** delay (`-Xlog:safepoint`): a thread in a long counted loop can't reach a safepoint quickly.
4. Tools: GC log viewers (GCeasy, GCViewer), JFR "GC" events, `jstat -gcutil <pid> 1000`.

**Fixes (in this order):**
- **Reduce allocation rate/garbage**: avoid creating huge temporary collections, reuse buffers, stream instead of materialise, avoid boxing in hot loops.
- **Fix leaks** that grow the old generation (forcing Full GCs).
- **Right-size the heap** (not too small; also not huge with an old collector).
- **Pick the right collector**: G1 (default since Java 9) targets pause goals: `-XX:MaxGCPauseMillis=200`; **ZGC** (`-XX:+UseZGC`) or **Shenandoah** (where available) give very short pauses on big heaps, trading some throughput/CPU.
- Disable `System.gc()` callers (`-XX:+DisableExplicitGC`) only after checking that nothing - e.g. direct-buffer cleanup - relies on it.
- Avoid swap: `-Xms` = `-Xmx` in servers, and `-XX:+AlwaysPreTouch` if memory should be committed up front.

**Prevention:** keep GC logs rotated and shipped; alert on max pause and on GC-time percentage; test with production-sized heaps; don't tune flags blindly - change one thing, measure.

**Pitfalls / follow-up probes:**
- "Throughput vs latency collector?" Parallel GC maximises throughput (long pauses); G1 balances; ZGC minimises pauses.
- Young GC pauses grow if lots of objects survive (copying cost is proportional to survivors, not garbage).
- More heap isn't always better: with an old collector, a bigger heap can mean a *longer* full GC.

**Remember:** *Read the GC log: which pause, what cause, what heap after GC - then cut allocation, fix leaks, or change collector.*

---

### Q246. p99 latency spikes while the average looks fine. How do you track down tail latency?

**Situation:** Average response is 40 ms, but 1% of requests take 2-3 s. Users notice, dashboards of averages look green.

**Say this first:** "Averages hide tails. p99 is where *waiting* shows up: queues, pauses, locks, slow dependencies. I look for what is **periodic or correlated** with the spikes."

**How you investigate - checklist of suspects:**

| Suspect | How to check |
|---|---|
| GC pauses | Do spike timestamps match GC-log pauses? |
| Thread-pool queueing | Pool queue size/active threads at spike time (requests wait before running) |
| Connection-pool wait | Time spent acquiring a connection vs running the query |
| Lock contention | Thread dump / JFR "Java Monitor Blocked" events |
| Slow dependency | Per-call timers/traces around each remote call; p99 of the downstream |
| Cold paths | JIT warm-up, first-use class loading, cold caches after deploy |
| CPU throttling | Container CPU limit hit: `cat /sys/fs/cgroup/cpu.stat` → `nr_throttled`, `throttled_usec` |
| Noisy neighbour / swap / DNS / retries | Host metrics; DNS lookup time; retry counts |

Practical steps:
1. **Break one request into stages** with timers (queue wait, validation, DB, remote call, serialisation). The stage whose p99 explodes is your suspect.
2. **Correlate** spike times with GC log, deploys, cron jobs, backups, cache expiry (a *thundering herd* when many entries expire together).
3. Capture a **JFR** recording during the bad period (`jcmd <pid> JFR.start duration=120s filename=p99.jfr settings=profile`) and look at latency, locks, I/O, GC.
4. If a load test shows the problem, beware **coordinated omission**: a tool that waits for each response before sending the next under-reports latency during a stall. Use an open-model load generator.

**Typical fixes:**
- Shorten pauses (GC tuning/ZGC), reduce allocation.
- Separate pools/bulkheads so slow work can't queue fast work; bound queues.
- **Timeouts** shorter than the SLA + limited retries with jitter; for idempotent reads, **hedged requests** (send a second after the p95 time).
- Cache with jittered expiry; request coalescing for hot keys.
- Raise/remove the CPU limit or set it with care (throttling hurts latency more than average).

**Prevention:** SLOs on percentiles, per-stage latency histograms, distributed tracing, and alert on p99 not average.

**Pitfalls / follow-up probes:**
- Averaging percentiles across instances is wrong; aggregate histograms instead.
- Fan-out amplifies tails: if a request calls 10 services in parallel, the chance of hitting at least one slow p99 call is ~10%.
- "Is it the JVM or the app?" - JVM events (GC, safepoints, compilation) are visible in the GC log and JFR.

**Remember:** *Average hides, p99 reveals waiting - split the request into stages and correlate with GC, queues, locks and throttling.*

---

### Q247. `ConcurrentModificationException` appears in production. What causes it and how do you fix it?

**Situation:** Occasional `java.util.ConcurrentModificationException` in logs, from a `for-each` over a list or map; sometimes only under load.

**Say this first:** "It means a collection was structurally modified while an iterator was walking it. It can happen in a **single thread** too - not only with multithreading."

**How it works:** `ArrayList`/`HashMap` keep a `modCount` counter. The iterator remembers the value at creation and, on each `next()`, throws if it changed. This is **fail-fast** and **best effort**: it's a bug detector, not a thread-safety guarantee.

**How you investigate:** the stack trace points at `next()` of the iterator; look at the loop's body and who else touches the same collection.

**Root cause 1 - single thread, removing inside the loop:**
```java
for (Order o : orders) {
    if (o.isCancelled()) orders.remove(o);        // CME on next iteration
}
```
*Fix:*
```java
orders.removeIf(Order::isCancelled);                       // best
// or iterator.remove():
for (Iterator<Order> it = orders.iterator(); it.hasNext();) {
    if (it.next().isCancelled()) it.remove();
}
```
**Root cause 2 - two threads:** one iterates a shared `ArrayList`/`HashMap`, another adds/removes. Here CME is the *lucky* outcome; without it you could silently see torn data.
*Fix options:*
- `ConcurrentHashMap` / `CopyOnWriteArrayList` (weakly consistent iterators, never throw CME; COW is for read-mostly small lists because every write copies the array).
- Iterate over a **snapshot**: `new ArrayList<>(list)` taken under the same lock that writers use.
- Guard all access with one lock, including the iteration.
- Better: don't share - make the collection immutable and swap a new version atomically (`volatile` reference to an unmodifiable copy).

**Prevention:**
- Don't expose internal mutable collections (return unmodifiable views/copies).
- Code review rule: "any shared collection has a named owner and a thread-safety policy".
- Test with concurrent stress (see testing section).

**Pitfalls / follow-up probes:**
- `Collections.synchronizedList` does **not** make iteration safe - you must `synchronized(list){...}` around the loop.
- Removing the *second-to-last* element in a for-each of an `ArrayList` doesn't throw (the loop just ends early) - a known quirk showing fail-fast is best effort.
- Modifying a `HashMap` value via `map.put(existingKey, v)` is not a structural change, but adding a new key is.

**Remember:** *CME = structural change during iteration; use removeIf/iterator.remove, or a concurrent/immutable collection - never rely on catching it.*

---

### Q248. A `HashMap` shared by threads: corrupted data, lost updates, even an infinite loop. What happened?

**Situation:** A cache/registry built on `HashMap` works in tests, then in production returns wrong or missing values, the count is off, and in an old JVM a thread spins at 100% CPU forever.

**Say this first:** "`HashMap` is **not thread-safe**. Concurrent writes can lose entries or corrupt its internal structure; reads concurrent with writes can see anything."

**How you investigate:**
- A thread dump with a thread `RUNNABLE` inside `HashMap.get`/`put`/`transfer` repeatedly, consuming CPU (use the 100%-CPU recipe: `top -H`, hex `nid`, `jstack`).
- Symptoms: `size()` disagrees with the content, keys that "disappeared", `NullPointerException` or `ClassCastException` from inside `HashMap`.

**Root cause:**
- **Lost updates:** two threads `put` into the same bucket; both read the old state, the last write wins - one entry vanishes. Or `map.put(k, map.get(k) + 1)` - a read-modify-write race.
- **Java 7 infinite loop:** while **resizing**, the bucket's linked list was re-linked in reverse order; two threads doing it concurrently could create a **cycle** in the list, so a later `get` on that bucket loops forever. Java 8 changed the resize algorithm (preserves order), which removed that specific loop, but concurrent use is **still unsafe** (lost entries, inconsistent size, visibility problems) - "fixed in 8" is not a licence to share a `HashMap`.
- **Visibility:** without a happens-before edge, another thread may see a stale table or a half-built object.

**Fix:**
```java
Map<String, LongAdder> counts = new ConcurrentHashMap<>();
counts.computeIfAbsent(key, k -> new LongAdder()).increment();     // atomic per-key operations
counts.merge(key, 1L, Long::sum);                                   // atomic read-modify-write
```
- Use `ConcurrentHashMap` and its **atomic** methods (`compute`, `merge`, `putIfAbsent`) - not `if (!map.containsKey(k)) map.put(k, v)`, which is a check-then-act race.
- Don't call a slow function or touch the same map from inside `computeIfAbsent`'s mapping lambda (it holds a bin lock; recursive updates can deadlock/throw, and a historic Java 8 bug could livelock).
- If the map is built once and then read-only: build it, wrap with `Map.copyOf`/`Collections.unmodifiableMap`, and publish it safely (`final` field or `volatile`).
- `Collections.synchronizedMap` gives method-level safety but compound actions still need external locking.

**Prevention:** document thread-safety of every shared field; static analysis (SpotBugs, ErrorProne) flags many of these; stress tests with many threads.

**Pitfalls / follow-up probes:**
- `ConcurrentHashMap` does not allow `null` keys/values - deliberately, to avoid ambiguity of `get` returning null.
- Is `ConcurrentHashMap.size()` exact? It's an estimate under concurrent updates (`mappingCount()` for big maps).
- `Hashtable` is thread-safe but coarse and legacy.

**Remember:** *Shared HashMap = lost updates and corruption (Java 7: infinite loop); use ConcurrentHashMap with atomic compute/merge.*

---

### Q249. Dates are randomly wrong under load - a `SimpleDateFormat` thread-safety bug.

**Situation:** A utility has `private static final SimpleDateFormat FMT = new SimpleDateFormat("yyyy-MM-dd");`. Under load you see wrong dates, `NumberFormatException: multiple points`, or `ArrayIndexOutOfBoundsException` from `DateFormat.parse`. It never fails in a single-thread test.

**Say this first:** "`SimpleDateFormat` keeps mutable scratch state (an internal `Calendar`) while formatting/parsing, so sharing one instance between threads corrupts it."

**How you investigate:**
- Stack traces pointing inside `java.text.DecimalFormat`/`SimpleDateFormat.parse`.
- Search the code base for `static ... SimpleDateFormat` / `DateFormat` / `NumberFormat` fields.
- Reproduce with a tiny stress test:
```java
ExecutorService pool = Executors.newFixedThreadPool(16);
for (int i = 0; i < 100_000; i++) {
    pool.submit(() -> FMT.parse("2024-05-17"));     // random exceptions / wrong dates
}
```
**Root cause:** `format()`/`parse()` write to the shared internal `Calendar` field; threads overwrite each other's intermediate values (a data race on mutable shared state).

**Fix (best → acceptable):**
```java
// 1. java.time (Java 8+): immutable and thread-safe
private static final DateTimeFormatter FMT = DateTimeFormatter.ofPattern("yyyy-MM-dd");
LocalDate d = LocalDate.parse("2024-05-17", FMT);

// 2. Legacy API required? One instance per thread
private static final ThreadLocal<SimpleDateFormat> FMT =
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
// 3. Or just create a new instance per call (cheap enough for most code paths)
```
Remember that `DateTimeFormatter` patterns are case-sensitive: `yyyy` vs `YYYY` (week-based year) and `MM` (month) vs `mm` (minutes) are classic bugs that give "wrong dates" without any concurrency at all.

**Prevention:**
- Ban `SimpleDateFormat`/`java.util.Date`/`Calendar` in new code (use `java.time`); add a static-analysis rule.
- Rule: a `static` field must be immutable or thread-safe. Review every shared mutable static.

**Pitfalls / follow-up probes:**
- Why not just `synchronized` around the format? Works, but serialises threads - a bottleneck.
- A `ThreadLocal` on pooled threads must be small and cleaned if its value holds a classloader (see the leak-sources question).
- `NumberFormat`/`DecimalFormat` have the exact same problem.

**Remember:** *SimpleDateFormat is stateful, not thread-safe - use immutable DateTimeFormatter.*

---

### Q250. Memory leaks via static collections, `ThreadLocal`, listeners and classloaders - how do you recognise each?

**Situation:** A heap dump shows a growing object graph and "Path to GC roots" leads to one of the structures below.

**Say this first:** "Every leak is the same story - a **long-lived GC root** (static field, running thread, classloader, JNI) holds a reference to something that should have died. I look for the root."

| Leak source | Typical root | Fix |
|---|---|---|
| **Static collection / unbounded cache** | `static Map<K,V>` that only gets `put` | Bound size + eviction (LRU `LinkedHashMap.removeEldestEntry`, or a cache library with max size/TTL); use `WeakHashMap` only for identity-like keys and know its caveats |
| **`ThreadLocal` in pooled threads** | Pool thread (lives forever) → `ThreadLocalMap` → value | `try { tl.set(x); ... } finally { tl.remove(); }` |
| **Listeners / callbacks / observers** | Long-lived event source holds a registered listener | Always deregister (`removeListener`) in `close()`/shutdown; or use weak references |
| **Classloader leak** (redeploys, plugins) | A parent-loader object/thread/driver holds a class from the child loader, pinning *all* of its classes (shows up as Metaspace growth) | Stop threads, deregister JDBC drivers/shutdown hooks, clear `ThreadLocal`s and statics on undeploy |
| **Inner/anonymous class capturing `this`** | A callback registered globally keeps the outer object alive | Use static nested class or lambda that doesn't capture; unregister |
| **Unbounded queue/list** | Producer faster than consumer | Bounded queue + backpressure |
| **Not closing resources** | Streams, connections hold buffers | try-with-resources |
| **Interning / `String.intern` abuse, huge keys** | Pool/cache grows | Don't intern user data |

```java
// ThreadLocal leak: value lives as long as the pool thread, i.e. forever
private static final ThreadLocal<byte[]> BUF = ThreadLocal.withInitial(() -> new byte[10_000_000]);
// 200 pool threads * 10 MB = 2 GB retained, even when idle
```

**How you investigate:** MAT → dominator tree → pick the big object → *Path to GC Roots (exclude weak/soft)*. The first non-JDK frame in that path is your owner. For classloaders: MAT's "Duplicate classes"/"Classloader explorer", `jcmd <pid> VM.classloader_stats`.

**Prevention:**
- Any `static` collection needs a size bound and a removal path. Review "register" calls: is there always an "unregister"?
- Keep `ThreadLocal` values small, and always `remove()` in `finally`. With virtual threads, per-thread caches are less sensible (millions of threads) - prefer explicit passing or scoped values (preview/new APIs).
- Memory soak tests and the "heap after GC" alert.

**Pitfalls / follow-up probes:**
- Since JDK 7u6, `String.substring` copies - the old "substring keeps the big string alive" leak is gone.
- `WeakReference` value caches can be collected too eagerly; `SoftReference` caches are cleared under memory pressure but give unpredictable behaviour - a real cache library is better.
- Finalizers/`Cleaner` mishandled can delay freeing native resources.

**Remember:** *Find the GC root: static field, pooled thread, listener list, or classloader - cut that reference.*

---

### Q251. `StackOverflowError` in production - recursion, `toString` cycles, and how to fix.

**Situation:** Requests fail with `java.lang.StackOverflowError` and a stack trace thousands of lines long, often with the same few frames repeated.

**Say this first:** "Each thread has a fixed-size stack (default usually 512 KB-1 MB per thread, set by `-Xss`); every method call uses a frame. Unbounded or very deep recursion exhausts it."

**How you investigate:** read the repeating frames at the top of the trace - they *are* the cycle:
```
at com.acme.Order.toString(Order.java:42)
at com.acme.Customer.toString(Customer.java:30)
at com.acme.Order.toString(Order.java:42)
...
```
The JVM caps the printed depth (about 1024 frames by default, `-XX:MaxJavaStackTraceDepth`), so the *bottom* of the real stack may be cut; the repeating pattern at the top is usually enough.

**Common root causes:**
1. **Missing/wrong base case** in a recursive function (`factorial(n)` with `n` never reaching 0, or a negative/very large input).
2. **Bidirectional references in `toString`/`equals`/`hashCode`/JSON serialisation** (`Order → Customer → List<Order> → ...`). The cycle never ends.
3. **Mutual recursion** between two methods or two listeners/decorators calling each other.
4. A **legitimately deep** structure (very long linked list, deeply nested JSON/regex on long input) processed recursively. Regex engines can overflow on long inputs with alternations.
5. Class initialisation cycles, or a proxy/decorator that wraps itself.

**Fix:**
```java
// break cycles: exclude the back-reference
@Override public String toString() { return "Order[id=" + id + ", customerId=" + customer.id() + "]"; }

// convert deep recursion to iteration with an explicit stack/queue
Deque<Node> stack = new ArrayDeque<>();
stack.push(root);
while (!stack.isEmpty()) { Node n = stack.pop(); visit(n); n.children().forEach(stack::push); }
```
- Validate/limit input depth (reject JSON nested 10 000 levels, limit regex input size).
- If you truly need deep recursion for a worker, run it in a dedicated thread with a bigger stack: `new Thread(null, task, "deep", 64L * 1024 * 1024)`; raising global `-Xss` multiplies memory by the thread count, so don't.

**Prevention:** unit-test `toString`/`equals`/`hashCode` on cyclic graphs; make object graphs acyclic or reference by id; add depth limits on untrusted input; tail-call elimination does **not** exist in Java, so don't rely on it.

**Pitfalls / follow-up probes:**
- `StackOverflowError` is an `Error`; after it, state may be inconsistent - don't blindly catch it.
- It's per thread: a thread-pool thread has the same `-Xss` as the main thread unless set otherwise.
- JIT inlining changes where it overflows, so depth varies between runs.

**Remember:** *Repeating frames = the cycle; fix the base case/back-reference or iterate with an explicit stack.*

---

### Q252. "Too many open files" - file, socket and connection leaks.

**Situation:** Errors: `java.io.IOException: Too many open files`, `SocketException: ... Too many open files`, or "connection pool exhausted / timeout waiting for connection". Gets worse over hours; restarts help.

**Say this first:** "Files, sockets and DB connections are OS-level resources. The JVM doesn't close them when the object becomes garbage in a timely way - *I* must close them, deterministically."

**How you investigate:**
```bash
ls /proc/<pid>/fd | wc -l                  # current open file descriptors (FDs)
ulimit -n ; cat /proc/<pid>/limits | grep 'open files'     # the limit (often 1024 by default!)
lsof -p <pid> | awk '{print $5}' | sort | uniq -c | sort -rn | head    # types: REG, IPv4, pipe...
lsof -p <pid> | grep -i reg | awk '{print $9}' | sort | uniq -c | sort -rn | head   # which files
ss -tanp | grep <pid> | awk '{print $2}' | sort | uniq -c            # socket states
```
- Many **`CLOSE_WAIT`** sockets = the remote side closed, *our app never called `close()`* → a leak in our code.
- Many `TIME_WAIT`/`ESTABLISHED` to one host = missing connection reuse/pooling.
- In-process count: `((com.sun.management.UnixOperatingSystemMXBean) ManagementFactory.getOperatingSystemMXBean()).getOpenFileDescriptorCount()`.
- Connection pools: thread dump shows threads waiting in `getConnection`, while pool stats show *active == max*. Pool "leak detection" settings print the stack where a connection was borrowed.

**Root causes:**
```java
// leaks if readLine throws
BufferedReader r = new BufferedReader(new FileReader(f));
String line = r.readLine();   // no close on exception

// Streams backed by an open directory/file - MUST be closed
Files.lines(p).count();       // leak!   Files.list/walk/find too
```
Also: `HttpURLConnection` response streams not drained/closed, new HTTP client per request, `ZipFile`/`JarFile`, `ResultSet`/`Statement`/`Connection` not closed on error paths, a `ProcessBuilder` process with unread output.

**Fix:**
```java
try (Stream<String> lines = Files.lines(p);
     Connection c = ds.getConnection();
     PreparedStatement ps = c.prepareStatement(SQL)) {
    ...
}   // closed in reverse order, even when an exception is thrown
```
- Reuse one HTTP client; set connect and read timeouts.
- Return pool connections in `finally`/try-with-resources; set a pool max wait timeout.
- Raise the OS limit (`ulimit -n 65535` / systemd `LimitNOFILE`) as a *safety margin only*.

**Prevention:** static analysis (SpotBugs "OBL"/"RCN" patterns, IDE warnings for unclosed `AutoCloseable`); alert on FD count versus the limit; pool leak detection in non-prod and prod.

**Pitfalls / follow-up probes:**
- `finalize()` and `Cleaner` are a safety net, not a strategy - they run late (or never) so FDs run out before GC notices.
- In try-with-resources, if both the body and `close()` throw, the `close()` exception is attached as **suppressed** (`getSuppressed()`).
- The GC can't see FD pressure, so a quiet heap = no cleanup.

**Remember:** *Count the FDs, find CLOSE_WAIT or the leaking file type, and close everything with try-with-resources.*

---

### Q253. The thread-pool queue keeps growing and memory rises. What is wrong with the executor?

**Situation:** Latency climbs, memory grows, and a dump shows millions of `Runnable`/`FutureTask` objects. No errors - until `OutOfMemoryError`.

**Say this first:** "A queue with no bound hides overload: it converts 'too much work' into 'infinite waiting and memory growth'. Overload must be visible and limited."

**How you investigate:**
- Expose or log `ThreadPoolExecutor` numbers: `getActiveCount()`, `getPoolSize()`, `getQueue().size()`, `getCompletedTaskCount()`, and the rejection count.
- Heap histogram: huge counts of `java.util.concurrent.FutureTask`, `LinkedBlockingQueue$Node`.
- Thread dump: all pool threads busy/blocked (on what? a slow DB or API?).

**Root cause:** arrival rate > service rate, and nothing pushes back.
- `Executors.newFixedThreadPool(n)` and `newSingleThreadExecutor()` use an **unbounded** `LinkedBlockingQueue` → the queue grows forever.
- `Executors.newCachedThreadPool()` has an unbounded **number of threads** → `unable to create native thread`.
- The trigger is usually a slow downstream, a traffic burst, or tasks that block.

**Fix - bounded queue + explicit rejection policy:**
```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    8, 16,                                   // core, max threads
    60, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(200),           // bounded queue
    namedThreadFactory("report-worker"),     // your own ThreadFactory so dumps show readable names
    new ThreadPoolExecutor.CallerRunsPolicy());    // slows the producer down = backpressure
```
Rejection policies: `AbortPolicy` (throw `RejectedExecutionException`, caller returns 429/503), `CallerRunsPolicy` (caller does the work - natural back-pressure), `DiscardPolicy`/`DiscardOldestPolicy` (silent loss - only when acceptable).

**How `ThreadPoolExecutor` really behaves (often asked):** it first creates threads up to *core*; then fills the **queue**; only when the queue is **full** does it add threads up to *max*; then it rejects. So with an *unbounded* queue, `max` is never used.

**Prevention:**
- Size pools using the workload: CPU-bound ≈ number of cores; I/O-bound ≈ cores × (1 + wait/compute) - and measure.
- Timeouts on tasks and downstream calls, circuit breaker/bulkhead for flaky dependencies, load shedding at the edge (reject early).
- Metrics + alerts on queue depth and rejected count; shut pools down on application stop.

**Pitfalls / follow-up probes:**
- A `parallelStream()` runs on the shared common `ForkJoinPool` - blocking calls there starve other users.
- `CompletableFuture.supplyAsync` without an executor argument uses the common pool too.
- `ScheduledThreadPoolExecutor` has an unbounded delayed queue; a task that throws stops its own periodic schedule silently unless you catch exceptions inside it.

**Remember:** *Unbounded queue = hidden overload; bound it, choose a rejection policy, and watch queue depth.*

---

### Q254. The application takes minutes to start. How do you find and speed up slow startup?

**Situation:** A plain Java service starts in 90 seconds locally but 4 minutes in the cluster; health checks fail, deployments time out.

**Say this first:** "Startup is mostly class loading, static initialisation, I/O and warm-up. I measure where time goes before changing anything."

**How you investigate:**
1. **Timestamps in logs** around phases (config load, DB connect, caches warmed...). Often one step is 80% of the time.
2. **Thread dumps during startup** (3-4 samples, 10 s apart): what is the main thread doing? `socketRead` → waiting for a remote service; `Class.forName`/`ClassLoader.loadClass` → class loading/scanning; `SecureRandom.nextBytes` → entropy.
3. **JFR from the start:** `-XX:StartFlightRecording=duration=120s,filename=startup.jfr` and open in JDK Mission Control.
4. **Class loading:** `-Xlog:class+load:file=classload.log` (Java 9+; `-verbose:class` on 8) - count lines; thousands of loads from many jars means classpath scanning or a huge dependency set.
5. **Compare environments:** CPU limit in the container (tiny CPU quota makes JIT and class loading slow), disk speed, DNS, network latency to DB/config server.

**Common root causes and fixes:**

| Cause | Fix |
|---|---|
| Eager loading of big data / warming caches synchronously | Load lazily or in the background; start serving with a readiness flag |
| Remote calls in static initialisers or constructors (DNS, DB, config) with long timeouts | Timeouts, retries off the critical path, parallelise independent calls |
| Classpath scanning / reflection-heavy libs on a huge classpath | Remove unused jars, build an index at build time, avoid scanning |
| `SecureRandom` blocking on `/dev/random` (older Linux/VM) | `-Djava.security.egd=file:/dev/./urandom` (check your security policy) |
| JIT cold start, too few CPUs | Give more CPU at startup; use **AppCDS** (class-data sharing archive: `-XX:ArchiveClassesAtExit=app.jsa`, then `-XX:SharedArchiveFile=app.jsa`, Java 13+) to speed class loading |
| Huge heap with pre-touch | Drop `-XX:+AlwaysPreTouch` if start time matters more |
| Sequential initialisation of independent components | Initialise concurrently with bounded executors |

**Prevention:** track startup time as a metric per build (fail the build on a regression); separate *liveness* from *readiness* probes with a startup grace period; keep static initialisers trivial.

**Pitfalls / follow-up probes:**
- A "slow static initialiser" that throws leaves the class unusable - later `NoClassDefFoundError: Could not initialize class X`.
- Don't judge by the first request: the JVM gets faster after warm-up (JIT). Distinguish "slow start" from "slow first requests".
- Ahead-of-time options (native image, snapshot/restore approaches) exist but change the runtime model; mention them as options, not first steps.

**Remember:** *Measure first - logs, startup thread dumps, JFR; fix the single dominant step (remote calls, scanning, eager loading).*

---

### Q255. How do you root-cause a `NullPointerException` in production quickly?

**Situation:** Log: `java.lang.NullPointerException` at `OrderService.java:87` - and the line contains four chained calls.

**Say this first:** "On modern Java the NPE message tells me *which* expression was null. Java 14 introduced *helpful NullPointerException messages*, on by default since Java 15."

**How you investigate:**
```
Exception in thread "main" java.lang.NullPointerException:
  Cannot invoke "com.acme.Address.city()" because the return value of
  "com.acme.Customer.address()" is null
```
That message names the call that failed (`Address.city()`) and **why** (`Customer.address()` returned null). Local variable names appear only if the class was compiled with `-g` (debug info); otherwise you see `"<local4>"`. Fields appear as `because "this.customer" is null`, array slots as `"arr[2]"`.

On Java 8-13 (no helpful messages): split the chain onto separate lines or temporarily add checks to pinpoint which part was null, and enable the flag on 14: `-XX:+ShowCodeDetailsInExceptionMessages`.

Then ask **why** it was null - the *source*, not the line where it blew up:
- Did a lookup return `null` (map `get`, repository `find`)?
- Did a constructor/setter leave a field unset on one code path?
- Missing data (optional column, absent JSON field, missing config)?
- Auto-unboxing: `int x = map.get(key);` throws NPE if the key is missing.
- Race/visibility: a field read before initialisation completed (unsafe publication).

**Fix:** fix the *origin*, and decide the semantic: **can it be null?**
```java
// make absence explicit
Optional<Customer> find(String id);                   // return type tells the story
String city = find(id).map(Customer::address).map(Address::city).orElse("UNKNOWN");

// fail fast with a clear message at the boundary
this.repo = Objects.requireNonNull(repo, "repo");

// default for collections: never return null, return List.of()
```
- Validate input at the boundary (constructor/public method) so the failure appears *where the bad data entered*.
- Use `@Nullable`/`@NonNull` annotations plus a checker (IDE inspections, NullAway/Checker Framework).
- Use records/immutable value objects with non-null invariants.

**Prevention:** tests for "missing/empty" inputs; code review rule "no method returns null for collections/Optional"; `Optional` for return values (not for fields/parameters); log the business key (order id) next to the stack trace.

**Pitfalls / follow-up probes:**
- Catching `NullPointerException` to handle control flow hides bugs.
- `Optional.get()` without `isPresent` just moves the NPE (`NoSuchElementException`); prefer `orElseThrow` with a message.
- Stack traces in production may be truncated by "fast throw" optimisation: the JIT can reuse a preallocated exception with **no stack trace** for hot repeated exceptions (`-XX:-OmitStackTraceInFastThrow` disables that). Look at *earlier* log lines of the same exception.

**Remember:** *Read the helpful NPE message for the null expression, then fix where null was born - make absence explicit.*

---

### Q256. A bug appears once in a thousand runs and vanishes when you add logging or a debugger (a "Heisenbug"). How do you reproduce a race condition?

**Situation:** Rare wrong results - duplicate orders, negative stock, a counter lower than expected - never reproducible locally; adding `System.out` makes it disappear.

**Say this first:** "Logging and debuggers change timing, which hides races. So I stop watching and start *forcing* the bad interleaving."

**How you investigate:**
1. **Look for the classic race patterns** by reading the code:
   - **Check-then-act:** `if (!map.containsKey(k)) map.put(k, v);`, `if (stock > 0) stock--;`
   - **Read-modify-write on shared state:** `count++` on a non-atomic field (three steps: read, add, write).
   - **Missing visibility:** a non-`volatile` flag read in a loop; a half-constructed object published via a non-final field; broken double-checked locking (needs `volatile`).
   - **Iteration while modifying**, shared non-thread-safe helpers (`SimpleDateFormat`, `HashMap`).
2. **Write a stress test** that aligns threads at the start to maximise contention:
```java
int N = 32, loops = 10_000;
CyclicBarrier go = new CyclicBarrier(N);
AtomicInteger errors = new AtomicInteger();
ExecutorService ex = Executors.newFixedThreadPool(N);
for (int t = 0; t < N; t++) ex.submit(() -> {
    go.await();                                   // all threads start together
    for (int i = 0; i < loops; i++) service.reserve("sku-1");
    return null;
});
ex.shutdown(); ex.awaitTermination(1, TimeUnit.MINUTES);
assertEquals(INITIAL - N * loops, stock.get());   // invariant broken => race found
```
3. **Make the race wider:** run with more threads than cores, repeat thousands of times, run on a loaded machine (CI), or temporarily insert `Thread.yield()`/tiny sleeps *between* the check and the act (in a test copy) to expose the window. For low-level memory-model questions, the OpenJDK **jcstress** harness exists.
4. **Observe without perturbing:** in-memory ring buffer of events with `System.nanoTime()` and thread id, dumped after failure; or **JFR** events (very low overhead); include thread names in logs.
5. **Static analysis:** SpotBugs/ErrorProne concurrency checks (e.g. unsynchronised access, inconsistent synchronisation).

**Fix:** pick the simplest correct tool - immutability and confinement first; then atomics (`AtomicInteger`, `LongAdder`, `compareAndSet`), concurrent collections with atomic methods, `synchronized`/`Lock` around the *whole* invariant, or a database-level guard (unique constraint, `UPDATE ... WHERE stock > 0`, optimistic version column) when state lives in a DB or across JVMs.

**Prevention:** document the thread-safety policy of every class (`@ThreadSafe`/`@NotThreadSafe`), minimise shared mutable state, add a stress test for every fixed race so it can't regress.

**Pitfalls / follow-up probes:**
- `volatile` gives visibility and ordering but **not atomicity** of `count++`.
- Making one method `synchronized` doesn't help if another path accesses the field unsynchronised - all accesses need the same lock.
- A race inside one JVM is fixed by locks; across multiple instances you need a DB constraint or distributed lock.

**Remember:** *Don't watch the race - provoke it: barrier + many threads + invariant check; then fix the whole check-then-act as one atomic step.*

---

### Q257. Production logging pitfalls: log level, string concatenation, PII, and blocking appenders.

**Situation:** Throughput drops, disks fill up, a compliance team finds card numbers in logs, or the app freezes whenever the log volume (or the log disk) hiccups.

**Say this first:** "Logging is I/O on the hot path and also a data-leak channel. I treat it as a feature with a cost and a security policy."

**How you investigate / common problems and fixes:**

**1. Wrong level.** `DEBUG`/`TRACE` left on in production floods disks and slows every request. Use `INFO` for business events, `WARN` for recoverable problems, `ERROR` only for things needing action (otherwise alerts become noise). Make levels changeable at runtime (JMX/config reload).

**2. Building messages that are never printed.**
```java
log.debug("Order " + order + " items " + items);            // builds the string even if DEBUG is off (and calls toString on big objects)
log.debug("Order {} items {}", order.id(), items.size());    // parameterised: no formatting unless enabled
if (log.isDebugEnabled()) { log.debug("dump: {}", expensiveDump()); }   // guard expensive *argument computation*
// java.util.logging / System.Logger style: pass a Supplier
logger.log(Level.DEBUG, () -> "dump: " + expensiveDump());
```
**3. Logging PII/secrets** (card numbers, passwords, tokens, Aadhaar/SSN, full request bodies, `Authorization` headers). Log ids, not payloads; mask at the source (`toString` of sensitive types returns `****`); a central log-scrubbing filter as a safety net; restrict log access and set retention. Also beware **log injection** - user input with newlines can forge log lines: sanitise/encode.

**4. Blocking appenders.** A synchronous file/console/network appender makes request threads wait for the disk or the log collector. A full disk or slow log shipper → **the whole app stalls** (dump shows threads `BLOCKED` in the logging framework's lock/`PrintStream.write`). Fix: **async appender with a bounded queue** and an explicit policy (drop `DEBUG/INFO` when full, never block the request path), rotate and cap files, ship logs from files/stdout asynchronously.

**5. Other classics:**
- Logging in tight loops or per-row inside batch jobs → sample or summarise.
- **Log and rethrow** → the same exception logged at every layer; log **once**, where it is handled, with the cause/stack: `log.error("payment failed orderId={}", id, ex);`
- Swallowed exceptions: `catch (Exception e) {}` or `e.getMessage()` only (loses the stack and type).
- No **correlation id**: use a per-request id in MDC/context so one request can be traced across threads and services.
- Unstructured text → prefer structured (JSON key/values) for searchability.
- Time/timezone: log in UTC with ISO-8601.

**Prevention:** logging guidelines in the repo, PII tests (scan sample logs in CI), alerts on log volume and on the logger's queue drops, review new `log.*` calls like any other code.

**Pitfalls / follow-up probes:**
- Parameterised logging doesn't avoid the cost of *computing* arguments (`expensiveDump()`), hence the guard/supplier.
- Async logging can lose the last lines on a crash - a conscious trade-off.
- `System.out.println`/`e.printStackTrace()` are unbuffered, synchronized and uncontrolled - use a logger.

**Remember:** *Right level, parameterised messages, no PII, async bounded appenders, log an error once with its stack trace.*

---

### Q258. Time zone and DST bugs: reports are off by a day, scheduled jobs run twice or never.

**Situation:** Daily totals are wrong for some users; a job scheduled at 02:30 skipped once in spring and ran twice in autumn; dates shift when the server moves region; "it works in the office (IST) but not on the UTC server".

**Say this first:** "A date-time without a time zone is only a *label*, not a moment. Production servers usually run in UTC; developer laptops don't."

**How you investigate:**
- Compare the JVM zone in each environment: `-Duser.timezone`, `TimeZone.getDefault()`, `ZoneId.systemDefault()`, OS `TZ`. Differences = your bug.
- Look at the types: `LocalDateTime` (no zone), `Date` (an instant, but prints in the default zone), `Instant`, `ZonedDateTime`, `OffsetDateTime`.
- Reproduce with fixed zones in a unit test (`Clock.fixed(..., ZoneId.of("America/New_York"))`) around transitions.

**Root causes:**
1. Using `LocalDate.now()`/`LocalDateTime.now()` (implicitly the **server** zone) for business logic.
2. Converting an instant to a date using the wrong zone ("end of day" for a customer in Sydney ≠ UTC end of day).
3. **DST gaps/overlaps:** in `America/New_York` on 2024-03-10, local times 02:00-03:00 *don't exist*; on 2024-11-03, 01:00-02:00 happens *twice*.
```java
ZoneId ny = ZoneId.of("America/New_York");
ZonedDateTime gap = ZonedDateTime.of(2024, 3, 10, 2, 30, 0, 0, ny);
System.out.println(gap);                       // 2024-03-10T03:30-04:00  (shifted forward)
ZonedDateTime t = ZonedDateTime.of(2024, 3, 9, 12, 0, 0, 0, ny);
t.plusDays(1);                                 // same local time next day (12:00, only 23 hours later)
t.plus(Duration.ofHours(24));                  // exactly 24 h later (13:00 local)
```
4. Storing local times without zone in the database, or parsing `"2024-05-01 10:00"` with an implicit default zone.
5. A fixed offset (`+05:30`) used where a *region* (`Asia/Kolkata`) is needed - offsets don't know about DST rule changes.

**Fix / rules:**
- **Store and transmit instants in UTC** (`Instant`, ISO-8601 with `Z`/offset, or timestamp-with-time-zone columns); convert to the user's `ZoneId` only for display and for "calendar" logic (days, months, midnight).
- Keep the user's/business **zone id** alongside data that needs calendar semantics (a recurring "every day at 09:00 local" meeting).
- Use `Period`/`plusDays` for calendar arithmetic and `Duration` for exact elapsed time.
- Inject a `Clock` (`LocalDate.now(clock)`); set the JVM zone explicitly if you must (`-Duser.timezone=UTC`) but do not rely on it for correctness.
- Schedule jobs in UTC or handle gaps/overlaps explicitly and make jobs **idempotent**.
- Use the `tzdata` that matches reality: governments change rules; update the JDK/tzdata regularly.

**Prevention:** tests around DST dates, month ends, leap day, negative-offset and +14h zones; code review ban on `LocalDateTime.now()` in domain logic; log in UTC.

**Pitfalls / follow-up probes:**
- `java.util.Date` has no zone; its `toString()` prints the default zone, which misleads people into thinking it has one.
- `ZoneId.of("IST")` is ambiguous/short-id; use `Asia/Kolkata`.
- `LocalDateTime` + a zone = "assume this wall-clock time in that zone" - and may hit a gap.

**Remember:** *Store UTC instants, convert at the edges, inject the Clock, and test DST transitions.*

---

### Q259. Money calculations are off by a paisa/cent: `double` and `BigDecimal` rounding bugs.

**Situation:** Invoices total 0.01 less than the sum of their lines; reconciliation with the bank fails; comparisons like `price == 0.3` are randomly false.

**Say this first:** "Binary floating point (`double`) can't represent most decimal fractions exactly. Money needs exact decimal arithmetic with an explicit rounding rule."

**How you investigate:** reproduce with the failing numbers and print them with full precision:
```java
System.out.println(0.1 + 0.2);                    // 0.30000000000000004
System.out.println(new BigDecimal(0.1));          // 0.1000000000000000055511151231257827... (the double's exact value)
System.out.println(BigDecimal.valueOf(0.1));      // 0.1  (uses Double.toString)
```
Find every place where money touches `float`/`double`, or where a `BigDecimal` is created from a `double`.

**Root causes and fixes:**
1. **`double` for money** → use `BigDecimal`, or integer **minor units** (`long cents`) with a currency code.
2. **`new BigDecimal(double)`** captures the binary error → use `new BigDecimal("0.1")` or `BigDecimal.valueOf(0.1)`.
3. **`equals` vs `compareTo`:** `new BigDecimal("2.0").equals(new BigDecimal("2.00"))` is **false** (scale differs); use `compareTo(...) == 0` for numeric equality. This also breaks `HashSet`/`HashMap` keys - normalise with `stripTrailingZeros()` or `setScale(2)`.
4. **Division without a rounding rule:** `BigDecimal.ONE.divide(new BigDecimal(3))` throws `ArithmeticException: Non-terminating decimal expansion`. Always specify scale and `RoundingMode`:
```java
BigDecimal share = total.divide(BigDecimal.valueOf(3), 2, RoundingMode.HALF_EVEN);
```
5. **Rounding too early/too often:** rounding each line and summing gives a different total than summing then rounding. Decide the business rule (per line vs per invoice) and apply it consistently.
6. **Allocation leftovers:** splitting 100.00 into three parts gives 33.33 × 3 = 99.99. Distribute the remaining cent(s) deliberately (e.g. give the remainder to the first part).
7. **Rounding mode:** `HALF_UP` ("school") vs `HALF_EVEN` ("banker's", reduces bias) - use what the finance/legal rule requires; never rely on `Math.round` for money.
8. **Currencies with different minor units** (JPY 0 decimals, KWD 3) - use `Currency.getDefaultFractionDigits()`.

**Prevention:** a small immutable `Money` value type (amount + currency, scale rules, `equals`/`hashCode` consistent) used everywhere; tests with awkward values (0.1, 0.005, 1/3, huge amounts); property tests like "sum of allocated parts equals the total".

**Pitfalls / follow-up probes:**
- `BigDecimal` is immutable - `a.add(b)` returns a new object; ignoring the result is a common bug.
- JSON libraries may parse numbers into `double` by default - configure them to read decimals as `BigDecimal`/strings.
- Database `FLOAT`/`DOUBLE` columns have the same issue - use `DECIMAL(p,s)`.

**Remember:** *Never double for money: BigDecimal from strings, explicit scale + RoundingMode, compareTo for equality, allocate the remainder.*

---

### Q260. Duplicates in a `HashSet`, and `==` that is true for 127 but false for 128 - `equals`/`hashCode` and the Integer cache.

**Situation (1):** A `HashSet<Customer>` contains two "identical" customers; `map.get(key)` returns `null` for a key you just put; `contains` fails after a field changed.
**Situation (2):** `Integer a = 127, b = 127; a == b` → `true`, but `Integer c = 128, d = 128; c == d` → `false`; a comparison "works in tests, fails in production".

**Say this first:** "`==` compares **references** (are they the same object?). `equals` compares **meaning**. Hash-based collections need `equals` *and* `hashCode` to agree."

**How you investigate (1):**
- Check the class: is `equals` overridden? `hashCode`? Both with the *same fields*?
```java
class Customer {
    String id;
    @Override public boolean equals(Object o) { return o instanceof Customer c && id.equals(c.id); }
    // hashCode NOT overridden -> default identity hash
}
set.add(new Customer("1")); set.add(new Customer("1"));
set.size();   // 2 ! equal objects landed in different buckets
```
- Equal objects must have equal hash codes, otherwise the set looks in the wrong bucket and never calls `equals`.
- **Mutable key fields:** if `hashCode` depends on a field you later change, the entry sits in the old bucket and can't be found - **never mutate fields used in `hashCode` while the object is a key.**
- Overriding `equals(Customer c)` (wrong parameter type) **overloads** instead of overrides - `@Override` catches this.
- Inheritance: `equals` with `getClass()` vs `instanceof` and symmetry problems with subclasses.
- Entity ids that are `null` before persistence make poor `hashCode` input.

**Fix (1):** override both consistently using immutable business fields, or use a **`record`** (auto-generated `equals`/`hashCode`/`toString` from components):
```java
record CustomerId(String value) {}                 // value-based equality for free
@Override public int hashCode() { return Objects.hash(id); }
```
Compare objects with `Objects.equals(a, b)` (null-safe), compare strings/boxed values with `equals`, never `==`.

**Situation (2) explained:** `Integer.valueOf` (used by autoboxing) caches instances for **-128..127**, so `==` happens to work for small numbers and fails for others (the upper bound can be raised with `-XX:AutoBoxCacheMax`, so behaviour can even differ between environments). Same caching for `Long`, `Short`, `Byte`, `Character` (0-127), and `Boolean`. `new Integer(128)` (deprecated) always creates a new object.
```java
Integer a = 1000, b = 1000;
a == b;           // false (different objects)
a.equals(b);      // true
a == 1000;        // true - unboxes to int, compares values
```
Watch out for: `Long id` comparisons in business code (`order.getId() == other.getId()`), `list.get(i) == list.get(j)`, and **unboxing NPE** (`int x = nullInteger`). Likewise `String` `==` works only for interned literals.

**Prevention:** let the IDE/`record` generate equals/hashCode; static analysis (SpotBugs `RC_REF_COMPARISON`, `HE_EQUALS_USE_HASHCODE`, ErrorProne `ReferenceEquality`); write a unit test for `equals/hashCode` contract (EqualsVerifier library); never use `==` on wrapper types.

**Pitfalls / follow-up probes:**
- Bucket collisions are legal; `hashCode` returning a constant is *correct* but turns the map into a list (in Java 8+ treeified to O(log n) only for `Comparable` keys).
- `Arrays` `equals` vs `Arrays.equals`; a `byte[]` key uses identity.
- `BigDecimal` equals depends on scale (see the money question).

**Remember:** *Equal objects must have equal hash codes (and immutable fields); compare wrappers and strings with equals, never ==.*

---

### Q261. A request is slow because it calls a remote service inside a loop (the plain-Java "N+1").

**Situation:** An endpoint for a page of 100 items takes 8 seconds. Each item triggers a separate HTTP/DB call (`for (Order o : orders) { customer = client.fetch(o.customerId()); ... }`). It was fast in dev with 3 items.

**Say this first:** "Latency is `N × round-trip`. 100 calls × 80 ms = 8 s, and p99 explodes because the slowest of N calls dominates. The fix is fewer round-trips or concurrent ones - with limits."

**How you investigate:**
- Trace/timing per call or a **thread dump** during the slow request: the thread is repeatedly inside `socketRead`/`executeQuery` called from your loop.
- Count calls per request (metrics, debug proxy, network capture); compare with the number of items.
- Profile with JFR: many short blocking I/O events from one call site.

**Root cause:** the code treats a remote call like a cheap method call. Network latency, serialization, connection acquisition and the downstream's own load multiply per iteration (also N+1 with JDBC: one query for the list, then one per row).

**Fixes (in order of preference):**
1. **Batch:** one call with many ids - `client.fetchAll(ids)` or `SELECT ... WHERE id IN (...)` (chunk the list, e.g. 500 per call, to respect limits).
2. **Cache / dedupe:** many orders share the same customer → collect distinct ids first.
```java
Set<String> ids = orders.stream().map(Order::customerId).collect(toSet());
Map<String, Customer> byId = client.fetchAll(ids);          // 1 call
orders.forEach(o -> enrich(o, byId.get(o.customerId())));
```
3. **Parallelise with a bounded, dedicated executor** when batching isn't possible:
```java
ExecutorService io = Executors.newFixedThreadPool(16);       // created once, not per request
List<CompletableFuture<Customer>> fs = ids.stream()
    .map(id -> CompletableFuture.supplyAsync(() -> client.fetch(id), io)
                                .orTimeout(2, TimeUnit.SECONDS))
    .toList();
List<Customer> result = fs.stream().map(CompletableFuture::join).toList();
```
4. **Reduce the work:** pagination, return only needed fields, precompute/denormalise.

**Cautions:** an unlimited fan-out can overwhelm the downstream (use a bounded pool or semaphore); don't use `parallelStream()` for blocking I/O (shared common pool, no timeouts); always set **timeouts** and decide on partial failure; keep retries bounded with jitter. Virtual threads (Java 21+) make thread-per-call cheap, but you still need a concurrency limit for the downstream.

**Prevention:** code-review rule "no remote call or query inside a loop"; tests asserting the number of calls (verify the client is called once); load test with realistic page sizes; metrics on calls-per-request.

**Pitfalls / follow-up probes:**
- Batching changes failure semantics (all-or-nothing vs per item) - define it.
- Parallel calls reduce latency but *increase* load on the downstream.
- A fast local dev environment (localhost, 1 ms) hides this class of bug entirely.

**Remember:** *N calls = N round-trips: batch and dedupe first, bounded parallelism second, timeouts always.*

---

### Q262. `NoSuchMethodError`, `NoClassDefFoundError`, `ClassNotFoundException` - how do you diagnose classpath problems ("jar hell")?

**Situation:** The code compiles and tests pass, but on the server you get `java.lang.NoSuchMethodError: com.lib.Util.parse(Ljava/lang/String;)V` or `NoClassDefFoundError: org/apache/...`.

**Say this first:** "These are **link-time/runtime** errors: the class the JVM found at runtime is not the one we compiled against - a missing jar, a different *version*, or two copies of the same class."

**What each error means:**

| Error | Meaning |
|---|---|
| `ClassNotFoundException` | Code asked **by name** (`Class.forName`, reflection, service loading) and the loader couldn't find it |
| `NoClassDefFoundError` | Class existed at compile time but is **missing at runtime**, *or* its static initialiser failed earlier (`ExceptionInInitializerError`, then "Could not initialize class X") |
| `NoSuchMethodError` / `NoSuchFieldError` | Class found, but the **version has no such method/field** - compiled against v2, running with v1 |
| `LinkageError`, `IncompatibleClassChangeError`, `AbstractMethodError` | Binary-incompatible class versions |
| `UnsupportedClassVersionError` | Class compiled for a newer Java than the running JVM |

**How you investigate:**
1. **Where was the class loaded from?**
```bash
java -Xlog:class+load ... | grep 'com.lib.Util'         # Java 9+  (Java 8: -verbose:class)
```
```java
System.out.println(Util.class.getProtectionDomain().getCodeSource().getLocation());   // which jar won?
```
2. **Which versions are on the classpath, and why?**
```bash
mvn dependency:tree -Dverbose -Dincludes=com.lib        # Maven
gradle dependencyInsight --dependency lib --configuration runtimeClasspath   # Gradle
jar tf some.jar | grep Util                              # which jars contain the class
javap -cp lib-1.0.jar -p com.lib.Util                    # the method signature that actually exists
```
3. **Duplicates:** two jars (or a fat/shaded jar plus the original) contain the same class; the first on the classpath wins, order is not obvious.
4. For `NoClassDefFoundError: Could not initialize class X`, find the **first** failure earlier in the logs (`ExceptionInInitializerError`) - the root cause is in X's static init.
5. In app servers / plugin systems with several classloaders: the class may be visible to one loader but not another (parent-first vs child-first).

**Root cause:** a transitive dependency pulled a different version than expected (version *conflict*, Maven "nearest wins"), a `provided`/`optional` dependency not present at runtime, a mismatched deploy (some jars updated, some not), or shaded duplicates.

**Fix:**
- Pin one version centrally (Maven `dependencyManagement`, Gradle constraints/`resolutionStrategy`); exclude the unwanted transitive jar; align related libraries to the same release.
- Ship exactly the jars you tested (build one artifact, deploy it as-is).
- Shade/relocate packages only when two versions must really coexist.

**Prevention:** Maven Enforcer rules (`dependencyConvergence`, `banDuplicateClasses` via extra-enforcer-rules); run integration tests on the *runtime* classpath, not just the test classpath; keep dependency updates small and frequent; scan with `mvn dependency:analyze`.

**Pitfalls / follow-up probes:**
- Java 9+ modules: reflective access errors (`InaccessibleObjectException`, "module does not open ...") need `--add-opens` - a different category of error.
- `ClassNotFoundException` is a checked exception (Exception); `NoClassDefFoundError` is an `Error`.
- Class loading order of the classpath is first-found-wins; tests pass because the test classpath order differs.

**Remember:** *Find which jar the class really loaded from and which versions are on the runtime classpath; pin one version.*

---

### Q263. "Works on my machine": JDK, encoding, timezone and container memory differences.

**Situation:** The same jar behaves differently on a laptop, CI and a production container: garbled characters, different dates, `OutOfMemoryError`/OOMKilled only in production, or different thread counts.

**Say this first:** "My laptop and the container differ in the *environment the JVM reads at startup*: JDK version/vendor, default charset, locale, time zone, CPU and memory limits, and file system. I make each of those explicit."

**How you investigate - print the runtime facts in each environment:**
```bash
java -version
java -XshowSettings:properties -version 2>&1 | egrep 'file.encoding|user.timezone|user.language|java.vendor|os.name'
java -XshowSettings:system -version          # container/cgroup limits (Java 14+)
java -XX:+PrintFlagsFinal -version | egrep 'MaxHeapSize|UseG1GC|UseParallelGC|MaxRAMPercentage|ActiveProcessorCount'
```
```java
Runtime.getRuntime().availableProcessors();  Runtime.getRuntime().maxMemory();
Charset.defaultCharset();  ZoneId.systemDefault();  Locale.getDefault();
```

**Typical differences and fixes:**

| Difference | Symptom | Fix |
|---|---|---|
| **JDK version / vendor** | Different default GC, removed APIs, behaviour changes | Pin the JDK in build and runtime image (toolchains); same major version everywhere |
| **Default charset** | `é` becomes `?`, wrong byte lengths. Before Java 18 default = OS setting; **since Java 18 (JEP 400) default is UTF-8** | Always pass a charset: `new String(bytes, UTF_8)`, `Files.readString(p, UTF_8)`, `-Dfile.encoding=UTF-8` |
| **Time zone / locale** | Wrong dates, `1,5` vs `1.5` decimal formatting | `-Duser.timezone=UTC`, avoid default-locale APIs: `String.format(Locale.ROOT, ...)`, `toLowerCase(Locale.ROOT)` (the Turkish-`I` bug) |
| **Case-sensitive file systems, path separators, line endings** | File not found only on Linux | Use `Path`/`Paths`, exact-case names, `System.lineSeparator()` |
| **Container memory limit** | Process killed with exit code 137 ("OOMKilled") although heap is fine | Heap is only part of memory; see below |
| **CPU limit** | `availableProcessors()` smaller → pool and GC thread counts differ, slow startup | Know the quota; set `-XX:ActiveProcessorCount` if needed; size pools explicitly |
| **Data/config/env vars/time** | Missing config, different feature flags | Config as code, same artifact promoted across environments |

**Container memory - the key knowledge:** modern JVMs are container-aware (Java 8u191+, 10+) and size the heap from the cgroup limit. The default max heap is **25 % of container memory** (`-XX:MaxRAMPercentage=25`), often too small; setting it too high leaves no room for the rest. The JVM uses memory beyond the heap too (metaspace, thread stacks, code cache, direct buffers, GC structures), so the container limit must be **heap + non-heap + headroom**.
```bash
-XX:MaxRAMPercentage=70.0 -XX:InitialRAMPercentage=50.0      # example for a 2 GB pod -> ~1.4 GB heap
# or explicit: -Xms1g -Xmx1g   (then keep the limit comfortably above 1g, e.g. 1.5-2g)
-XX:+ExitOnOutOfMemoryError -XX:+HeapDumpOnOutOfMemoryError
```
Choose the percentage from measured non-heap usage (`-XX:NativeMemoryTracking=summary`), not a guess.

**Prevention:** run CI in the same base image as production; log the JVM/environment facts at startup; tests with `-Duser.timezone`/locale/encoding variations; load-test inside a container with production limits.

**Pitfalls / follow-up probes:**
- Heap OOME (`Java heap space`, JVM-reported) vs **OOMKilled** (kernel-reported, no Java stack trace) - different problems.
- Cgroup v1 vs v2 differences matter for old JDK builds - use a recent JDK patch release.
- A very low CPU limit with many GC/compiler threads → throttling and latency spikes.

**Remember:** *Make JDK, charset, timezone, locale and memory/CPU limits explicit; the container limit = heap + non-heap + headroom.*

---

### Q264. How do you take a heap dump and thread dump safely in production?

**Situation:** You must capture evidence from a live, misbehaving production JVM without making things worse or leaking data.

**Say this first:** "Thread dumps are cheap and safe; heap dumps are heavy and sensitive. I take the light evidence first, protect the service, and handle the dump like production data."

**Thread dump (cheap - take 3, a few seconds apart):**
```bash
jcmd <pid> Thread.print -l > threads-$(date +%s).txt     # preferred
jstack -l <pid> > threads.txt                            # same idea
kill -3 <pid>                                            # writes to the JVM's stdout/log; does NOT kill it
```
It briefly reaches a safepoint (pauses all threads for milliseconds to a second with thousands of threads) - fine on a hung app, still avoid doing it in a tight loop. Also available programmatically via `ThreadMXBean.dumpAllThreads`.

**Heap dump (heavy - plan it):**
```bash
jcmd <pid> GC.heap_dump /dumps/heap.hprof                # dumps live objects only; runs a full GC first
jcmd <pid> GC.heap_dump -all /dumps/heap.hprof           # include unreachable objects (no GC trigger)
jmap -dump:live,format=b,file=/dumps/heap.hprof <pid>    # older equivalent
# automatic on crash:
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/dumps
```
**Safety checklist:**
1. **It pauses the application** for seconds to minutes (proportional to heap size, e.g. a 16 GB heap) - so first **remove one instance from the load balancer** (or pick a canary/replica), then dump. Never do it on the only instance without accepting an outage.
2. **Disk space:** the file is about as big as the used heap; write to a volume with enough space *that is not the application's data/log volume* (a full disk will take the service down). In containers, mount a volume or the dump disappears with the pod.
3. **Run as the same OS user** as the JVM (the attach mechanism requires it), with the same JDK version; `jcmd` (JDK) isn't in "JRE-only"/distroless images - plan for debug tooling (e.g. a debug sidecar/ephemeral container sharing the process namespace, or JFR enabled by startup flags).
4. **Data protection:** a heap dump contains **everything in memory** - passwords, tokens, customer data. Store it encrypted with restricted access, transfer securely, delete after analysis, comply with data policies; consider analysing in place.
5. Compress before transfer (`gzip`); analyse on a machine with RAM at least the dump size (MAT needs a big `-Xmx` setting in `MemoryAnalyzer.ini`).
6. Record the context: time, version, load, what you saw (metrics) - a dump without context is hard to interpret.

**Lighter alternatives first:** `jcmd <pid> GC.class_histogram` (counts per class; triggers a full GC unless `-all`), `jstat -gcutil`, and **JFR** (`jcmd <pid> JFR.start duration=60s filename=/dumps/x.jfr settings=profile`) - low overhead (typically a few percent), great for CPU, allocation, locks and GC without a full dump.

**Prevention:** pre-configure `HeapDumpOnOutOfMemoryError` and a dump path with space; document a runbook ("how to get a thread dump in our pods"); verify tooling works in staging *before* an incident; keep a JFR-capable JDK in images.

**Pitfalls / follow-up probes:**
- `jmap -histo:live` and `GC.heap_dump` both force a full GC (a pause) - not harmless on a huge heap.
- A hung JVM may not respond to the attach API; `kill -3` or a core dump (`gcore`) is the fallback, and `jhsdb` can analyse core files.
- Thread dumps from a *containerised* JVM: `kill -3` output goes to the container's stdout/log stream.

**Remember:** *Thread dumps ×3 are cheap; heap dumps pause, fill disks and hold secrets - drain the instance, mind disk, protect the file.*

---

### Q265. Which JVM metrics do you watch, and how do you roll out, roll back and write a post-mortem?

**Situation:** You are on call for a Java service; you need early warning, a safe release process, and a way to learn from incidents.

**Say this first:** "I watch *saturation and errors*, not just CPU. The key JVM signals are heap *after GC*, GC time, threads, pool queues, and latency percentiles."

**Metrics to watch (and the failure each one predicts):**

| Metric | Healthy shape | What an anomaly means |
|---|---|---|
| **Heap used after GC** (old gen) | Flat floor | Rising floor → leak (see the gradual-slowdown question) |
| **GC pause time & frequency / % time in GC** | Short, rare pauses, GC < ~5% time | Long pauses → latency; high GC% → heap too small |
| **Allocation rate** | Stable | Jump → new inefficient code path |
| **Thread count** (live, blocked) | Stable | Climb → thread leak; many `BLOCKED` → contention |
| **Pool active/queue size/rejections** | Queue near 0 | Growing queue → overload (see unbounded-queue question) |
| **CPU (process vs system), load, throttling** | Headroom | 100% → hot loop/GC; throttled → limit too low |
| **Open FDs vs limit, connection-pool usage** | Far below limit | Leak |
| **Metaspace / direct buffers / non-heap** | Plateau | Growth → class/buffer leak |
| **Latency p50/p95/p99, error rate, throughput** | Within SLO | The user-facing truth; alert on these first |

They come from standard **JMX MBeans** (`MemoryMXBean`, `GarbageCollectorMXBean`, `ThreadMXBean`, `OperatingSystemMXBean`, `BufferPoolMXBean`) and can be exported to your monitoring system (metrics libraries/JMX exporters). Use the **RED** (rate, errors, duration) method for services and **USE** (utilisation, saturation, errors) for resources. Alert on symptoms (SLO burn, p99, errors) and use resource metrics for diagnosis, to avoid alert fatigue.

**Safe rollout / rollback:**
- Build **one immutable artifact**, promote the same one through environments; keep the previous version ready.
- **Canary / gradual rollout** (1% → 10% → 100%) or **blue-green**; compare canary vs baseline (errors, latency, GC, memory) with automatic halt criteria.
- **Feature flags** to decouple deploy from release and to switch off a bad feature without redeploying.
- **Backward-compatible changes**: database migrations in expand → migrate → contract steps so old and new versions can run together and a rollback never meets a schema it can't read; tolerant readers for message/API formats.
- Rollback must be **rehearsed**, fast and not dependent on a rebuild; define rollback triggers before the deploy. Roll **forward** only when the fix is tiny and well tested.
- Graceful shutdown (stop accepting, drain in-flight work, `shutdown()` pools, close resources) so deploys don't drop requests; readiness vs liveness checks.
- Deploy away from peak hours; have an owner on call.

**Post-mortem (blameless):**
1. **Summary & impact** (who, how long, how much - numbers).
2. **Timeline** with timestamps (detect, diagnose, mitigate, resolve).
3. **Root cause and contributing factors** ("5 whys" - systems and process, not "Bob made a mistake").
4. **Detection:** how did we find out - alert or customer? Why not sooner?
5. **What went well / badly / lucky.**
6. **Action items** - each with an owner, priority and due date (fix, detect, prevent, runbook), tracked to closure.
Share it widely; the goal is learning and preventing repeats.

**Pitfalls / follow-up probes:**
- Mitigate first (rollback, flag off, scale out), investigate second - restore service before hunting the root cause, but **save evidence** (dumps, logs) first if it costs little.
- Averages and CPU alone miss most JVM problems (e.g. a leak with 20% CPU).
- Alerts with no runbook are noise.

**Remember:** *Watch heap-after-GC, GC time, threads, queues, CPU and p99; ship canary + flags + compatible migrations with a rehearsed rollback; write blameless post-mortems with owned actions.*

---

# 11. Java Coding Round — Problems You Will Be Asked

### Q266. How should you approach any coding question in an interview?

**Answer:**
The interviewer is judging your *thinking*, not only the final code. Follow the same seven steps every time:

| Step | What you do | Example line you can say |
|---|---|---|
| 1. **Clarify** | Ask about input size, nulls, duplicates, negative numbers, sorted or not, case-sensitivity, expected output on invalid input | "Can the array be empty? Are values only positive? Is memory a constraint?" |
| 2. **Examples** | Write 2–3 small examples, including one edge case | "For `[1,2,3]` the answer is X; for `[]` it is Y." |
| 3. **Brute force** | State the simple solution and its cost, even if you will not code it | "Two nested loops, O(n²). Works but too slow for 10⁶ items." |
| 4. **Optimise** | Look for the trick: hash map, two pointers, sorting, sliding window, stack, bit tricks | "A `HashMap` removes the inner loop → O(n) time, O(n) space." |
| 5. **Code** | Write clean code with meaningful names; talk while you type | |
| 6. **Test** | Dry-run your own code with the example and one edge case before the interviewer asks | "Let me trace `[3,3]`…" |
| 7. **Complexity** | Say time **and** space; mention trade-offs | "O(n) time, O(n) extra space; I can do O(1) space if sorting is allowed." |

**Senior-level add-ons:**
- Use the standard library when the question allows (`Collections`, `Arrays`, streams) but be ready to write it by hand when they say "without using built-in methods".
- Mention thread-safety only when it matters (shared state, caches, counters).
- Mention overflow (`int` vs `long`), `null` handling and `Integer` caching traps (`==` vs `equals`) — these small catches separate a 9-year developer from a fresher.
- If you are stuck, say the brute force out loud and improve it; silence is the worst answer.

**Remember:** **C-E-B-O-C-T-C** = *Clarify, Examples, Brute force, Optimise, Code, Test, Complexity.*

---

### Q267. Reverse a string, and reverse the words of a sentence.

**Problem:** Given `"hello"` return `"olleh"`. Given `"the sky is blue"` return `"blue is sky the"`. Also reverse each word in place: `"olleh dlrow"` for `"hello world"`.

**Approach:**
- A `String` is immutable, so copy it into a `char[]`. Put one pointer at the start and one at the end, swap, and move both towards the middle until they meet.
- For words: split on one-or-more spaces (`\\s+`), reverse the array of words, and join with a single space.
- For "reverse each word": split, reverse every word with `StringBuilder.reverse()`, join back.

```java
import java.util.*;
import java.util.stream.*;

class ReverseDemo {
    static String reverse(String s) {
        if (s == null) return null;
        char[] c = s.toCharArray();
        for (int i = 0, j = c.length - 1; i < j; i++, j--) {
            char t = c[i]; c[i] = c[j]; c[j] = t;
        }
        return new String(c);
    }

    static String reverseWords(String s) {
        List<String> words = Arrays.asList(s.trim().split("\\s+"));
        Collections.reverse(words);                       // reverses the backing array
        return String.join(" ", words);
    }

    static String reverseEachWord(String s) {
        return Arrays.stream(s.trim().split("\\s+"))
                .map(w -> new StringBuilder(w).reverse().toString())
                .collect(Collectors.joining(" "));
    }

    static String reverseRecursive(String s) {           // shown only to explain; O(n²) copies and deep stack
        return s.isEmpty() ? s : reverseRecursive(s.substring(1)) + s.charAt(0);
    }

    public static void main(String[] args) {
        System.out.println(reverse("hello"));                    // olleh
        System.out.println(reverseWords("  the sky  is blue ")); // blue is sky the
        System.out.println(reverseEachWord("hello world"));      // olleh dlrow
    }
}
```

**Complexity:** Time O(n), space O(n) (a new string/array is needed because `String` is immutable). The recursive version is O(n²) and can overflow the stack.

**Edge cases:** `null`, empty string, single character, multiple/leading/trailing spaces, Unicode surrogate pairs (emoji). `StringBuilder.reverse()` handles surrogate pairs correctly; the manual `char` swap does **not**.

**Follow-up:** "Reverse a string without extra space?" → impossible for `String` (immutable); possible for `char[]` using the two-pointer swap. "Reverse words in a `char[]` in place?" → reverse the whole array, then reverse each word.

---

### Q268. Check whether a string or a number is a palindrome.

**Problem:** `"A man, a plan, a canal: Panama"` is a palindrome (ignore case and non-alphanumerics). `121` is a palindrome number, `-121` and `10` are not. Do the number check **without converting to a String**.

**Approach:**
- *String:* two pointers from both ends. Skip characters that are not letters/digits and compare the rest in lower case.
- *Number:* negative numbers and numbers ending with 0 (except 0 itself) can never be palindromes. Otherwise reverse only the **second half** of the digits and compare it with the first half. Reversing only half also avoids integer overflow.

```java
import java.util.stream.IntStream;

class PalindromeDemo {
    static boolean isPalindrome(String s) {
        int i = 0, j = s.length() - 1;
        while (i < j) {
            while (i < j && !Character.isLetterOrDigit(s.charAt(i))) i++;
            while (i < j && !Character.isLetterOrDigit(s.charAt(j))) j--;
            if (Character.toLowerCase(s.charAt(i++)) != Character.toLowerCase(s.charAt(j--))) return false;
        }
        return true;
    }

    // Simple stream variant for a plain (already cleaned) string
    static boolean isPalindromeStream(String s) {
        return IntStream.range(0, s.length() / 2)
                .allMatch(i -> s.charAt(i) == s.charAt(s.length() - 1 - i));
    }

    static boolean isPalindrome(int n) {
        if (n < 0 || (n % 10 == 0 && n != 0)) return false;
        int reversedHalf = 0;
        while (n > reversedHalf) {
            reversedHalf = reversedHalf * 10 + n % 10;
            n /= 10;
        }
        return n == reversedHalf || n == reversedHalf / 10;   // even / odd digit count
    }

    public static void main(String[] args) {
        System.out.println(isPalindrome("A man, a plan, a canal: Panama")); // true
        System.out.println(isPalindrome(12321));                           // true
        System.out.println(isPalindrome(10));                              // false
    }
}
```

**Complexity:** String O(n) time, O(1) space. Number O(log₁₀ n) time, O(1) space.

**Edge cases:** empty string (true), only punctuation (true), `0` (true), `Integer.MAX_VALUE`-sized values (half-reverse avoids overflow), negative numbers.

**Follow-up:** "Longest palindromic substring?" → expand around each centre, O(n²). "Can you make it a palindrome by deleting one character?" → on first mismatch try skipping left or right char.

---

### Q269. Check whether two strings are anagrams.

**Problem:** `"listen"` and `"silent"` are anagrams — same letters, same counts, different order.

**Approach:**
- If lengths differ → not anagrams.
- Count letters of the first string in a map; for the second string decrement the counts. If any count is missing or goes below zero, they are not anagrams.
- Alternative: sort both character arrays and compare (simpler, O(n log n)).

```java
import java.util.*;

class AnagramDemo {
    static boolean isAnagram(String a, String b) {
        if (a == null || b == null || a.length() != b.length()) return false;
        Map<Character, Integer> count = new HashMap<>();
        for (char c : a.toCharArray()) count.merge(c, 1, Integer::sum);
        for (char c : b.toCharArray()) {
            Integer n = count.get(c);
            if (n == null || n == 0) return false;
            count.put(c, n - 1);
        }
        return true;
    }

    static boolean isAnagramSorted(String a, String b) {
        char[] x = a.toCharArray(), y = b.toCharArray();
        Arrays.sort(x);
        Arrays.sort(y);
        return Arrays.equals(x, y);
    }

    // For phrases: normalise first
    static boolean isPhraseAnagram(String a, String b) {
        return isAnagramSorted(a.replaceAll("\\s+", "").toLowerCase(),
                               b.replaceAll("\\s+", "").toLowerCase());
    }

    public static void main(String[] args) {
        System.out.println(isAnagram("listen", "silent"));          // true
        System.out.println(isAnagramSorted("rat", "car"));          // false
        System.out.println(isPhraseAnagram("Dormitory", "Dirty room")); // true
    }
}
```

**Complexity:** Map version O(n) time, O(k) space (k = distinct characters). Sorting version O(n log n) time.

**Edge cases:** different lengths, `null`, upper/lower case, spaces and punctuation, Unicode. For lower-case `a–z` only, a `int[26]` array is faster than a map.

**Follow-up:** "Group a list of words into anagram groups" → see the *Group Anagrams* problem below.

---

### Q270. Find the first non-repeating character in a string.

**Problem:** `"swiss"` → `'w'`. `"aabb"` → none.

**Approach:**
- Walk the string once and count every character in a **`LinkedHashMap`** — it keeps insertion order.
- Then pick the first entry whose count is 1. Order of insertion equals order of first appearance, so that entry is the answer.
- Stream version does exactly the same with `groupingBy(..., LinkedHashMap::new, counting())`.

```java
import java.util.*;
import java.util.function.Function;
import java.util.stream.*;

class FirstUniqueDemo {
    static Character firstUnique(String s) {
        Map<Character, Integer> count = new LinkedHashMap<>();
        for (char c : s.toCharArray()) count.merge(c, 1, Integer::sum);
        for (Map.Entry<Character, Integer> e : count.entrySet())
            if (e.getValue() == 1) return e.getKey();
        return null;
    }

    static Character firstUniqueStream(String s) {
        return s.chars().mapToObj(c -> (char) c)
                .collect(Collectors.groupingBy(Function.identity(), LinkedHashMap::new, Collectors.counting()))
                .entrySet().stream()
                .filter(e -> e.getValue() == 1)
                .map(Map.Entry::getKey)
                .findFirst()
                .orElse(null);
    }

    public static void main(String[] args) {
        System.out.println(firstUnique("swiss"));        // w
        System.out.println(firstUniqueStream("aabb"));   // null
    }
}
```

**Complexity:** O(n) time, O(k) space (k = distinct characters).

**Edge cases:** all characters repeat, empty string, case sensitivity (`'S'` vs `'s'`), return type should be `Character`/`OptionalInt`/index, not `char`, so "none" can be expressed.

**Follow-up:** "Do it on a stream of characters you cannot re-read?" → keep a `LinkedHashSet` of candidates plus a `HashSet` of seen-twice. "Return the index?" → two passes with an `int[]` count array.

---

### Q271. Character frequency (map and stream) and counting vowels.

**Problem:** For `"programming"` print each character with its count. Count vowels in a sentence.

**Approach:**
- `Map.merge(key, 1, Integer::sum)` is the cleanest "increment or insert" idiom.
- With streams, convert the string to a `Stream<Character>` and use `groupingBy(identity, counting())`. Remember `String.chars()` returns an `IntStream` of code points, so map to `char` first.
- Vowel count: lower-case the text and filter characters that exist in `"aeiou"`.

```java
import java.util.*;
import java.util.function.Function;
import java.util.stream.*;

class FrequencyDemo {
    static Map<Character, Integer> frequency(String s) {
        Map<Character, Integer> m = new LinkedHashMap<>();
        for (char c : s.toCharArray()) m.merge(c, 1, Integer::sum);
        return m;
    }

    static Map<Character, Long> frequencyStream(String s) {
        return s.chars().mapToObj(c -> (char) c)
                .collect(Collectors.groupingBy(Function.identity(), LinkedHashMap::new, Collectors.counting()));
    }

    static long countVowels(String s) {
        return s.toLowerCase().chars().filter(c -> "aeiou".indexOf(c) >= 0).count();
    }

    static int countVowelsLoop(String s) {
        int n = 0;
        for (char c : s.toLowerCase().toCharArray())
            if (c == 'a' || c == 'e' || c == 'i' || c == 'o' || c == 'u') n++;
        return n;
    }

    public static void main(String[] args) {
        System.out.println(frequency("programming"));        // {p=1, r=2, o=1, g=2, a=1, m=2, i=1, n=1}
        System.out.println(frequencyStream("banana"));       // {b=1, a=3, n=2}
        System.out.println(countVowels("Hello World"));      // 3
    }
}
```

**Complexity:** O(n) time, O(k) space.

**Edge cases:** `null`/empty input, upper-case vowels, spaces and punctuation counted as characters (filter with `Character::isLetter` if needed), `HashMap` gives no order — use `LinkedHashMap` (insertion order) or `TreeMap` (sorted).

**Follow-up:** "Print characters sorted by frequency descending" → sort the entry set with `Map.Entry.comparingByValue(Comparator.reverseOrder())`.

---

### Q272. Remove duplicates from an array or list while preserving order.

**Problem:** `[3,1,3,2,1]` → `[3,1,2]`. Also: remove duplicates **in place** from a *sorted* array and return the new length.

**Approach:**
- *Unsorted, keep order:* a `LinkedHashSet` remembers first-seen order and drops repeats. Streams: `distinct()` does the same (it is order-preserving on ordered streams).
- *Sorted array in place:* keep a write index `k`. A value is new only when it differs from the last written value `a[k-1]`; if so copy it to `a[k]` and advance `k`.

```java
import java.util.*;
import java.util.stream.*;

class RemoveDuplicatesDemo {
    static <T> List<T> dedupe(List<T> list) {
        return new ArrayList<>(new LinkedHashSet<>(list));
    }

    static <T> List<T> dedupeStream(List<T> list) {
        return list.stream().distinct().collect(Collectors.toList());
    }

    static int[] dedupe(int[] a) {
        return Arrays.stream(a).distinct().toArray();
    }

    // sorted array, in place, O(1) extra space
    static int removeDuplicatesSorted(int[] a) {
        if (a.length == 0) return 0;
        int k = 1;
        for (int i = 1; i < a.length; i++) {
            if (a[i] != a[k - 1]) a[k++] = a[i];
        }
        return k;                                   // first k elements are unique
    }

    public static void main(String[] args) {
        System.out.println(dedupe(List.of(3, 1, 3, 2, 1)));                    // [3, 1, 2]
        System.out.println(Arrays.toString(dedupe(new int[]{3, 1, 3, 2, 1}))); // [3, 1, 2]
        int[] sorted = {1, 1, 2, 2, 2, 3};
        int k = removeDuplicatesSorted(sorted);
        System.out.println(Arrays.toString(Arrays.copyOf(sorted, k)));        // [1, 2, 3]
    }
}
```

**Complexity:** Set/stream: O(n) time, O(n) space. Sorted in-place: O(n) time, O(1) space.

**Edge cases:** empty input, all equal, no duplicates, `null` elements (`LinkedHashSet` allows one `null`; `List.of` does not allow any). Elements need correct `equals`/`hashCode`.

**Follow-up:** "Why not `HashSet`?" → it loses order. "What if the objects should be unique by one field only?" → `toMap(Employee::id, e -> e, (a, b) -> a, LinkedHashMap::new).values()`.

---

### Q273. Find duplicate elements in an array or list.

**Problem:** `[4,3,2,7,8,2,3,1]` → duplicates `{2,3}`.

**Approach:**
- `Set.add()` returns `false` when the element already exists. Use one `Set` to track "seen" and another for "duplicates" — a single pass.
- Streams: group by value, count, keep counts > 1. (Avoid `Collections.frequency` inside a loop — that is O(n²).)

```java
import java.util.*;
import java.util.function.Function;
import java.util.stream.*;

class FindDuplicatesDemo {
    static <T> Set<T> findDuplicates(Collection<T> items) {
        Set<T> seen = new HashSet<>();
        Set<T> dups = new LinkedHashSet<>();
        for (T t : items) {
            if (!seen.add(t)) dups.add(t);
        }
        return dups;
    }

    static <T> Set<T> findDuplicatesStream(Collection<T> items) {
        return items.stream()
                .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()))
                .entrySet().stream()
                .filter(e -> e.getValue() > 1)
                .map(Map.Entry::getKey)
                .collect(Collectors.toSet());
    }

    // Values are in 1..n: O(1) extra space by marking visited indexes negative (mutates input)
    static List<Integer> findDuplicatesInRange(int[] a) {
        List<Integer> res = new ArrayList<>();
        for (int x : a) {
            int idx = Math.abs(x) - 1;
            if (a[idx] < 0) res.add(Math.abs(x));
            else a[idx] = -a[idx];
        }
        return res;
    }

    public static void main(String[] args) {
        System.out.println(findDuplicates(List.of(4, 3, 2, 7, 8, 2, 3, 1)));        // [2, 3]
        System.out.println(findDuplicatesStream(List.of("a", "b", "a")));          // [a]
        System.out.println(findDuplicatesInRange(new int[]{4, 3, 2, 7, 8, 2, 3, 1})); // [2, 3]
    }
}
```

**Complexity:** Set approach O(n) time, O(n) space. Index-marking approach O(n) time, O(1) extra space but destroys the input.

**Edge cases:** empty collection, element appearing 3+ times (reported once), `null` elements.

**Follow-up:** "Array of n+1 numbers in 1..n with exactly one duplicate, no modification, O(1) space?" → Floyd's cycle detection on the index graph.

---

### Q274. Find the second highest (Nth highest) number, and the Nth highest salary using streams.

**Problem:** From `[12, 35, 1, 10, 34, 1]` the second highest *distinct* value is `34`. From a list of employees return the employee(s) with the 3rd highest salary.

**Approach:**
- *Second highest in one pass:* track `max` and `second`. If a number is greater than `max`, old `max` becomes `second`. Else if it is greater than `second` and **not equal** to `max`, update `second`.
- *Nth highest, general:* keep a **min-heap of size N** (`PriorityQueue`). The root is always the Nth largest. Or use a `TreeSet` (sorted, distinct).
- *Salary with streams:* group employees by salary into a `TreeMap` sorted in reverse, skip `n-1` groups, take the next.

```java
import java.util.*;
import java.util.stream.*;

class NthHighestDemo {
    record Employee(int id, String name, String dept, double salary) {}

    static Integer secondHighest(int[] a) {
        Integer max = null, second = null;
        for (int x : a) {
            if (max == null || x > max) { second = max; max = x; }
            else if (x != max && (second == null || x > second)) second = x;
        }
        return second;                                   // null when there is no second distinct value
    }

    static Integer nthHighestDistinct(int[] a, int n) {
        TreeSet<Integer> top = new TreeSet<>();          // keeps the n largest distinct values
        for (int x : a) {
            top.add(x);
            if (top.size() > n) top.pollFirst();
        }
        return top.size() == n ? top.first() : null;
    }

    static Optional<Integer> nthHighestStream(int[] a, int n) {
        return Arrays.stream(a).boxed().distinct()
                .sorted(Comparator.reverseOrder())
                .skip(n - 1).findFirst();
    }

    static Optional<List<Employee>> nthHighestSalary(List<Employee> emps, int n) {
        return emps.stream()
                .collect(Collectors.groupingBy(Employee::salary,
                        () -> new TreeMap<Double, List<Employee>>(Comparator.reverseOrder()),
                        Collectors.toList()))
                .values().stream()
                .skip(n - 1).findFirst();
    }

    public static void main(String[] args) {
        int[] a = {12, 35, 1, 10, 34, 1};
        System.out.println(secondHighest(a));            // 34
        System.out.println(nthHighestDistinct(a, 3));    // 12
        System.out.println(nthHighestStream(a, 2));      // Optional[34]

        List<Employee> emps = List.of(
                new Employee(1, "Asha", "IT", 90000), new Employee(2, "Ravi", "IT", 120000),
                new Employee(3, "Meena", "HR", 90000), new Employee(4, "John", "HR", 70000));
        System.out.println(nthHighestSalary(emps, 2));   // the two employees earning 90000
    }
}
```

**Complexity:** Single pass O(n) for 2nd highest; TreeSet/heap O(n log N); sorted-stream O(n log n).

**Edge cases:** fewer than N distinct values (return empty/null, do not throw `IndexOutOfBounds`), duplicates of the max value, all values equal, negative numbers, `Integer.MIN_VALUE` as a legitimate value (this is why `Integer` null is used instead of a sentinel).

**Follow-up:** "Nth largest in an unsorted array in O(n) average?" → Quickselect. "Top-K from a huge stream?" → size-K min-heap.

---

### Q275. Find the missing number in an array containing 0..n.

**Problem:** `[3,0,1]` (n = 3) → missing `2`.

**Approach:**
- The sum of `0..n` is `n(n+1)/2`. Subtract the actual sum; the difference is the missing number. Use `long` so the sum cannot overflow.
- XOR alternative: XOR of all indexes and all values cancels every present number, leaving the missing one. No overflow at all.

```java
import java.util.stream.IntStream;

class MissingNumberDemo {
    static int missingBySum(int[] a) {
        long n = a.length;
        long expected = n * (n + 1) / 2;
        long actual = 0;
        for (int x : a) actual += x;
        return (int) (expected - actual);
    }

    static int missingByXor(int[] a) {
        int xor = a.length;                  // start with n because indexes only reach n-1
        for (int i = 0; i < a.length; i++) xor ^= i ^ a[i];
        return xor;
    }

    static int missingStream(int[] a) {
        int n = a.length;
        return (int) ((long) n * (n + 1) / 2 - IntStream.of(a).asLongStream().sum());
    }

    public static void main(String[] args) {
        System.out.println(missingBySum(new int[]{3, 0, 1}));              // 2
        System.out.println(missingByXor(new int[]{9, 6, 4, 2, 3, 5, 7, 0, 1})); // 8
        System.out.println(missingStream(new int[]{0, 1}));                // 2
    }
}
```

**Complexity:** O(n) time, O(1) space.

**Edge cases:** missing `0`, missing `n`, single element array, large `n` (sum overflow in `int`).

**Follow-up:** "Two numbers missing?" → use sum and sum of squares, or XOR then split by a set bit. "Array is sorted?" → binary search on `a[i] != i`.

---

### Q276. Two Sum — find two indexes whose values add up to a target.

**Problem:** `nums = [2,7,11,15]`, `target = 9` → `[0,1]`. Exactly one solution exists; do not use the same element twice.

**Approach:**
- Brute force checks every pair, O(n²).
- Better: while scanning, store `value → index` in a `HashMap`. For each number `x`, the partner we need is `target - x`. If it is already in the map, we are done; otherwise store `x`. One pass.

```java
import java.util.*;

class TwoSumDemo {
    static int[] twoSum(int[] nums, int target) {
        Map<Integer, Integer> seen = new HashMap<>();      // value -> index
        for (int i = 0; i < nums.length; i++) {
            Integer j = seen.get(target - nums[i]);
            if (j != null) return new int[]{j, i};
            seen.put(nums[i], i);
        }
        throw new IllegalArgumentException("No two sum solution");
    }

    // If the array is sorted and only the values/boolean are needed: O(1) space
    static boolean twoSumSorted(int[] a, int target) {
        int l = 0, r = a.length - 1;
        while (l < r) {
            long s = (long) a[l] + a[r];
            if (s == target) return true;
            if (s < target) l++; else r--;
        }
        return false;
    }

    public static void main(String[] args) {
        System.out.println(Arrays.toString(twoSum(new int[]{2, 7, 11, 15}, 9)));  // [0, 1]
        System.out.println(Arrays.toString(twoSum(new int[]{3, 3}, 6)));          // [0, 1]
        System.out.println(twoSumSorted(new int[]{1, 2, 4, 9}, 8));               // false
    }
}
```

**Complexity:** HashMap O(n) time, O(n) space. Sorted two-pointer O(n) time, O(1) space.

**Edge cases:** duplicates (`[3,3]`, target 6 — checking the map **before** inserting handles it), negative numbers, `target - x` overflow (use `long` for extreme values), no solution (throw or return empty).

**Follow-up:** "Return all unique pairs?" → sort + two pointers skipping duplicates. "Input is a stream?" → the hash map approach works online.

---

### Q277. Three Sum — all unique triplets that add up to zero.

**Problem:** `[-1,0,1,2,-1,-4]` → `[[-1,-1,2],[-1,0,1]]`. The result must not contain duplicate triplets.

**Approach:**
- Sort the array. Fix the first number `a[i]`; then the problem becomes "two sum = −a[i]" on the remaining right part, solved with two pointers `l` and `r`.
- To avoid duplicate triplets, skip an `i` equal to the previous `i`, and after finding a triplet skip repeated values on both sides.
- If `a[i] > 0`, no triplet is possible (all later numbers are bigger) → stop early.

```java
import java.util.*;

class ThreeSumDemo {
    static List<List<Integer>> threeSum(int[] input) {
        int[] a = input.clone();                       // do not mutate the caller's array
        Arrays.sort(a);
        List<List<Integer>> res = new ArrayList<>();
        for (int i = 0; i < a.length - 2; i++) {
            if (a[i] > 0) break;
            if (i > 0 && a[i] == a[i - 1]) continue;
            int l = i + 1, r = a.length - 1;
            while (l < r) {
                int sum = a[i] + a[l] + a[r];
                if (sum < 0) l++;
                else if (sum > 0) r--;
                else {
                    res.add(List.of(a[i], a[l], a[r]));
                    l++; r--;
                    while (l < r && a[l] == a[l - 1]) l++;
                    while (l < r && a[r] == a[r + 1]) r--;
                }
            }
        }
        return res;
    }

    public static void main(String[] args) {
        System.out.println(threeSum(new int[]{-1, 0, 1, 2, -1, -4}));  // [[-1, -1, 2], [-1, 0, 1]]
        System.out.println(threeSum(new int[]{0, 0, 0, 0}));           // [[0, 0, 0]]
    }
}
```

**Complexity:** O(n²) time (sort is O(n log n)), O(1) extra space apart from the output (O(log n) for sort).

**Edge cases:** fewer than 3 elements, all zeros, many duplicates, large values causing `int` overflow in the sum (use `long`).

**Follow-up:** "3Sum closest to target?" → same loop, track the smallest `|sum - target|`. "4Sum?" → one more outer loop, O(n³).

---

### Q278. Maximum subarray sum (Kadane's algorithm).

**Problem:** `[-2,1,-3,4,-1,2,1,-5,4]` → `6` (subarray `[4,-1,2,1]`).

**Approach:**
- Walk once, keeping `current` = best sum of a subarray **ending at this index**. At every element choose: extend the previous subarray (`current + x`) or start fresh from `x`. Whichever is bigger wins.
- Keep the global best across all positions.
- Start from `a[0]` (not 0) so all-negative arrays return the least negative number.

```java
class KadaneDemo {
    static int maxSubarray(int[] a) {
        int current = a[0], best = a[0];
        for (int i = 1; i < a.length; i++) {
            current = Math.max(a[i], current + a[i]);
            best = Math.max(best, current);
        }
        return best;
    }

    // Also return the range
    static int[] maxSubarrayRange(int[] a) {             // {sum, start, end}
        int current = a[0], best = a[0], start = 0, bestStart = 0, bestEnd = 0;
        for (int i = 1; i < a.length; i++) {
            if (a[i] > current + a[i]) { current = a[i]; start = i; }
            else current += a[i];
            if (current > best) { best = current; bestStart = start; bestEnd = i; }
        }
        return new int[]{best, bestStart, bestEnd};
    }

    public static void main(String[] args) {
        System.out.println(maxSubarray(new int[]{-2, 1, -3, 4, -1, 2, 1, -5, 4})); // 6
        System.out.println(maxSubarray(new int[]{-3, -1, -2}));                    // -1
    }
}
```

**Complexity:** O(n) time, O(1) space.

**Edge cases:** all negatives, single element, empty array (decide: throw or return 0 — ask the interviewer), overflow on huge inputs (use `long`).

**Follow-up:** "Maximum product subarray?" → track both max and min (a negative can flip them). "Circular array?" → max(Kadane, total − min subarray).

---

### Q279. Move all zeros to the end, keeping the order of the other elements.

**Problem:** `[0,1,0,3,12]` → `[1,3,12,0,0]` in place.

**Approach:**
- Use a write pointer `pos`. Scan with `i`; whenever `a[i]` is non-zero copy it to `a[pos]` and advance `pos`.
- After the scan, fill everything from `pos` to the end with zeros. (Or swap instead of copy to do it in one go.)

```java
import java.util.*;
import java.util.stream.*;

class MoveZerosDemo {
    static void moveZeros(int[] a) {
        int pos = 0;
        for (int x : a) {
            if (x != 0) a[pos++] = x;
        }
        while (pos < a.length) a[pos++] = 0;
    }

    // Stream variant (creates a new array)
    static int[] moveZerosStream(int[] a) {
        return IntStream.concat(
                Arrays.stream(a).filter(x -> x != 0),
                Arrays.stream(a).filter(x -> x == 0)).toArray();
    }

    public static void main(String[] args) {
        int[] a = {0, 1, 0, 3, 12};
        moveZeros(a);
        System.out.println(Arrays.toString(a));                                         // [1, 3, 12, 0, 0]
        System.out.println(Arrays.toString(moveZerosStream(new int[]{0, 0, 5, 0, 7}))); // [5, 7, 0, 0, 0]
    }
}
```

**Complexity:** O(n) time, O(1) space (stream version O(n) space).

**Edge cases:** no zeros, all zeros, empty array, single element. The order of non-zero elements must stay stable — a simple "sort by zero" would break that.

**Follow-up:** "Minimise the number of writes?" → swap only when `i != pos`. "Move a given value `k` to the end?" → same code with `x != k`.

---

### Q280. Rotate an array to the right by k steps.

**Problem:** `[1,2,3,4,5,6,7]`, `k = 3` → `[5,6,7,1,2,3,4]`.

**Approach:**
- Normalise `k = k % n` (rotating by `n` gives the same array).
- Three reversals do it in place: reverse the whole array, then reverse the first `k` elements, then reverse the remaining `n-k` elements.
- Example: `1234567` → `7654321` → `567 4321` → `567 1234`.

```java
import java.util.*;

class RotateArrayDemo {
    static void rotate(int[] a, int k) {
        int n = a.length;
        if (n == 0) return;
        k = ((k % n) + n) % n;               // handles negative k (rotate left) too
        reverse(a, 0, n - 1);
        reverse(a, 0, k - 1);
        reverse(a, k, n - 1);
    }

    private static void reverse(int[] a, int i, int j) {
        while (i < j) {
            int t = a[i]; a[i++] = a[j]; a[j--] = t;
        }
    }

    // Easier but O(n) extra space
    static int[] rotateCopy(int[] a, int k) {
        int n = a.length;
        int[] r = new int[n];
        for (int i = 0; i < n; i++) r[(i + k) % n] = a[i];
        return r;
    }

    public static void main(String[] args) {
        int[] a = {1, 2, 3, 4, 5, 6, 7};
        rotate(a, 3);
        System.out.println(Arrays.toString(a));                             // [5, 6, 7, 1, 2, 3, 4]
        System.out.println(Arrays.toString(rotateCopy(new int[]{1, 2}, 5))); // [2, 1]
    }
}
```

**Complexity:** Reversal method O(n) time, O(1) space. Copy method O(n) time, O(n) space.

**Edge cases:** `k = 0`, `k > n`, `k` negative, empty or single element array.

**Follow-up:** "Rotate a `List`?" → `Collections.rotate(list, k)`. "Rotate a linked list?" → make it circular, then break at `n - k`.

---

### Q281. Merge two sorted arrays and two sorted linked lists.

**Problem:** `[1,3,5]` and `[2,4,6,8]` → `[1,2,3,4,5,6,8]`. Same for two sorted singly linked lists.

**Approach:**
- Keep one pointer in each input. Always take the smaller of the two current elements and move that pointer. When one input ends, copy the rest of the other.
- For linked lists use a **dummy head node** so you never need a special case for the first node, and re-link existing nodes instead of creating new ones.
- Take from the left list on ties (`<=`) to keep the merge **stable**.

```java
import java.util.*;

class MergeSortedDemo {
    static int[] merge(int[] a, int[] b) {
        int[] r = new int[a.length + b.length];
        int i = 0, j = 0, k = 0;
        while (i < a.length && j < b.length) r[k++] = a[i] <= b[j] ? a[i++] : b[j++];
        while (i < a.length) r[k++] = a[i++];
        while (j < b.length) r[k++] = b[j++];
        return r;
    }

    static class ListNode {
        int val; ListNode next;
        ListNode(int val) { this.val = val; }
    }

    static ListNode merge(ListNode a, ListNode b) {
        ListNode dummy = new ListNode(0), tail = dummy;
        while (a != null && b != null) {
            if (a.val <= b.val) { tail.next = a; a = a.next; }
            else { tail.next = b; b = b.next; }
            tail = tail.next;
        }
        tail.next = (a != null) ? a : b;
        return dummy.next;
    }

    static ListNode of(int... v) {
        ListNode dummy = new ListNode(0), t = dummy;
        for (int x : v) { t.next = new ListNode(x); t = t.next; }
        return dummy.next;
    }

    public static void main(String[] args) {
        System.out.println(Arrays.toString(merge(new int[]{1, 3, 5}, new int[]{2, 4, 6, 8}))); // [1, 2, 3, 4, 5, 6, 8]
        for (ListNode n = merge(of(1, 4), of(2, 3, 5)); n != null; n = n.next) System.out.print(n.val + " ");
    }
}
```

**Complexity:** O(m + n) time. Arrays need O(m + n) output space; the linked-list merge is O(1) extra space.

**Edge cases:** one input empty or `null`, all of one array smaller than the other, duplicates across arrays.

**Follow-up:** "Merge `k` sorted lists?" → `PriorityQueue` of heads, O(N log k). "Merge in place into the first array that has spare room at the end?" → fill from the back.

---

### Q282. Binary search — and the common off-by-one bugs.

**Problem:** In a sorted `int[]` return the index of `target`, or `-1`. Then return the index of the **first** occurrence when duplicates exist.

**Approach:**
- Keep a range `[lo, hi]` that *might* contain the target. Look at the middle: if equal, done; if the middle is smaller, the target must be on the right (`lo = mid + 1`); otherwise on the left (`hi = mid - 1`).
- Compute `mid = lo + (hi - lo) / 2`, **not** `(lo + hi) / 2` — the latter overflows for large indexes.
- For "first occurrence", when you find the target do not stop: remember it and keep searching the left half.

```java
import java.util.Arrays;

class BinarySearchDemo {
    static int search(int[] a, int target) {
        int lo = 0, hi = a.length - 1;               // inclusive range
        while (lo <= hi) {                           // <= because a single element is still a valid range
            int mid = lo + (hi - lo) / 2;
            if (a[mid] == target) return mid;
            if (a[mid] < target) lo = mid + 1;       // +1 / -1 are mandatory, else infinite loop
            else hi = mid - 1;
        }
        return -1;
    }

    static int firstOccurrence(int[] a, int target) {
        int lo = 0, hi = a.length - 1, ans = -1;
        while (lo <= hi) {
            int mid = lo + (hi - lo) / 2;
            if (a[mid] == target) { ans = mid; hi = mid - 1; }
            else if (a[mid] < target) lo = mid + 1;
            else hi = mid - 1;
        }
        return ans;
    }

    static int recursive(int[] a, int target, int lo, int hi) {
        if (lo > hi) return -1;
        int mid = lo + (hi - lo) / 2;
        if (a[mid] == target) return mid;
        return a[mid] < target ? recursive(a, target, mid + 1, hi) : recursive(a, target, lo, mid - 1);
    }

    public static void main(String[] args) {
        int[] a = {1, 2, 4, 4, 4, 7, 9};
        System.out.println(search(a, 7));                       // 5
        System.out.println(firstOccurrence(a, 4));              // 2
        System.out.println(recursive(a, 3, 0, a.length - 1));   // -1
        System.out.println(Arrays.binarySearch(a, 9));          // 6 (library version; negative = -(insertionPoint) - 1)
    }
}
```

**Common bugs to name in the interview:**
1. `(lo + hi) / 2` integer overflow.
2. `while (lo < hi)` with an inclusive `hi` → misses the last element.
3. `lo = mid` or `hi = mid` (without ±1) → infinite loop.
4. Running it on an **unsorted** array.
5. With duplicates, `Arrays.binarySearch` may return *any* matching index.

**Complexity:** O(log n) time, O(1) space (recursive: O(log n) stack).

**Edge cases:** empty array, one element, target smaller than all / larger than all, duplicates.

**Follow-up:** "Search in a rotated sorted array?" → at each step one half is always sorted; decide which half to keep. "Square root of `x` without `Math.sqrt`?" → binary search on the answer.

---

### Q283. FizzBuzz.

**Problem:** For 1..n print `Fizz` for multiples of 3, `Buzz` for multiples of 5, `FizzBuzz` for multiples of both, else the number.

**Approach:**
- The only trap is the order: test the "both" case (`i % 15 == 0`) **first**, otherwise `Fizz` or `Buzz` would catch it.
- Streams: `IntStream.rangeClosed(1, n).mapToObj(...)` with a chained ternary.

```java
import java.util.stream.IntStream;

class FizzBuzzDemo {
    static void fizzBuzz(int n) {
        for (int i = 1; i <= n; i++) {
            if (i % 15 == 0) System.out.println("FizzBuzz");
            else if (i % 3 == 0) System.out.println("Fizz");
            else if (i % 5 == 0) System.out.println("Buzz");
            else System.out.println(i);
        }
    }

    static void fizzBuzzStream(int n) {
        IntStream.rangeClosed(1, n)
                .mapToObj(i -> i % 15 == 0 ? "FizzBuzz"
                             : i % 3 == 0 ? "Fizz"
                             : i % 5 == 0 ? "Buzz"
                             : String.valueOf(i))
                .forEach(System.out::println);
    }

    public static void main(String[] args) {
        fizzBuzz(15);
        fizzBuzzStream(5);
    }
}
```

**Complexity:** O(n) time, O(1) space.

**Edge cases:** `n = 0` or negative (prints nothing).

**Follow-up:** "Make it extensible (add 7 → `Bazz`)?" → build the word by concatenating from a list of `(divisor, word)` rules and fall back to the number if the word is empty. This is a good place to show the Open/Closed principle.

---

### Q284. Fibonacci — recursion, iteration, memoization, streams.

**Problem:** Return the nth Fibonacci number (`0, 1, 1, 2, 3, 5, 8…`) and print the first n numbers.

**Approach:**
- *Plain recursion:* `fib(n) = fib(n-1) + fib(n-2)` recomputes the same values again and again → exponential O(2ⁿ).
- *Memoization:* store each computed value so every `fib(k)` is computed once → O(n).
- *Iteration:* only the last two values are needed → O(n) time, O(1) space. This is the answer to give.
- *Stream:* `Stream.iterate` with a pair `{a, b}` as the state.
- `long` overflows after `fib(92)`; use `BigInteger` beyond that.

```java
import java.math.BigInteger;
import java.util.*;
import java.util.stream.*;

class FibonacciDemo {
    static long fibRecursive(int n) {                       // O(2^n) - only to explain why it is bad
        return n < 2 ? n : fibRecursive(n - 1) + fibRecursive(n - 2);
    }

    static long fibMemo(int n) {
        return fibMemo(n, new long[n + 1]);
    }
    private static long fibMemo(int n, long[] memo) {
        if (n < 2) return n;
        if (memo[n] != 0) return memo[n];
        return memo[n] = fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
    }

    static long fibIterative(int n) {
        if (n < 2) return n;
        long prev = 0, curr = 1;
        for (int i = 2; i <= n; i++) {
            long next = prev + curr;
            prev = curr;
            curr = next;
        }
        return curr;
    }

    static List<Long> firstN(int n) {
        return Stream.iterate(new long[]{0, 1}, f -> new long[]{f[1], f[0] + f[1]})
                .limit(n).map(f -> f[0]).collect(Collectors.toList());
    }

    static BigInteger fibBig(int n) {
        BigInteger a = BigInteger.ZERO, b = BigInteger.ONE;
        for (int i = 0; i < n; i++) { BigInteger t = a.add(b); a = b; b = t; }
        return a;
    }

    public static void main(String[] args) {
        System.out.println(fibRecursive(10));   // 55
        System.out.println(fibMemo(50));        // 12586269025
        System.out.println(fibIterative(92));   // 7540113804746346429 (largest that fits in long)
        System.out.println(firstN(10));         // [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
        System.out.println(fibBig(100));        // 354224848179261915075
    }
}
```

**Complexity:** Recursion O(2ⁿ) time. Memo O(n) time and O(n) space (plus recursion depth). Iterative O(n) time, O(1) space. Matrix exponentiation / fast doubling: O(log n).

**Edge cases:** `n = 0`, `n = 1`, negative `n` (reject), `n > 92` for `long`. Deep memoised recursion can throw `StackOverflowError` for very large `n`.

**Follow-up:** "Why not memoize with `HashMap.computeIfAbsent` recursively?" → it throws `ConcurrentModificationException` in Java 9+ because the map is modified inside the compute function. "Climbing stairs?" → same recurrence.

---

### Q285. Prime check and Sieve of Eratosthenes.

**Problem:** Check whether `n` is prime; list all primes up to `n`.

**Approach:**
- A number is not prime if it has a divisor up to `√n` (divisors come in pairs, one of them is always ≤ √n). Test only 2 and odd numbers up to √n. Write the loop as `i <= n / i` so `i * i` can never overflow.
- *Sieve:* make a `boolean[]` marked "prime". For each prime `p`, cross out its multiples starting from `p*p`. Everything left unmarked is prime.

```java
import java.util.*;
import java.util.stream.*;

class PrimeDemo {
    static boolean isPrime(int n) {
        if (n < 2) return false;
        if (n < 4) return true;                    // 2 and 3
        if (n % 2 == 0) return false;
        for (int i = 3; i <= n / i; i += 2) {
            if (n % i == 0) return false;
        }
        return true;
    }

    static List<Integer> sieve(int n) {
        boolean[] composite = new boolean[n + 1];
        List<Integer> primes = new ArrayList<>();
        for (int i = 2; i <= n; i++) {
            if (composite[i]) continue;
            primes.add(i);
            for (long j = (long) i * i; j <= n; j += i) composite[(int) j] = true;
        }
        return primes;
    }

    static List<Integer> primesStream(int n) {
        return IntStream.rangeClosed(2, n).filter(PrimeDemo::isPrime).boxed().collect(Collectors.toList());
    }

    public static void main(String[] args) {
        System.out.println(isPrime(97));        // true
        System.out.println(isPrime(1));         // false
        System.out.println(sieve(30));          // [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
        System.out.println(primesStream(20));   // [2, 3, 5, 7, 11, 13, 17, 19]
    }
}
```

**Complexity:** `isPrime` O(√n) time, O(1) space. Sieve O(n log log n) time, O(n) space.

**Edge cases:** `0`, `1`, negatives, `2` (the only even prime), perfect squares like 49, `Integer.MAX_VALUE`.

**Follow-up:** "Is a 100-digit number prime?" → `BigInteger.isProbablePrime(certainty)` (Miller–Rabin). "Primes in a range `[l, r]` with huge `r`?" → segmented sieve.

---

### Q286. Factorial — long overflow and BigInteger.

**Problem:** Compute `n!` for small and very large `n`.

**Approach:**
- `20!` is the largest factorial that fits into a `long`. `21!` silently overflows to a wrong value — Java does not warn you.
- `Math.multiplyExact` throws `ArithmeticException` on overflow, which is the safe way to detect it.
- For big values use `BigInteger` and multiply in a loop or with `reduce`.

```java
import java.math.BigInteger;
import java.util.stream.*;

class FactorialDemo {
    static long factorial(int n) {
        if (n < 0) throw new IllegalArgumentException("n must be >= 0");
        long result = 1;
        for (int i = 2; i <= n; i++) result = Math.multiplyExact(result, i);   // throws after 20!
        return result;
    }

    static long factorialRecursive(int n) {
        return n <= 1 ? 1 : n * factorialRecursive(n - 1);
    }

    static BigInteger factorialBig(int n) {
        BigInteger r = BigInteger.ONE;
        for (int i = 2; i <= n; i++) r = r.multiply(BigInteger.valueOf(i));
        return r;
    }

    static BigInteger factorialStream(int n) {
        return IntStream.rangeClosed(2, n).mapToObj(BigInteger::valueOf)
                .reduce(BigInteger.ONE, BigInteger::multiply);
    }

    public static void main(String[] args) {
        System.out.println(factorial(20));        // 2432902008176640000
        System.out.println(factorialBig(25));     // 15511210043330985984000000
        System.out.println(factorialStream(5));   // 120
        try { factorial(21); } catch (ArithmeticException e) { System.out.println("overflow detected"); }
    }
}
```

**Complexity:** O(n) multiplications. `BigInteger` multiplications cost more as numbers grow, overall roughly O(n² log n) for naive multiplication. Recursion uses O(n) stack.

**Edge cases:** `0! = 1`, `1! = 1`, negative input, `n = 21` for `long`, recursion depth for large `n`.

**Follow-up:** "Count the trailing zeros of `n!` without computing it?" → sum `n/5 + n/25 + n/125 + …`. "Why does `BigInteger` not overflow?" → it uses an `int[]` magnitude array that grows as needed; objects are immutable so use the returned value.

---

### Q287. GCD and LCM.

**Problem:** GCD(48, 18) = 6, LCM(4, 6) = 12.

**Approach:**
- *Euclid:* `gcd(a, b) = gcd(b, a % b)` until `b` is 0. The remainder shrinks fast, so it is very quick.
- *LCM:* `lcm(a, b) = a / gcd(a, b) * b`. Divide **first**, then multiply, to reduce the overflow risk (`a * b` can overflow before dividing).
- For a list of numbers, fold using `reduce`.

```java
import java.math.BigInteger;
import java.util.stream.IntStream;

class GcdLcmDemo {
    static int gcd(int a, int b) {
        a = Math.abs(a); b = Math.abs(b);
        while (b != 0) {
            int t = a % b;
            a = b;
            b = t;
        }
        return a;
    }

    static int gcdRecursive(int a, int b) {
        return b == 0 ? Math.abs(a) : gcdRecursive(b, a % b);
    }

    static long lcm(int a, int b) {
        if (a == 0 || b == 0) return 0;
        return Math.abs((long) a / gcd(a, b) * b);
    }

    static int gcdOfArray(int... nums) {
        return IntStream.of(nums).reduce(0, GcdLcmDemo::gcd);   // gcd(0, x) = x
    }

    public static void main(String[] args) {
        System.out.println(gcd(48, 18));                                  // 6
        System.out.println(lcm(4, 6));                                    // 12
        System.out.println(gcdOfArray(12, 18, 24));                       // 6
        System.out.println(BigInteger.valueOf(48).gcd(BigInteger.valueOf(18))); // 6
    }
}
```

**Complexity:** O(log(min(a, b))) time, O(1) space (recursive: O(log) stack).

**Edge cases:** one value is 0 (`gcd(0, x) = x`), both 0 (gcd 0, lcm 0), negative numbers, LCM overflow for large inputs (return `long` or `BigInteger`), `Integer.MIN_VALUE` where `Math.abs` stays negative.

**Follow-up:** "Reduce a fraction `a/b`?" → divide both by their gcd. "Why does Euclid terminate?" → the second argument strictly decreases and stays ≥ 0.

---

### Q288. Bit manipulation — count set bits, power of two, swap without a temp variable.

**Problem:** (a) Count the `1` bits in an `int`. (b) Check if `n` is a power of two. (c) Swap two ints without a third variable.

**Approach:**
- *Count bits (Kernighan):* `n & (n - 1)` clears the lowest set bit. Repeat until `n` is 0; the number of repeats is the answer.
- *Power of two:* such a number has exactly one set bit, so `n > 0 && (n & (n - 1)) == 0`.
- *Swap with XOR:* `a ^= b; b ^= a; a ^= b;` XOR is its own inverse. Arithmetic swap (`a = a + b; b = a - b; a = a - b;`) also works for `int` because overflow wraps around consistently.
- Java is **pass-by-value**, so a `swap(int, int)` method cannot swap the caller's variables — you must swap array elements or fields.

```java
class BitDemo {
    static int countSetBits(int n) {
        int count = 0;
        while (n != 0) {
            n &= (n - 1);
            count++;
        }
        return count;                      // works for negative ints too (loop runs <= 32 times)
    }

    static boolean isPowerOfTwo(int n) {
        return n > 0 && (n & (n - 1)) == 0;
    }

    static void swapInArray(int[] a, int i, int j) {
        if (i == j) return;                // XOR-swapping the same slot would zero it
        a[i] ^= a[j];
        a[j] ^= a[i];
        a[i] ^= a[j];
    }

    public static void main(String[] args) {
        System.out.println(countSetBits(13));                // 3 (1101)
        System.out.println(Integer.bitCount(13));            // library version, same result
        System.out.println(isPowerOfTwo(64));                // true
        System.out.println(isPowerOfTwo(0));                 // false

        int a = 5, b = 9;
        a = a + b; b = a - b; a = a - b;
        System.out.println(a + " " + b);                     // 9 5
        a ^= b; b ^= a; a ^= b;
        System.out.println(a + " " + b);                     // 5 9
    }
}
```

**Complexity:** `countSetBits` O(k) where k = number of set bits (≤ 32). Others O(1).

**Edge cases:** `0` (not a power of two), negative numbers (`Integer.MIN_VALUE` has one set bit but is negative → `n > 0` check matters), swapping an element with itself via XOR.

**Follow-up:** "Find the single non-duplicated number in an array where others appear twice?" → XOR everything. "Check if the i-th bit is set?" → `(n >> i & 1) == 1`.

---

### Q289. Check balanced parentheses using a Deque.

**Problem:** `"{[()]}"` is balanced; `"([)]"` and `"(("` are not.

**Approach:**
- Walk through the characters. On an opening bracket **push** it. On a closing bracket, the top of the stack must be its matching opener — pop it; otherwise it is unbalanced.
- At the end the stack must be empty (no unclosed openers).
- Use `ArrayDeque` as a stack. Do **not** use `Stack` (legacy, synchronised).

```java
import java.util.*;

class BalancedParenthesesDemo {
    private static final Map<Character, Character> PAIRS = Map.of(')', '(', ']', '[', '}', '{');

    static boolean isBalanced(String s) {
        Deque<Character> stack = new ArrayDeque<>();
        for (char c : s.toCharArray()) {
            if (PAIRS.containsValue(c)) {
                stack.push(c);
            } else if (PAIRS.containsKey(c)) {
                if (stack.isEmpty() || !stack.pop().equals(PAIRS.get(c))) return false;
            }                                          // other characters are ignored
        }
        return stack.isEmpty();
    }

    public static void main(String[] args) {
        System.out.println(isBalanced("{[()]}"));   // true
        System.out.println(isBalanced("([)]"));     // false
        System.out.println(isBalanced("(("));       // false
        System.out.println(isBalanced(")"));        // false
        System.out.println(isBalanced(""));         // true
    }
}
```

**Complexity:** O(n) time, O(n) space.

**Edge cases:** starts with a closing bracket, extra opening brackets at the end (stack not empty), empty string, other characters mixed in. `ArrayDeque` does not allow `null`, which is fine here.

**Follow-up:** "Only one bracket type?" → a single counter is enough. "Minimum insertions to balance?" → track open count and unmatched closers. "Longest valid parentheses substring?" → stack of indexes.

---

### Q290. Reverse a singly linked list (iterative and recursive).

**Problem:** `1→2→3→4→null` becomes `4→3→2→1→null`.

**Approach:**
- *Iterative:* keep `prev` (starts `null`) and `curr` (starts at head). For each node: save `next`, point `curr.next` to `prev`, then move `prev` and `curr` one step forward. When `curr` is `null`, `prev` is the new head.
- *Recursive:* reverse the rest of the list first (`head.next` onwards). Then make `head.next.next = head` and cut the old link `head.next = null`. The head returned from the deepest call is passed all the way up.

```java
class ReverseListDemo {
    static class Node {
        int val; Node next;
        Node(int val) { this.val = val; }
    }

    static Node reverseIterative(Node head) {
        Node prev = null, curr = head;
        while (curr != null) {
            Node next = curr.next;      // 1. remember
            curr.next = prev;           // 2. flip the link
            prev = curr;                // 3. advance
            curr = next;
        }
        return prev;
    }

    static Node reverseRecursive(Node head) {
        if (head == null || head.next == null) return head;
        Node newHead = reverseRecursive(head.next);
        head.next.next = head;
        head.next = null;
        return newHead;
    }

    static Node build(int... v) {
        Node dummy = new Node(0), t = dummy;
        for (int x : v) { t.next = new Node(x); t = t.next; }
        return dummy.next;
    }

    static void print(Node n) {
        StringBuilder sb = new StringBuilder();
        for (; n != null; n = n.next) sb.append(n.val).append(n.next != null ? "->" : "");
        System.out.println(sb);
    }

    public static void main(String[] args) {
        print(reverseIterative(build(1, 2, 3, 4)));   // 4->3->2->1
        print(reverseRecursive(build(1, 2, 3, 4)));   // 4->3->2->1
        print(reverseIterative(null));                // (empty)
    }
}
```

**Complexity:** O(n) time. Iterative O(1) space; recursive O(n) stack (can overflow on very long lists).

**Edge cases:** `null` head, single node, two nodes.

**Follow-up:** "Reverse only between positions `m` and `n`?" → find the node before `m` and reverse that segment. "Reverse in groups of `k`?" → reverse each group and connect. "Doubly linked list?" → swap `next` and `prev` of each node.

---

### Q291. Detect a cycle in a linked list (Floyd) and find the middle node.

**Problem:** Return whether a linked list has a cycle; find where the cycle starts; return the middle node in one pass.

**Approach:**
- Use **two pointers**: `slow` moves 1 step, `fast` moves 2 steps.
- *Cycle:* if there is a loop, the fast pointer eventually laps the slow one and they meet. If there is none, `fast` reaches `null`.
- *Cycle start:* after they meet, reset one pointer to the head. Move both one step at a time; they meet at the cycle's first node (mathematically, the distance from head to the cycle start equals the distance from the meeting point to the cycle start).
- *Middle:* when `fast` reaches the end, `slow` is at the middle. For an even count this returns the **second** middle.

```java
class FloydDemo {
    static class Node {
        int val; Node next;
        Node(int val) { this.val = val; }
    }

    static boolean hasCycle(Node head) {
        Node slow = head, fast = head;
        while (fast != null && fast.next != null) {
            slow = slow.next;
            fast = fast.next.next;
            if (slow == fast) return true;
        }
        return false;
    }

    static Node cycleStart(Node head) {
        Node slow = head, fast = head;
        while (fast != null && fast.next != null) {
            slow = slow.next;
            fast = fast.next.next;
            if (slow == fast) {
                slow = head;
                while (slow != fast) { slow = slow.next; fast = fast.next; }
                return slow;
            }
        }
        return null;
    }

    static Node middle(Node head) {
        Node slow = head, fast = head;
        while (fast != null && fast.next != null) {
            slow = slow.next;
            fast = fast.next.next;
        }
        return slow;
    }

    public static void main(String[] args) {
        Node a = new Node(1), b = new Node(2), c = new Node(3), d = new Node(4), e = new Node(5);
        a.next = b; b.next = c; c.next = d; d.next = e;
        System.out.println(middle(a).val);        // 3
        System.out.println(hasCycle(a));          // false
        e.next = c;                               // create cycle 3->4->5->3
        System.out.println(hasCycle(a));          // true
        System.out.println(cycleStart(a).val);    // 3
    }
}
```

**Complexity:** O(n) time, O(1) space. (A `HashSet` of visited nodes also works but costs O(n) space.)

**Edge cases:** empty list, single node (with or without self-loop), two nodes, even vs odd length for the middle.

**Follow-up:** "Length of the cycle?" → after meeting, count steps until you meet again. "Is a linked list a palindrome?" → find middle, reverse second half, compare.

---

### Q292. Merge overlapping intervals.

**Problem:** `[[1,3],[2,6],[8,10],[15,18]]` → `[[1,6],[8,10],[15,18]]`.

**Approach:**
- Sort intervals by start. Then walk once, keeping the last merged interval.
- If the current interval starts **at or before** the last one's end, they overlap → extend the last end to `max(lastEnd, currentEnd)`. Otherwise start a new interval.
- Taking the max of the ends matters because a later interval can sit entirely inside the earlier one (`[1,10]` and `[2,3]`).

```java
import java.util.*;

class MergeIntervalsDemo {
    static int[][] merge(int[][] intervals) {
        if (intervals.length <= 1) return intervals;
        int[][] sorted = intervals.clone();
        Arrays.sort(sorted, Comparator.comparingInt(a -> a[0]));
        List<int[]> result = new ArrayList<>();
        int[] cur = sorted[0].clone();
        for (int i = 1; i < sorted.length; i++) {
            if (sorted[i][0] <= cur[1]) {
                cur[1] = Math.max(cur[1], sorted[i][1]);
            } else {
                result.add(cur);
                cur = sorted[i].clone();
            }
        }
        result.add(cur);
        return result.toArray(new int[0][]);
    }

    public static void main(String[] args) {
        int[][] in = {{8, 10}, {1, 3}, {2, 6}, {15, 18}};
        System.out.println(Arrays.deepToString(merge(in)));                        // [[1, 6], [8, 10], [15, 18]]
        System.out.println(Arrays.deepToString(merge(new int[][]{{1, 4}, {4, 5}}))); // [[1, 5]]
    }
}
```

**Complexity:** O(n log n) time (sorting), O(n) space for the output.

**Edge cases:** empty input, one interval, touching intervals (`[1,4]`,`[4,5]` — decide if they merge; usually yes), nested intervals, unsorted input, equal intervals.

**Follow-up:** "Insert a new interval into a sorted list?" → three-phase scan in O(n). "Minimum meeting rooms?" → sort starts and ends separately, or use a min-heap of end times.

---

### Q293. Group anagrams.

**Problem:** `["eat","tea","tan","ate","nat","bat"]` → `[["eat","tea","ate"],["tan","nat"],["bat"]]`.

**Approach:**
- Anagrams become identical when their letters are sorted. So use the **sorted letters as a map key** and group the original words under it.
- Faster key for lowercase `a–z`: a 26-length count array turned into a string — avoids the sort (O(k) per word instead of O(k log k)).

```java
import java.util.*;
import java.util.stream.*;

class GroupAnagramsDemo {
    static List<List<String>> group(String[] words) {
        Map<String, List<String>> map = new HashMap<>();
        for (String w : words) {
            char[] c = w.toCharArray();
            Arrays.sort(c);
            map.computeIfAbsent(new String(c), k -> new ArrayList<>()).add(w);
        }
        return new ArrayList<>(map.values());
    }

    static Collection<List<String>> groupStream(String[] words) {
        return Arrays.stream(words)
                .collect(Collectors.groupingBy(w -> {
                    char[] c = w.toCharArray();
                    Arrays.sort(c);
                    return new String(c);
                }))
                .values();
    }

    public static void main(String[] args) {
        System.out.println(group(new String[]{"eat", "tea", "tan", "ate", "nat", "bat"}));
        System.out.println(groupStream(new String[]{"abc", "cab", "xyz"}));
    }
}
```

**Complexity:** O(n · k log k) time for `n` words of length at most `k`; O(n · k) space.

**Edge cases:** empty string (all empty strings group together), single word, different cases (normalise with `toLowerCase`), duplicates in the input.

**Follow-up:** "Do it in O(n · k)?" → use the 26-count key. "Why is `char[]` not a good map key?" → arrays use identity `hashCode`/`equals`; convert to `String` or `List`.

---

### Q294. Longest substring without repeating characters.

**Problem:** `"abcabcbb"` → `3` (`"abc"`). `"bbbbb"` → `1`. `"pwwkew"` → `3` (`"wke"`).

**Approach:**
- Use a **sliding window** `[start, i]` that never contains a repeated character.
- Keep a map `char → last index seen`. When the current character was seen at an index `>= start`, move `start` to one past that index. Update the map and the best length every step.
- The check `>= start` is critical: an old occurrence that is already outside the window must be ignored.

```java
import java.util.*;

class LongestSubstringDemo {
    static int lengthOfLongestSubstring(String s) {
        Map<Character, Integer> last = new HashMap<>();
        int best = 0, start = 0;
        for (int i = 0; i < s.length(); i++) {
            char c = s.charAt(i);
            Integer prev = last.get(c);
            if (prev != null && prev >= start) start = prev + 1;
            last.put(c, i);
            best = Math.max(best, i - start + 1);
        }
        return best;
    }

    static String longestSubstring(String s) {            // returns the substring itself
        Map<Character, Integer> last = new HashMap<>();
        int best = 0, start = 0, bestStart = 0;
        for (int i = 0; i < s.length(); i++) {
            Integer prev = last.get(s.charAt(i));
            if (prev != null && prev >= start) start = prev + 1;
            last.put(s.charAt(i), i);
            if (i - start + 1 > best) { best = i - start + 1; bestStart = start; }
        }
        return s.substring(bestStart, bestStart + best);
    }

    public static void main(String[] args) {
        System.out.println(lengthOfLongestSubstring("abcabcbb"));  // 3
        System.out.println(lengthOfLongestSubstring("abba"));      // 2
        System.out.println(longestSubstring("pwwkew"));            // wke
    }
}
```

**Complexity:** O(n) time, O(min(n, alphabet)) space.

**Edge cases:** empty string (0), all same characters, all distinct, `"abba"` (the stale-index trap), spaces/symbols.

**Follow-up:** "At most `k` distinct characters?" → sliding window with a count map. "Longest substring with all unique characters in a stream?" → same window, evict from the left.

---

### Q295. Longest common prefix.

**Problem:** `["flower","flow","flight"]` → `"fl"`. `["dog","car"]` → `""`.

**Approach:**
- Take the first string as the candidate prefix. For every other string, shorten the prefix until the string **starts with** it. If the prefix becomes empty, stop early.
- Alternative: vertical scan — compare the character at column `i` across all strings; stop at the first mismatch.

```java
class LongestCommonPrefixDemo {
    static String longestCommonPrefix(String[] strs) {
        if (strs == null || strs.length == 0) return "";
        String prefix = strs[0];
        for (int i = 1; i < strs.length && !prefix.isEmpty(); i++) {
            while (!strs[i].startsWith(prefix)) {
                prefix = prefix.substring(0, prefix.length() - 1);
            }
        }
        return prefix;
    }

    static String verticalScan(String[] strs) {
        if (strs == null || strs.length == 0) return "";
        for (int col = 0; col < strs[0].length(); col++) {
            char c = strs[0].charAt(col);
            for (int row = 1; row < strs.length; row++) {
                if (col == strs[row].length() || strs[row].charAt(col) != c)
                    return strs[0].substring(0, col);
            }
        }
        return strs[0];
    }

    public static void main(String[] args) {
        System.out.println(longestCommonPrefix(new String[]{"flower", "flow", "flight"})); // fl
        System.out.println(verticalScan(new String[]{"dog", "car"}).isEmpty());            // true
        System.out.println(verticalScan(new String[]{"a"}));                               // a
    }
}
```

**Complexity:** O(S) time where S is the total number of characters in all strings; O(1) extra space.

**Edge cases:** empty array, array with one string, an empty string inside the array, one string being a prefix of another, `null` entries.

**Follow-up:** "Very many strings?" → build a `Trie` and walk while there is a single child and no word end. "Longest common *suffix*?" → reverse the strings first.

---

### Q296. String to integer (atoi) with overflow handling, and "is this a valid number?".

**Problem:** Implement `atoi`: skip leading spaces, read an optional `+`/`-`, read digits until a non-digit, and clamp to `Integer.MIN_VALUE..MAX_VALUE` on overflow. `"  -42abc"` → `-42`, `"91283472332"` → `2147483647`.

**Approach:**
- Parse step by step: whitespace → sign → digits.
- The overflow check has to happen **before** the multiplication: if `result > MAX/10`, or `result == MAX/10` and the next digit is greater than 7 (the last digit of 2147483647), the next step would overflow. Return `MAX` or `MIN` according to the sign. (For `-2147483648`, digit 8 with a negative sign correctly returns `MIN`.)
- Do not use `long` as a "cheat" unless the interviewer allows it; checking before multiplying works in `int` alone.
- For a *validation* question use `Integer.parseInt` in a `try/catch` or a regex; `NumberFormatException` is the library's signal.

```java
class AtoiDemo {
    static int myAtoi(String s) {
        int i = 0, n = s.length();
        while (i < n && s.charAt(i) == ' ') i++;
        int sign = 1;
        if (i < n && (s.charAt(i) == '+' || s.charAt(i) == '-')) {
            sign = s.charAt(i) == '-' ? -1 : 1;
            i++;
        }
        int result = 0;
        while (i < n && s.charAt(i) >= '0' && s.charAt(i) <= '9') {
            int digit = s.charAt(i) - '0';
            if (result > Integer.MAX_VALUE / 10 || (result == Integer.MAX_VALUE / 10 && digit > 7)) {
                return sign == 1 ? Integer.MAX_VALUE : Integer.MIN_VALUE;
            }
            result = result * 10 + digit;
            i++;
        }
        return result * sign;
    }

    static boolean isValidInt(String s) {
        try {
            Integer.parseInt(s);                     // accepts +/- sign, no spaces, no decimals
            return true;
        } catch (NumberFormatException e) {
            return false;
        }
    }

    static boolean isNumeric(String s) {             // simple decimal number: optional sign, digits, optional fraction
        return s != null && s.matches("[+-]?(\\d+(\\.\\d*)?|\\.\\d+)");
    }

    public static void main(String[] args) {
        System.out.println(myAtoi("  -42abc"));        // -42
        System.out.println(myAtoi("91283472332"));     // 2147483647
        System.out.println(myAtoi("-2147483648"));     // -2147483648
        System.out.println(myAtoi("words 987"));       // 0
        System.out.println(isValidInt("12a"));         // false
        System.out.println(isNumeric("-3.14"));        // true
    }
}
```

**Complexity:** O(n) time, O(1) space.

**Edge cases:** empty/blank string, only a sign (`"+"`), `"+-12"` (returns 0), leading zeros, `"-2147483648"`, `"2147483648"` (clamps), decimals (`"3.14"` → 3), `null`.

**Follow-up:** "Why does `Integer.parseInt(\" 12\")` fail?" → it does not trim. "Parse a double?" → `Double.parseDouble` also accepts `"NaN"`, `"Infinity"` and trailing `d`/`f`, so a regex is safer for strict validation.

---

### Q297. Implement a simplified custom HashMap (separate chaining).

**Problem:** Build `MyHashMap<K, V>` with `put`, `get`, `remove`, `size`, null-key support and automatic resizing.

**Approach:**
- An array of **buckets**; each bucket is a singly linked list of nodes (`hash, key, value, next`). This is "separate chaining" — the same idea as Java's `HashMap` (which additionally converts long chains to red-black trees).
- Bucket index = `hash & (length - 1)`. This only works when the length is a **power of two**, which is why we always double the table.
- Spread the hash (`h ^ (h >>> 16)`) so the high bits influence the index.
- `put`: walk the chain; if the key exists replace the value, else add a new node. If `size > capacity × 0.75`, **resize** to double capacity and re-link every node into the new table.
- Equality uses `Objects.equals` so a `null` key works (stored with hash 0).

```java
import java.util.Objects;

class MyHashMap<K, V> {
    private static class Node<K, V> {
        final int hash; final K key; V value; Node<K, V> next;
        Node(int hash, K key, V value, Node<K, V> next) {
            this.hash = hash; this.key = key; this.value = value; this.next = next;
        }
    }

    private static final float LOAD_FACTOR = 0.75f;
    private Node<K, V>[] table;
    private int size;

    @SuppressWarnings("unchecked")
    MyHashMap() { table = (Node<K, V>[]) new Node[16]; }

    private static int hash(Object key) {
        int h = (key == null) ? 0 : key.hashCode();
        return h ^ (h >>> 16);
    }

    private static int indexFor(int hash, int length) { return hash & (length - 1); }

    public V put(K key, V value) {
        int h = hash(key), i = indexFor(h, table.length);
        for (Node<K, V> n = table[i]; n != null; n = n.next) {
            if (n.hash == h && Objects.equals(n.key, key)) {
                V old = n.value;
                n.value = value;
                return old;
            }
        }
        table[i] = new Node<>(h, key, value, table[i]);       // insert at the head of the chain
        if (++size > table.length * LOAD_FACTOR) resize();
        return null;
    }

    public V get(Object key) {
        int h = hash(key);
        for (Node<K, V> n = table[indexFor(h, table.length)]; n != null; n = n.next)
            if (n.hash == h && Objects.equals(n.key, key)) return n.value;
        return null;
    }

    public V remove(Object key) {
        int h = hash(key), i = indexFor(h, table.length);
        Node<K, V> prev = null;
        for (Node<K, V> n = table[i]; n != null; prev = n, n = n.next) {
            if (n.hash == h && Objects.equals(n.key, key)) {
                if (prev == null) table[i] = n.next; else prev.next = n.next;
                size--;
                return n.value;
            }
        }
        return null;
    }

    public boolean containsKey(Object key) {
        int h = hash(key);
        for (Node<K, V> n = table[indexFor(h, table.length)]; n != null; n = n.next)
            if (n.hash == h && Objects.equals(n.key, key)) return true;
        return false;
    }

    public int size() { return size; }

    @SuppressWarnings("unchecked")
    private void resize() {
        Node<K, V>[] old = table;
        Node<K, V>[] bigger = (Node<K, V>[]) new Node[old.length * 2];
        for (Node<K, V> head : old) {
            while (head != null) {
                Node<K, V> next = head.next;
                int i = indexFor(head.hash, bigger.length);   // stored hash: no hashCode() call again
                head.next = bigger[i];
                bigger[i] = head;
                head = next;
            }
        }
        table = bigger;
    }

    public static void main(String[] args) {
        MyHashMap<String, Integer> m = new MyHashMap<>();
        for (int i = 0; i < 100; i++) m.put("k" + i, i);
        m.put(null, -1);
        System.out.println(m.size() + " " + m.get("k42") + " " + m.get(null)); // 101 42 -1
        System.out.println(m.remove("k42") + " " + m.get("k42") + " " + m.size()); // 42 null 100
    }
}
```

**Complexity:** `put/get/remove` average O(1), worst O(n) if all keys collide (Java 8+ real `HashMap` trees the bucket → O(log n)). Resize is O(n) but amortised O(1) per insert.

**Edge cases:** `null` key, overwriting an existing key, removing the head/middle/tail of a chain, many collisions, mutable keys (a changed `hashCode` makes the entry unreachable).

**Follow-up:** "Make it thread-safe?" → lock per bucket or use `ConcurrentHashMap` (CAS on empty bucket + `synchronized` on the bucket head). "Why is capacity a power of two?" → `&` instead of `%`, and resize splits each bucket cleanly in two.

---

### Q298. Implement an LRU cache (LinkedHashMap and manual doubly linked list + map).

**Problem:** Fixed capacity. `get(key)` and `put(key, value)` must both be O(1). When full, `put` evicts the **least recently used** entry.

**Approach:**
- *Quick version:* `LinkedHashMap` with `accessOrder = true` moves an entry to the end on every access. Override `removeEldestEntry` to remove the oldest once `size() > capacity`.
- *Manual version (what interviewers want to see):* a `HashMap<K, Node>` gives O(1) lookup and a **doubly linked list** gives O(1) "move to front" and "remove from tail". Keep most-recent next to `head`, least-recent next to `tail`. Sentinel `head`/`tail` nodes remove all null checks.
- Every `get` or `put` of an existing key moves the node to the front. A `put` on a full cache removes `tail.prev`, and also removes its key from the map.

```java
import java.util.*;

class LruWithLinkedHashMap<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    LruWithLinkedHashMap(int capacity) {
        super(16, 0.75f, true);                // true = access order
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }
}

class LruCache<K, V> {
    private class Node {
        K key; V value; Node prev, next;
        Node(K key, V value) { this.key = key; this.value = value; }
    }

    private final int capacity;
    private final Map<K, Node> map = new HashMap<>();
    private final Node head = new Node(null, null);     // most recently used side
    private final Node tail = new Node(null, null);     // least recently used side

    LruCache(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException("capacity must be > 0");
        this.capacity = capacity;
        head.next = tail;
        tail.prev = head;
    }

    public V get(K key) {
        Node n = map.get(key);
        if (n == null) return null;
        unlink(n);
        addFirst(n);
        return n.value;
    }

    public void put(K key, V value) {
        Node n = map.get(key);
        if (n != null) {
            n.value = value;
            unlink(n);
            addFirst(n);
            return;
        }
        if (map.size() == capacity) {
            Node lru = tail.prev;
            unlink(lru);
            map.remove(lru.key);                         // must remove from the map too
        }
        n = new Node(key, value);
        map.put(key, n);
        addFirst(n);
    }

    private void unlink(Node n) { n.prev.next = n.next; n.next.prev = n.prev; }

    private void addFirst(Node n) {
        n.next = head.next;
        n.prev = head;
        head.next.prev = n;
        head.next = n;
    }

    public int size() { return map.size(); }

    public static void main(String[] args) {
        LruCache<Integer, String> c = new LruCache<>(2);
        c.put(1, "one"); c.put(2, "two");
        c.get(1);                    // 1 becomes most recent
        c.put(3, "three");           // evicts 2
        System.out.println(c.get(2) + " " + c.get(1) + " " + c.get(3));   // null one three

        LruWithLinkedHashMap<String, Integer> m = new LruWithLinkedHashMap<>(2);
        m.put("a", 1); m.put("b", 2); m.get("a"); m.put("c", 3);
        System.out.println(m.keySet());                                    // [a, c]
    }
}
```

**Complexity:** `get` and `put` are O(1) time; O(capacity) space.

**Edge cases:** capacity 1, `put` of an existing key (must update and refresh, not evict), `get` of a missing key, `null` values (ambiguous with "missing" — use `Optional` or a contains check).

**Follow-up:** "Make it thread-safe?" → wrap in `synchronized` / `ReentrantLock` (even `get` mutates order, so a `ReadWriteLock` does not help). For production use Caffeine, which uses the W-TinyLFU policy and striped buffers. "LFU instead of LRU?" → map of frequency → linked set, track `minFreq`.

---

### Q299. Implement a thread-safe bounded blocking queue.

**Problem:** A queue with a fixed capacity. `put` blocks while the queue is full; `take` blocks while it is empty. Many producers and consumers.

**Approach:**
- Store items in a **circular array** (`head`, `tail`, `count`).
- Guard all state with **one lock** and two conditions: `notFull` (producers wait) and `notEmpty` (consumers wait).
- Always wait in a **`while` loop**, never an `if`, because of spurious wakeups and because another thread may consume the signal first.
- After a successful `put` signal `notEmpty`; after a successful `take` signal `notFull`. Using two conditions lets us signal exactly the right kind of waiter (`signal()` instead of `signalAll()`).
- `await()` releases the lock while waiting and re-acquires it on wakeup; the `unlock` goes in `finally`.

```java
import java.util.concurrent.locks.*;

class BoundedBlockingQueue<T> {
    private final Object[] items;
    private int head, tail, count;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    BoundedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        items = new Object[capacity];
    }

    public void put(T item) throws InterruptedException {
        lock.lockInterruptibly();
        try {
            while (count == items.length) notFull.await();
            items[tail] = item;
            tail = (tail + 1) % items.length;
            count++;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    @SuppressWarnings("unchecked")
    public T take() throws InterruptedException {
        lock.lockInterruptibly();
        try {
            while (count == 0) notEmpty.await();
            T item = (T) items[head];
            items[head] = null;                   // let GC reclaim it
            head = (head + 1) % items.length;
            count--;
            notFull.signal();
            return item;
        } finally {
            lock.unlock();
        }
    }

    public int size() {
        lock.lock();
        try { return count; } finally { lock.unlock(); }
    }

    public static void main(String[] args) throws Exception {
        BoundedBlockingQueue<Integer> q = new BoundedBlockingQueue<>(2);
        Thread producer = new Thread(() -> {
            try { for (int i = 1; i <= 5; i++) q.put(i); } catch (InterruptedException ignored) { }
        });
        producer.start();
        for (int i = 0; i < 5; i++) System.out.println("took " + q.take());
        producer.join();
    }
}
```

**Complexity:** `put`/`take` O(1); blocking time depends on the other side.

**Edge cases:** capacity 1, interrupt while waiting (propagate `InterruptedException`), wrap-around of the circular indexes, `null` items (allow or reject — `java.util.concurrent` queues reject `null`).

**Follow-up:** "Do it with `synchronized`, `wait()`, `notifyAll()`?" → single monitor, must use `notifyAll()` because producers and consumers wait on the same monitor. "Why is `ArrayBlockingQueue` better?" → battle-tested, supports timeouts, fairness. "Higher throughput?" → `LinkedBlockingQueue` uses two locks (put-lock and take-lock).

---

### Q300. Implement a simple rate limiter (token bucket and sliding window).

**Problem:** Allow at most N requests per time window per client; reject the rest.

**Approach:**
- *Token bucket:* a bucket holds up to `capacity` tokens and refills at `refillPerSecond`. A request takes one token; no token → rejected. It allows short **bursts** up to `capacity` while enforcing an average rate. Instead of a background refill thread, compute refill **lazily** from the elapsed time on each call.
- *Sliding window log:* store timestamps of recent requests in a deque. Drop timestamps older than the window; allow only if fewer than `limit` remain. It is exact but uses memory proportional to the limit.
- Use `System.nanoTime()` (monotonic), not `currentTimeMillis()` (can jump when the clock changes).
- Per-client limiting: `ConcurrentHashMap<String, Limiter>` with `computeIfAbsent`.

```java
import java.util.*;
import java.util.concurrent.*;

class TokenBucket {
    private final long capacity;
    private final double refillPerNano;
    private double tokens;
    private long lastRefill;

    TokenBucket(long capacity, double refillPerSecond) {
        this.capacity = capacity;
        this.refillPerNano = refillPerSecond / 1_000_000_000.0;
        this.tokens = capacity;
        this.lastRefill = System.nanoTime();
    }

    synchronized boolean tryAcquire() {
        long now = System.nanoTime();
        tokens = Math.min(capacity, tokens + (now - lastRefill) * refillPerNano);
        lastRefill = now;
        if (tokens >= 1) {
            tokens -= 1;
            return true;
        }
        return false;
    }
}

class SlidingWindowLimiter {
    private final int limit;
    private final long windowNanos;
    private final Deque<Long> timestamps = new ArrayDeque<>();

    SlidingWindowLimiter(int limit, long windowMillis) {
        this.limit = limit;
        this.windowNanos = TimeUnit.MILLISECONDS.toNanos(windowMillis);
    }

    synchronized boolean tryAcquire() {
        long now = System.nanoTime();
        while (!timestamps.isEmpty() && now - timestamps.peekFirst() >= windowNanos) {
            timestamps.pollFirst();
        }
        if (timestamps.size() < limit) {
            timestamps.addLast(now);
            return true;
        }
        return false;
    }
}

class RateLimiterDemo {
    private final ConcurrentMap<String, TokenBucket> perClient = new ConcurrentHashMap<>();

    boolean allow(String clientId) {
        return perClient.computeIfAbsent(clientId, k -> new TokenBucket(5, 1)).tryAcquire();
    }

    public static void main(String[] args) throws Exception {
        TokenBucket bucket = new TokenBucket(3, 1);          // burst of 3, then 1 per second
        for (int i = 1; i <= 5; i++) System.out.println("request " + i + " allowed? " + bucket.tryAcquire());
        Thread.sleep(1100);
        System.out.println("after 1.1s allowed? " + bucket.tryAcquire());

        SlidingWindowLimiter w = new SlidingWindowLimiter(2, 500);
        System.out.println(w.tryAcquire() + " " + w.tryAcquire() + " " + w.tryAcquire());  // true true false
    }
}
```

**Complexity:** Token bucket O(1) time and space per client. Sliding window log O(limit) space per client; each call is amortised O(1).

**Edge cases:** first request after a long idle period (tokens capped at capacity), `capacity` of 0, concurrent callers (hence `synchronized` or CAS), many unique client ids (need eviction — memory leak otherwise).

**Follow-up:** "Distributed rate limiting?" → Redis with an atomic Lua script (INCR + EXPIRE, or a token bucket script), or Bucket4j / Resilience4j `RateLimiter`. "Fixed window vs sliding window?" → fixed window allows 2× bursts across a boundary.

---

### Q301. Implement a custom ArrayList (dynamic array).

**Problem:** Build `MyArrayList<E>` that supports `add`, `add(index, e)`, `get`, `set`, `remove(index)`, `size` and iteration, growing automatically.

**Approach:**
- Hold an `Object[] data` and an `int size` (logical length). `data.length` is the capacity.
- When `size == data.length` allocate a bigger array (Java uses **×1.5**) and copy with `Arrays.copyOf`. Doubling or 1.5× makes `add` **amortised O(1)**; growing by +1 would be O(n²) overall.
- `add/remove` in the middle shift elements with `System.arraycopy`.
- After `remove`, set the freed last slot to `null` so the garbage collector can reclaim the object (avoids a memory leak).
- Implement `Iterable<E>` so for-each works.

```java
import java.util.*;

class MyArrayList<E> implements Iterable<E> {
    private static final int DEFAULT_CAPACITY = 10;
    private Object[] data = new Object[DEFAULT_CAPACITY];
    private int size;

    public boolean add(E e) {
        ensureCapacity(size + 1);
        data[size++] = e;
        return true;
    }

    public void add(int index, E e) {
        if (index < 0 || index > size) throw new IndexOutOfBoundsException("Index: " + index + ", Size: " + size);
        ensureCapacity(size + 1);
        System.arraycopy(data, index, data, index + 1, size - index);
        data[index] = e;
        size++;
    }

    @SuppressWarnings("unchecked")
    public E get(int index) {
        Objects.checkIndex(index, size);
        return (E) data[index];
    }

    public E set(int index, E e) {
        Objects.checkIndex(index, size);
        E old = get(index);
        data[index] = e;
        return old;
    }

    public E remove(int index) {
        Objects.checkIndex(index, size);
        E old = get(index);
        int moved = size - index - 1;
        if (moved > 0) System.arraycopy(data, index + 1, data, index, moved);
        data[--size] = null;
        return old;
    }

    public int size() { return size; }
    public boolean isEmpty() { return size == 0; }

    private void ensureCapacity(int min) {
        if (min <= data.length) return;
        int newCap = Math.max(min, data.length + (data.length >> 1));
        data = Arrays.copyOf(data, newCap);
    }

    @Override
    public Iterator<E> iterator() {
        return new Iterator<>() {
            private int cursor;
            @Override public boolean hasNext() { return cursor < size; }
            @SuppressWarnings("unchecked")
            @Override public E next() {
                if (cursor >= size) throw new NoSuchElementException();
                return (E) data[cursor++];
            }
        };
    }

    @Override
    public String toString() {
        StringJoiner j = new StringJoiner(", ", "[", "]");
        for (E e : this) j.add(String.valueOf(e));
        return j.toString();
    }

    public static void main(String[] args) {
        MyArrayList<String> list = new MyArrayList<>();
        for (int i = 0; i < 12; i++) list.add("v" + i);       // triggers a resize
        list.add(0, "first");
        list.remove(5);
        System.out.println(list.size() + " " + list.get(0) + " " + list);
    }
}
```

**Complexity:** `get/set` O(1); `add` at the end amortised O(1); `add/remove` at an index O(n); iteration O(n).

**Edge cases:** index equal to `size` (valid for `add`, invalid for `get`), negative index, growing from capacity 0, removing the last element (no shifting), memory leak if the slot is not nulled.

**Follow-up:** "Add fail-fast iteration?" → keep a `modCount` and compare it in `next()` to throw `ConcurrentModificationException`. "Why is `ArrayList` faster than `LinkedList` in practice?" → contiguous memory and CPU cache locality.

---

### Q302. Implement a stack using two queues, and a queue using two stacks.

**Problem:** Build LIFO behaviour with only FIFO queues, and FIFO behaviour with only LIFO stacks.

**Approach:**
- *Queue using two stacks (very common):* use an `in` stack and an `out` stack. `enqueue` pushes onto `in`. `dequeue` pops from `out`; if `out` is empty, first **move everything** from `in` to `out` (this reverses the order, so the oldest element ends on top). Every element is moved at most once, so `dequeue` is **amortised O(1)**.
- *Stack using two queues:* make `push` do the work. Add the new item to the empty helper queue `q2`, then move all of `q1` behind it, then swap the two queues. The most recent item is now at the front. `pop` is a simple `poll`. (It can also be done with a single queue by rotating after each push.)

```java
import java.util.*;

class QueueUsingTwoStacks<T> {
    private final Deque<T> in = new ArrayDeque<>();
    private final Deque<T> out = new ArrayDeque<>();

    public void enqueue(T x) { in.push(x); }

    public T dequeue() {
        shift();
        if (out.isEmpty()) throw new NoSuchElementException("queue is empty");
        return out.pop();
    }

    public T peek() {
        shift();
        if (out.isEmpty()) throw new NoSuchElementException("queue is empty");
        return out.peek();
    }

    private void shift() {
        if (out.isEmpty()) {
            while (!in.isEmpty()) out.push(in.pop());
        }
    }

    public boolean isEmpty() { return in.isEmpty() && out.isEmpty(); }
}

class StackUsingTwoQueues<T> {
    private Queue<T> q1 = new LinkedList<>();
    private Queue<T> q2 = new LinkedList<>();

    public void push(T x) {
        q2.offer(x);
        while (!q1.isEmpty()) q2.offer(q1.poll());
        Queue<T> t = q1; q1 = q2; q2 = t;
    }

    public T pop() {
        if (q1.isEmpty()) throw new NoSuchElementException("stack is empty");
        return q1.poll();
    }

    public T top() {
        if (q1.isEmpty()) throw new NoSuchElementException("stack is empty");
        return q1.peek();
    }

    public boolean isEmpty() { return q1.isEmpty(); }
}

class StackQueueDemo {
    public static void main(String[] args) {
        QueueUsingTwoStacks<Integer> q = new QueueUsingTwoStacks<>();
        q.enqueue(1); q.enqueue(2); q.enqueue(3);
        System.out.println(q.dequeue());     // 1
        q.enqueue(4);
        System.out.println(q.dequeue() + " " + q.dequeue() + " " + q.dequeue()); // 2 3 4

        StackUsingTwoQueues<String> s = new StackUsingTwoQueues<>();
        s.push("a"); s.push("b"); s.push("c");
        System.out.println(s.pop() + " " + s.pop() + " " + s.top());             // c b a
    }
}
```

**Complexity:** Queue from two stacks: `enqueue` O(1), `dequeue` amortised O(1). Stack from two queues (this version): `push` O(n), `pop` O(1). Space O(n).

**Edge cases:** dequeue/pop on empty (throw `NoSuchElementException`), interleaving enqueue and dequeue (do **not** refill `out` while it still has items — it would break ordering), single element.

**Follow-up:** "Why `ArrayDeque` instead of `Stack`?" → `Stack` is a legacy synchronised `Vector`; `Deque` is the recommended stack API. "Make `pop` O(1) and `push` O(n) or the opposite?" → either trade-off is valid; state which you chose.

---

### Q303. Design an immutable class and a thread-safe singleton.

**Problem:** (a) Write a truly immutable class that holds a mutable `Date` and a `List`. (b) Write a thread-safe singleton and compare approaches.

**Approach — immutable class (five rules):**
1. Make the class `final` (so no subclass can add mutable state).
2. Make all fields `private final`.
3. No setters.
4. **Defensive copy** mutable inputs in the constructor.
5. **Defensive copy** (or unmodifiable view) mutable fields in getters. Never leak internal references.

**Approach — singleton:**
- *Enum singleton:* the JVM guarantees one instance, serialization-safe and reflection-safe (Joshua Bloch's recommendation).
- *Initialization-on-demand holder:* the nested class is loaded only when `getInstance()` is first called; class loading is thread-safe, lazy and lock-free.
- *Double-checked locking:* needs the field to be **`volatile`**, otherwise another thread can see a half-constructed object (reordering of allocate / assign / construct).

```java
import java.util.*;

final class Employee {                      // immutable
    private final String name;
    private final Date joined;
    private final List<String> skills;

    Employee(String name, Date joined, List<String> skills) {
        this.name = Objects.requireNonNull(name);
        this.joined = new Date(joined.getTime());           // defensive copy in
        this.skills = List.copyOf(skills);                  // immutable copy; rejects nulls
    }

    public String getName() { return name; }
    public Date getJoined() { return new Date(joined.getTime()); }  // defensive copy out
    public List<String> getSkills() { return skills; }              // already immutable

    public Employee withSkill(String skill) {              // "wither" returns a new object
        List<String> copy = new ArrayList<>(skills);
        copy.add(skill);
        return new Employee(name, joined, copy);
    }
}

enum EnumSingleton {
    INSTANCE;
    public String hello() { return "hello"; }
}

class HolderSingleton {
    private HolderSingleton() { }
    private static class Holder {
        static final HolderSingleton INSTANCE = new HolderSingleton();
    }
    static HolderSingleton getInstance() { return Holder.INSTANCE; }
}

class DclSingleton {
    private static volatile DclSingleton instance;
    private DclSingleton() { }

    static DclSingleton getInstance() {
        DclSingleton local = instance;
        if (local == null) {
            synchronized (DclSingleton.class) {
                local = instance;
                if (local == null) instance = local = new DclSingleton();
            }
        }
        return local;
    }
}

class ImmutableSingletonDemo {
    public static void main(String[] args) {
        List<String> skills = new ArrayList<>(List.of("Java"));
        Employee e = new Employee("Asha", new Date(), skills);
        skills.add("Hacked");                                 // does not affect e
        System.out.println(e.getSkills());                   // [Java]
        System.out.println(e.withSkill("Spring").getSkills()); // [Java, Spring]
        System.out.println(HolderSingleton.getInstance() == HolderSingleton.getInstance()); // true
        System.out.println(DclSingleton.getInstance() == DclSingleton.getInstance());       // true
        System.out.println(EnumSingleton.INSTANCE.hello());
    }
}
```

**Complexity:** O(1) for the singletons; the immutable constructor is O(size of the list).

**Edge cases:** `Collections.unmodifiableList(list)` is only a **view** — if the caller keeps the original list and mutates it, your "immutable" object changes. `List.copyOf` makes a real copy. A shallow-immutable class holding mutable elements is still not immutable. Reflection, serialization and cloning can break a classic singleton (enum avoids all three).

**Follow-up:** "Why is `volatile` needed in DCL?" → without it, the reference can be published before the constructor finishes. "Use a Java `record`?" → records are shallowly immutable: fields are final, but you must still copy mutable components in the compact constructor.

---

### Q304. Sort a Map by value (and by key).

**Problem:** Given `{"a"=3,"b"=1,"c"=2}` produce a map ordered by value ascending/descending.

**Approach:**
- A `HashMap` has no order and `TreeMap` can only sort by **key**. So take the `entrySet()`, sort the entries with `Map.Entry.comparingByValue()`, and collect into a **`LinkedHashMap`** (it preserves insertion order).
- Collecting with `Collectors.toMap` requires a merge function and a `LinkedHashMap::new` supplier, otherwise the order is lost.
- Add a secondary comparator (key) to break ties and keep results deterministic.

```java
import java.util.*;
import java.util.stream.*;

class SortMapDemo {
    static <K extends Comparable<K>, V extends Comparable<V>> Map<K, V> sortByValueDesc(Map<K, V> map) {
        return map.entrySet().stream()
                .sorted(Map.Entry.<K, V>comparingByValue().reversed()
                        .thenComparing(Map.Entry.comparingByKey()))
                .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue,
                        (a, b) -> a, LinkedHashMap::new));
    }

    static Map<String, Integer> sortByValueAsc(Map<String, Integer> map) {
        List<Map.Entry<String, Integer>> entries = new ArrayList<>(map.entrySet());
        entries.sort(Map.Entry.comparingByValue());                 // pre-Java 8 style with a List
        Map<String, Integer> result = new LinkedHashMap<>();
        for (Map.Entry<String, Integer> e : entries) result.put(e.getKey(), e.getValue());
        return result;
    }

    public static void main(String[] args) {
        Map<String, Integer> m = new HashMap<>(Map.of("a", 3, "b", 1, "c", 2, "d", 3));
        System.out.println(sortByValueDesc(m));   // {a=3, d=3, c=2, b=1}
        System.out.println(sortByValueAsc(m));    // b=1, c=2, then a=3 and d=3 in some order
        System.out.println(new TreeMap<>(m));     // sorted by key: {a=3, b=1, c=2, d=3}
    }
}
```

**Complexity:** O(n log n) time, O(n) space.

**Edge cases:** equal values (tie-breaker), `null` values (comparator throws NPE — use `Comparator.nullsLast`), the result is a **snapshot**; later changes to the original map are not reflected.

**Follow-up:** "Keep the map always sorted by value?" → not possible with one map; keep a second `TreeSet` of entries or use a priority queue. "Top 3 by value?" → `.limit(3)` after sorting, or a size-3 heap.

---

### Q305. Sort employees by multiple fields using a Comparator chain.

**Problem:** Sort by department ascending, then salary descending, then name ascending. Handle `null` names.

**Approach:**
- `Comparator.comparing(keyExtractor)` builds a comparator from a field. Chain more with `.thenComparing(...)`. Use `.reversed()` on the specific comparator you want to reverse, or `Comparator.reverseOrder()` as the second argument.
- Be careful: `.reversed()` at the **end of a chain** reverses the *whole* chain, not just the last step.
- For primitive fields use `comparingDouble` / `comparingInt` to avoid boxing.
- For nullable fields wrap with `Comparator.nullsFirst/nullsLast`.

```java
import java.util.*;
import java.util.stream.*;

class SortEmployeesDemo {
    record Employee(int id, String name, String dept, double salary) {}

    public static void main(String[] args) {
        List<Employee> list = new ArrayList<>(List.of(
                new Employee(1, "Zoya", "IT", 90000),
                new Employee(2, "Amit", "IT", 120000),
                new Employee(3, "Meena", "HR", 70000),
                new Employee(4, "Bala", "IT", 90000),
                new Employee(5, null, "HR", 70000)));

        Comparator<Employee> byDeptSalaryName =
                Comparator.comparing(Employee::dept)
                        .thenComparing(Employee::salary, Comparator.reverseOrder())
                        .thenComparing(Employee::name, Comparator.nullsLast(Comparator.naturalOrder()));

        list.sort(byDeptSalaryName);
        list.forEach(System.out::println);
        // HR Meena 70000, HR null 70000, IT Amit 120000, IT Bala 90000, IT Zoya 90000

        List<Employee> sortedCopy = list.stream()
                .sorted(Comparator.comparingDouble(Employee::salary).reversed()
                        .thenComparing(Employee::id))
                .collect(Collectors.toList());
        System.out.println(sortedCopy.get(0).name());   // Amit
    }
}
```

**Complexity:** O(n log n) time (TimSort, stable). Space O(n) for the stream copy; `List.sort` is in place.

**Edge cases:** `null` fields (NPE unless wrapped), equal elements (stable sort keeps original order), inconsistent comparators (never write `a - b` for ints — it overflows; use `Integer.compare`).

**Follow-up:** "Sort in the `compareTo` method itself?" → implement `Comparable<Employee>` for the natural order and use `Comparator` for alternative orders. "Is `Collections.sort` stable?" → yes, it uses TimSort (merge sort variant).

---

### Q306. Employee stream tasks — group by department, average, highest paid per department, partition, join names, top 3.

**Problem:** On a `List<Employee>`: group by department; average salary per department; highest-paid employee in each department; partition by salary threshold; comma-joined names; top 3 salaries; department with the highest total payroll.

**Approach:** Almost every task is `Collectors.groupingBy(classifier, downstream)`. The *downstream collector* decides what you keep per group: `counting()`, `averagingDouble()`, `summingDouble()`, `mapping()`, `maxBy()`. Wrap `maxBy` with `collectingAndThen(..., Optional::get)` if you want the object, not an `Optional`. `partitioningBy` is a `groupingBy` on a boolean and always contains both keys.

```java
import java.util.*;
import java.util.function.Function;
import java.util.stream.*;

class EmployeeStreamsDemo {
    record Employee(int id, String name, String dept, double salary) {}

    public static void main(String[] args) {
        List<Employee> emps = List.of(
                new Employee(1, "Asha", "IT", 90000),
                new Employee(2, "Ravi", "IT", 120000),
                new Employee(3, "Meena", "HR", 70000),
                new Employee(4, "John", "HR", 75000),
                new Employee(5, "Sara", "FIN", 95000),
                new Employee(6, "Kiran", "IT", 60000));

        // 1. Group by department
        Map<String, List<Employee>> byDept = emps.stream().collect(Collectors.groupingBy(Employee::dept));

        // 2. Count and average salary per department
        Map<String, Long> countPerDept = emps.stream()
                .collect(Collectors.groupingBy(Employee::dept, Collectors.counting()));
        Map<String, Double> avgPerDept = emps.stream()
                .collect(Collectors.groupingBy(Employee::dept, Collectors.averagingDouble(Employee::salary)));

        // 3. Highest paid employee per department
        Map<String, Employee> topPerDept = emps.stream()
                .collect(Collectors.groupingBy(Employee::dept,
                        Collectors.collectingAndThen(
                                Collectors.maxBy(Comparator.comparingDouble(Employee::salary)),
                                Optional::get)));

        // 4. Partition: salary > 80000
        Map<Boolean, List<String>> partition = emps.stream()
                .collect(Collectors.partitioningBy(e -> e.salary() > 80000,
                        Collectors.mapping(Employee::name, Collectors.toList())));

        // 5. Join all names, sorted
        String names = emps.stream().map(Employee::name).sorted().collect(Collectors.joining(", ", "[", "]"));

        // 6. Top 3 highest paid
        List<Employee> top3 = emps.stream()
                .sorted(Comparator.comparingDouble(Employee::salary).reversed())
                .limit(3).collect(Collectors.toList());

        // 7. Department with the highest total payroll
        Optional<Map.Entry<String, Double>> richestDept = emps.stream()
                .collect(Collectors.groupingBy(Employee::dept, Collectors.summingDouble(Employee::salary)))
                .entrySet().stream().max(Map.Entry.comparingByValue());

        // 8. Salary statistics, and name -> salary map
        DoubleSummaryStatistics stats = emps.stream().mapToDouble(Employee::salary).summaryStatistics();
        Map<String, Double> nameToSalary = emps.stream()
                .collect(Collectors.toMap(Employee::name, Employee::salary));

        System.out.println(countPerDept);                       // IT=3, HR=2, FIN=1 (order not guaranteed)
        System.out.println(avgPerDept.get("IT"));               // 90000.0
        System.out.println(topPerDept.get("HR").name());        // John
        System.out.println(partition.get(true));                // [Asha, Ravi, Sara]
        System.out.println(names);                              // [Asha, John, Kiran, Meena, Ravi, Sara]
        System.out.println(top3.stream().map(Employee::name).toList());  // [Ravi, Sara, Asha]
        System.out.println(richestDept.get().getKey());         // IT
        System.out.println(stats.getMax() + " " + stats.getAverage());
        System.out.println(nameToSalary.size() + " " + byDept.size());   // 6 3
    }
}
```

**Complexity:** O(n) for grouping and statistics; O(n log n) for sort-based tasks (top-N can be O(n log k) with a heap).

**Edge cases:** empty list (`Optional::get` is safe inside `groupingBy` only because each group has at least one element — never call it on a standalone `maxBy` of an empty stream), `toMap` throws `IllegalStateException` on duplicate keys unless you pass a merge function, `null` department (`groupingBy` throws NPE on null keys), `averagingDouble` returns `0.0` for an empty group.

**Follow-up:** "Why not `parallelStream()`?" → only worthwhile for large CPU-bound data; shared mutable state inside lambdas is a bug. "`toList()` vs `Collectors.toList()`?" → `Stream.toList()` (Java 16+) returns an unmodifiable list and allows `null`s.

---

### Q307. Flatten a nested list.

**Problem:** `[1, [2, [3, 4]], 5, [[6]]]` → `[1,2,3,4,5,6]`. Also flatten `List<List<Integer>>` with streams.

**Approach:**
- *`List<List<T>>` (one level):* `flatMap(List::stream)`.
- *Arbitrary depth:* walk the structure recursively. If an element is a `List`, recurse into it; otherwise add it to the result.
- *Without recursion* (avoids stack overflow for very deep nesting): use a `Deque` of iterators — push the iterator of each nested list and continue with the top iterator.

```java
import java.util.*;
import java.util.stream.*;

class FlattenDemo {
    static List<Integer> flattenOneLevel(List<List<Integer>> nested) {
        return nested.stream().flatMap(List::stream).collect(Collectors.toList());
    }

    static List<Object> flatten(List<?> nested) {
        List<Object> out = new ArrayList<>();
        flattenInto(nested, out);
        return out;
    }

    private static void flattenInto(List<?> in, List<Object> out) {
        for (Object o : in) {
            if (o instanceof List<?> inner) flattenInto(inner, out);     // pattern matching (Java 16+)
            else out.add(o);
        }
    }

    static List<Object> flattenIterative(List<?> nested) {
        List<Object> out = new ArrayList<>();
        Deque<Iterator<?>> stack = new ArrayDeque<>();
        stack.push(nested.iterator());
        while (!stack.isEmpty()) {
            Iterator<?> it = stack.peek();
            if (!it.hasNext()) { stack.pop(); continue; }
            Object o = it.next();
            if (o instanceof List<?> inner) stack.push(inner.iterator());
            else out.add(o);
        }
        return out;
    }

    public static void main(String[] args) {
        List<Object> nested = List.of(1, List.of(2, List.of(3, 4)), 5, List.of(List.of(6)));
        System.out.println(flatten(nested));                // [1, 2, 3, 4, 5, 6]
        System.out.println(flattenIterative(nested));       // [1, 2, 3, 4, 5, 6]
        System.out.println(flattenOneLevel(List.of(List.of(1, 2), List.of(), List.of(3)))); // [1, 2, 3]
    }
}
```

**Complexity:** O(N) time where N is the total number of elements (including nested lists); O(depth) extra space (plus the output).

**Edge cases:** empty inner lists, `null` elements, a list that contains itself (infinite recursion — mention it), very deep nesting (use the iterative version).

**Follow-up:** "Flatten a `Map<String, List<String>>` into all values?" → `map.values().stream().flatMap(List::stream)`. "Implement `NestedIterator`?" → the Deque-of-iterators approach with a lazy `hasNext`.

---

### Q308. Matrix problems — rotate 90° in place and spiral traversal.

**Problem:** Rotate an `n × n` matrix 90° clockwise in place. Print any `m × n` matrix in spiral order.

**Approach:**
- *Rotate clockwise:* **transpose** the matrix (swap `m[i][j]` with `m[j][i]`), then **reverse every row**. For anticlockwise, reverse each row first and then transpose (or reverse the columns).
- *Spiral:* maintain four boundaries `top, bottom, left, right`. Walk the top row left→right, then shrink `top`; right column top→bottom, shrink `right`; bottom row right→left (only if `top <= bottom`), shrink `bottom`; left column bottom→top (only if `left <= right`), shrink `left`. The two guard `if`s prevent duplicates for single-row/single-column leftovers.

```java
import java.util.*;

class MatrixDemo {
    static void rotateClockwise(int[][] m) {
        int n = m.length;
        for (int i = 0; i < n; i++)
            for (int j = i + 1; j < n; j++) {
                int t = m[i][j]; m[i][j] = m[j][i]; m[j][i] = t;
            }
        for (int[] row : m) {
            for (int l = 0, r = n - 1; l < r; l++, r--) {
                int t = row[l]; row[l] = row[r]; row[r] = t;
            }
        }
    }

    static List<Integer> spiral(int[][] m) {
        List<Integer> res = new ArrayList<>();
        if (m.length == 0) return res;
        int top = 0, bottom = m.length - 1, left = 0, right = m[0].length - 1;
        while (top <= bottom && left <= right) {
            for (int j = left; j <= right; j++) res.add(m[top][j]);
            top++;
            for (int i = top; i <= bottom; i++) res.add(m[i][right]);
            right--;
            if (top <= bottom) {
                for (int j = right; j >= left; j--) res.add(m[bottom][j]);
                bottom--;
            }
            if (left <= right) {
                for (int i = bottom; i >= top; i--) res.add(m[i][left]);
                left++;
            }
        }
        return res;
    }

    public static void main(String[] args) {
        int[][] m = {{1, 2, 3}, {4, 5, 6}, {7, 8, 9}};
        System.out.println(spiral(m));                           // [1, 2, 3, 6, 9, 8, 7, 4, 5]
        rotateClockwise(m);
        System.out.println(Arrays.deepToString(m));              // [[7, 4, 1], [8, 5, 2], [9, 6, 3]]
        System.out.println(spiral(new int[][]{{1, 2, 3, 4}}));   // [1, 2, 3, 4]
        System.out.println(spiral(new int[][]{{1}, {2}, {3}}));  // [1, 2, 3]
    }
}
```

**Complexity:** Rotate O(n²) time, O(1) space. Spiral O(m·n) time, O(1) extra space apart from the output.

**Edge cases:** empty matrix, `1 × 1`, a single row or column (spiral guards), non-square matrices (rotation in place only works for square; a non-square needs a new array), jagged arrays.

**Follow-up:** "Set entire row and column to zero if an element is zero in O(1) space?" → use the first row and column as markers. "Search in a row/column-sorted matrix?" → start from the top-right corner, O(m + n).

---

### Q309. String compression and checking if one string is a rotation of another.

**Problem:** (a) Compress `"aaabbc"` to `"a3b2c1"`; if the result is not shorter return the original. (b) Is `"erbottlewat"` a rotation of `"waterbottle"`?

**Approach:**
- *Compression:* walk the string and count the length of each run of equal characters. Append the character and the count with a `StringBuilder`. Compare lengths at the end.
- *Rotation:* if the lengths are equal, `s2` is a rotation of `s1` exactly when `s2` is a substring of `s1 + s1`. (Doubling the string contains every rotation.)

```java
class CompressionRotationDemo {
    static String compress(String s) {
        if (s == null || s.length() < 2) return s;
        StringBuilder sb = new StringBuilder();
        int count = 1;
        for (int i = 1; i <= s.length(); i++) {
            if (i < s.length() && s.charAt(i) == s.charAt(i - 1)) {
                count++;
            } else {
                sb.append(s.charAt(i - 1)).append(count);
                count = 1;
            }
        }
        return sb.length() < s.length() ? sb.toString() : s;
    }

    static boolean isRotation(String a, String b) {
        return a != null && b != null && a.length() == b.length() && (a + a).contains(b);
    }

    public static void main(String[] args) {
        System.out.println(compress("aaabbc"));                      // a3b2c1 -> same length (6) so original kept
        System.out.println(compress("aaaaabbbbbcc"));                // a5b5c2
        System.out.println(compress("abc"));                         // abc
        System.out.println(isRotation("waterbottle", "erbottlewat")); // true
        System.out.println(isRotation("abc", "acb"));                // false
    }
}
```

**Complexity:** Compression O(n) time, O(n) space. Rotation O(n) average with `contains` (worst case depends on the substring algorithm, still practically linear).

**Edge cases:** empty string, single character, no repeated characters (compressed form is longer → return original), counts of 10 or more (multi-digit — `StringBuilder.append(int)` handles it), empty strings in `isRotation` (`"".contains("")` is true), `null`.

**Follow-up:** "Decompress `a3b2`?" → parse the letter, then read digits until the next letter (counts can have several digits). "Check rotation without creating the doubled string?" → KMP / index arithmetic modulo `n`.

---

### Q310. Producer–consumer using `BlockingQueue`.

**Problem:** One thread produces items, another consumes them; the producer must wait when the buffer is full, the consumer must wait when it is empty. (Concurrency is covered in depth in another section; this is the short coding answer.)

**Approach:**
- `ArrayBlockingQueue` already does all the waiting: `put()` blocks when full and `take()` blocks when empty. No explicit `wait`/`notify` is required.
- To stop the consumer cleanly send a **poison pill** (a special sentinel value) after the last item. Each consumer exits when it takes the pill.
- Always restore the interrupt flag when catching `InterruptedException`.

```java
import java.util.concurrent.*;

class ProducerConsumerDemo {
    private static final int POISON = -1;

    public static void main(String[] args) throws Exception {
        BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(3);

        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 6; i++) {
                    queue.put(i);                       // blocks when full
                    System.out.println("produced " + i);
                }
                queue.put(POISON);                      // tell the consumer to stop
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "producer");

        Thread consumer = new Thread(() -> {
            try {
                while (true) {
                    int item = queue.take();            // blocks when empty
                    if (item == POISON) break;
                    System.out.println("consumed " + item);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "consumer");

        producer.start();
        consumer.start();
        producer.join();
        consumer.join();
    }
}
```

**Complexity:** O(1) per operation; throughput is bounded by the slower side and the buffer size.

**Edge cases:** several consumers (send one poison pill **per** consumer), producer faster than consumer (bounded queue gives natural back-pressure — an unbounded queue would grow until `OutOfMemoryError`), interrupts, exceptions inside the consumer (the thread dies silently unless handled).

**Follow-up:** "Use `offer(item, timeout, unit)` / `poll(timeout, unit)`?" → avoids blocking forever. "Better than raw threads?" → submit producer and consumer to an `ExecutorService`. "Virtual threads?" → blocking `put/take` is cheap on virtual threads (Java 21).

---

### Q311. Print odd and even numbers alternately with two threads (wait/notify and Semaphore).

**Problem:** Two threads print `1, 2, 3, … N` in order — one prints odd numbers, the other prints even numbers.

**Approach:**
- *wait/notify:* both threads share a lock object and a counter. A thread waits **in a `while` loop** until it is its turn (odd/even matches), prints, increments the counter and calls `notifyAll()`. When the counter passes `N` both threads must exit (and wake the other so it can exit too).
- *Semaphore:* give the odd thread one permit, the even thread zero. Each thread acquires *its own* semaphore, prints, then releases the *other* semaphore. The hand-over order is enforced by the permits, no condition checks needed.

```java
import java.util.concurrent.Semaphore;

class OddEvenWaitNotify {
    private final Object lock = new Object();
    private int n = 1;
    private final int max;

    OddEvenWaitNotify(int max) { this.max = max; }

    void run(boolean odd) throws InterruptedException {
        synchronized (lock) {
            while (true) {
                while (n <= max && (n % 2 != 0) != odd) lock.wait();   // not my turn
                if (n > max) { lock.notifyAll(); return; }
                System.out.println(Thread.currentThread().getName() + ": " + n++);
                lock.notifyAll();
            }
        }
    }

    public static void main(String[] args) throws Exception {
        OddEvenWaitNotify p = new OddEvenWaitNotify(10);
        Thread t1 = new Thread(() -> { try { p.run(true); } catch (InterruptedException ignored) { } }, "odd");
        Thread t2 = new Thread(() -> { try { p.run(false); } catch (InterruptedException ignored) { } }, "even");
        t1.start(); t2.start();
        t1.join(); t2.join();
    }
}

class OddEvenSemaphore {
    public static void main(String[] args) throws Exception {
        final int max = 10;
        Semaphore oddTurn = new Semaphore(1);
        Semaphore evenTurn = new Semaphore(0);

        Thread odd = new Thread(() -> {
            try {
                for (int i = 1; i <= max; i += 2) {
                    oddTurn.acquire();
                    System.out.println("odd : " + i);
                    evenTurn.release();
                }
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        });
        Thread even = new Thread(() -> {
            try {
                for (int i = 2; i <= max; i += 2) {
                    evenTurn.acquire();
                    System.out.println("even: " + i);
                    oddTurn.release();
                }
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        });
        odd.start(); even.start();
        odd.join(); even.join();
    }
}
```

**Complexity:** O(N) time; O(1) space.

**Edge cases:** `wait()` must be inside `synchronized` (else `IllegalMonitorStateException`), `if` instead of `while` breaks on spurious wakeups, `notify()` instead of `notifyAll()` can wake the wrong thread and deadlock with more than two threads, odd `N` vs even `N` (the semaphore version leaves one harmless extra permit), a forgotten exit condition leaves a thread waiting forever.

**Follow-up:** "Three threads printing 1,2,3,1,2,3…?" → generalise to a `turn` index and `n % 3`, or a ring of semaphores. "Why Semaphore over wait/notify?" → no shared-flag bookkeeping, and permits are not lost if release happens before acquire.

---

### Q312. Deadlock — demonstrate it and fix it.

**Problem:** Write code that can deadlock, then fix it. (Deep coverage is in the concurrency section; this is the short coding version.)

**Approach:**
- A deadlock needs all four conditions: mutual exclusion, hold-and-wait, no pre-emption, and **circular wait**. Easiest to break in practice: circular wait.
- Classic bug: `transfer(A→B)` locks A then B, while at the same time `transfer(B→A)` locks B then A. Each thread holds one lock and waits for the other forever.
- *Fix 1 — global lock ordering:* always lock the account with the smaller id first, regardless of transfer direction.
- *Fix 2 — `tryLock` with timeout:* if the second lock cannot be obtained, release the first and retry/back off.
- Diagnose a running deadlock with `jstack <pid>` or `jcmd <pid> Thread.print` (reports "Found one Java-level deadlock") or `ThreadMXBean.findDeadlockedThreads()`.

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

class Account {
    final int id;
    private long balance;
    final ReentrantLock lock = new ReentrantLock();

    Account(int id, long balance) { this.id = id; this.balance = balance; }
    void debit(long amt) { balance -= amt; }
    void credit(long amt) { balance += amt; }
    long balance() { return balance; }
}

class Bank {
    // BUGGY: lock order depends on the argument order -> can deadlock
    static void transferDeadlockProne(Account from, Account to, long amt) {
        synchronized (from) {
            pause();                                    // widens the race window for the demo
            synchronized (to) {
                from.debit(amt);
                to.credit(amt);
            }
        }
    }

    // FIX 1: consistent global lock order (by id)
    static void transferOrdered(Account from, Account to, long amt) {
        Account first = from.id < to.id ? from : to;
        Account second = first == from ? to : from;
        synchronized (first) {
            synchronized (second) {
                from.debit(amt);
                to.credit(amt);
            }
        }
    }

    // FIX 2: tryLock with timeout and retry
    static boolean transferTryLock(Account from, Account to, long amt) throws InterruptedException {
        while (true) {
            if (from.lock.tryLock(50, TimeUnit.MILLISECONDS)) {
                try {
                    if (to.lock.tryLock(50, TimeUnit.MILLISECONDS)) {
                        try {
                            from.debit(amt);
                            to.credit(amt);
                            return true;
                        } finally {
                            to.lock.unlock();
                        }
                    }
                } finally {
                    from.lock.unlock();
                }
            }
            Thread.sleep(10);                           // back off, then retry
        }
    }

    private static void pause() {
        try { Thread.sleep(50); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }

    public static void main(String[] args) throws Exception {
        Account a = new Account(1, 1000), b = new Account(2, 1000);
        // swap in transferDeadlockProne to see both threads hang forever
        Thread t1 = new Thread(() -> transferOrdered(a, b, 100));
        Thread t2 = new Thread(() -> transferOrdered(b, a, 50));
        t1.start(); t2.start();
        t1.join(); t2.join();
        System.out.println(a.balance() + " " + b.balance());   // 950 1050
    }
}
```

**Complexity:** Locking adds O(1) overhead per transfer. Ordered locking guarantees progress; `tryLock` guarantees no permanent block but can **livelock** without randomised back-off.

**Edge cases:** transfer to the **same** account (`from == to` — with ordering you lock the same object twice, which is fine for `synchronized` because it is reentrant, but check and short-circuit it), equal ids, a lock held while calling foreign/"alien" methods (hidden cycles), nested lock acquisition across several classes.

**Follow-up:** "Other ways to avoid deadlocks?" → hold fewer locks, use higher-level classes (`ConcurrentHashMap`, `AtomicLong`), keep lock scope small, never call external code while holding a lock. "Deadlock vs livelock vs starvation?" → deadlock: nobody moves; livelock: everybody moves but makes no progress; starvation: one thread never gets the resource.

---

# 12. Tricky Output & Gotcha Questions (What Does This Print?)

> **How to use this section:** cover the answer, read the code, say the output *and the reason* out loud. Interviewers rarely care about the output itself — they want to hear the **rule**. Unless a full class is shown, treat each snippet as the body of `main`. Every output below was checked by compiling and running on a recent JDK (Java 25). Outputs marked "non-deterministic" depend on timing, so the *reason* is the answer.

### Q313. String pool: `==` vs `equals()` vs `intern()`

```java
String a = "java";
String b = "java";
String c = new String("java");
String d = c.intern();
System.out.println(a == b);
System.out.println(a == c);
System.out.println(a.equals(c));
System.out.println(a == d);
```

**Output:**
```
true
false
true
true
```

**Why:** Identical string *literals* are stored once in the String pool, so `a` and `b` point to the same object. `new String("java")` always creates a **new** object on the heap, so `a == c` is false. `equals()` compares the characters, so it is true. `intern()` returns the pooled instance, so `d` is the same object as `a`.

**Remember:** *`==` compares references, `equals()` compares content. Never use `==` on Strings (except for deliberate interning).*

---

### Q314. Compile-time constants: why `final` changes the `==` result

```java
String s1 = "java";
final String x = "ja";
String y = "ja";
System.out.println(s1 == (x + "va"));
System.out.println(s1 == (y + "va"));
System.out.println(s1 == (y + "va").intern());
```

**Output:**
```
true
false
true
```

**Why:** `x` is a `final` String initialised with a literal, so it is a **compile-time constant**. The compiler folds `x + "va"` into the literal `"java"` — same pooled object. `y` is not final, so `y + "va"` is computed at **runtime** and creates a new object. `intern()` brings it back to the pool.

**Remember:** *Only constant expressions (literals + `final` constants) are folded at compile time.*

---

### Q315. String concatenation: evaluation order and the loop trap

```java
String s = "";
for (int i = 0; i < 3; i++) s += i;
System.out.println(s);
System.out.println(1 + 2 + "a" + 1 + 2);
System.out.println("" + 1 + 2);
```

**Output:**
```
012
3a12
12
```

**Why:** `+` is evaluated **left to right**. In `1 + 2 + "a" + 1 + 2`, the first `1 + 2` is still number addition (`3`); once a String appears, everything after is concatenation → `"3a"`, then `"3a1"`, then `"3a12"`. In `"" + 1 + 2` the String is on the left from the start, so it is `"12"`.
Performance side note: `s += i` inside a loop builds a **new String every iteration** (O(n²) copying). Use one `StringBuilder` for loops.

**Remember:** *Left to right: numbers add until the first String, then everything concatenates.*

---

### Q316. Integer cache: `127` vs `128`

```java
Integer a = 127, b = 127;
Integer c = 128, d = 128;
System.out.println(a == b);
System.out.println(c == d);
System.out.println(c.equals(d));
int prim = 128;
System.out.println(c == prim);
Long l = 127L;
System.out.println(l.equals(127));
System.out.println(l.equals(127L));
```

**Output:**
```
true
false
true
true
false
true
```

**Why:** Autoboxing uses `Integer.valueOf()`, which returns **cached** objects for −128 to 127. So `a` and `b` are the same object, but 128 creates two different objects and `c == d` compares references. `c == prim` unboxes `c` and compares numbers → true. `Long.equals(Object)` requires the argument to be a `Long`; `127` is boxed to `Integer`, so it returns false.

**Remember:** *Compare wrapper objects with `equals()`; `==` on wrappers is only "accidentally" right inside the cache. `equals` also checks the type.*

---

### Q317. Ternary operator traps: unboxing NPE and numeric promotion

```java
boolean flag = true;
Integer boxed = null;
try {
    Integer r = flag ? boxed : 0;
    System.out.println(r);
} catch (NullPointerException e) {
    System.out.println("NPE");
}
Integer ok = flag ? boxed : Integer.valueOf(0);
System.out.println(ok);
System.out.println(true ? 1 : 2.0);
Object o1 = true ? 1 : "s";
System.out.println(o1);
Object o2 = true ? Integer.valueOf(1) : Double.valueOf(2);
System.out.println(o2);
System.out.println(true ? 'a' : 0);
int i = 0;
System.out.println(true ? 'a' : i);
```

**Output:**
```
NPE
null
1.0
1
1.0
a
97
```

**Why:**
- `flag ? boxed : 0` has one `Integer` and one `int`, so the result type is `int` → `boxed` is **unboxed**, and unboxing `null` throws NPE. With two `Integer` operands no unboxing happens.
- `true ? 1 : 2.0` — numeric operands are promoted to a common type (`double`), so `1` becomes `1.0`.
- `Object o1 = true ? 1 : "s"` — not both numeric, so no promotion; result stays `1`.
- `Integer` and `Double` (both convertible to numbers) are unboxed and promoted → `1.0`, even though you assign to `Object`.
- `'a' : 0` — `0` is a constant that fits in `char`, so the type is `char` → prints `a`. With a non-constant `int i` the type is `int` → `97`.

**Remember:** *If both branches are numeric (even boxed), the ternary promotes and unboxes — watch `null` and mixed types.*

---

### Q318. `char` arithmetic

```java
char c = 'a';
System.out.println(c + 1);
System.out.println((char) (c + 1));
c += 1;
System.out.println(c);
System.out.println('a' + 'b' + "c");
System.out.println("c" + 'a' + 'b');
StringBuilder sb = new StringBuilder();
sb.append('a' + 1);
sb.append((char) ('a' + 1));
System.out.println(sb);
```

**Output:**
```
98
b
b
195c
cab
98b
```

**Why:** In arithmetic, `char` is promoted to `int`, so `c + 1` is the number `98`. You must cast back to `char` to get `b`. Compound assignment `c += 1` has an **implicit cast**, so it stays a `char`. `'a' + 'b'` is `97 + 98 = 195` (numbers!) and only then joins `"c"`. With the String first, it is concatenation: `"cab"`. `sb.append(int)` appends the number `98`.

**Remember:** *`char` + anything numeric = `int`. Cast back when you want a character.*

---

### Q319. `i = i++` and friends

```java
int i = 0;
i = i++;
System.out.println(i);
int j = 5;
j = j++ + ++j;
System.out.println(j);
```

**Output:**
```
0
12
```

**Why:** `i++` returns the **old value** (0) and schedules the increment, but then the assignment writes that old value back into `i`, wiping the increment. For `j = j++ + ++j`: `j++` gives 5 (j becomes 6); `++j` makes j 7 and gives 7; the sum 12 is assigned.

**Remember:** *Post-increment returns the old value; never assign a variable from its own `++`. Don't write such code.*

---

### Q320. Integer overflow and `Math.abs(Integer.MIN_VALUE)`

```java
System.out.println(Integer.MAX_VALUE + 1);
System.out.println(1000 * 60 * 60 * 24 * 30);
System.out.println(1000 * 60 * 60 * 24 * 30L);
System.out.println(Math.abs(Integer.MIN_VALUE));
System.out.println(-Integer.MIN_VALUE == Integer.MIN_VALUE);
try { Math.addExact(Integer.MAX_VALUE, 1); }
catch (ArithmeticException e) { System.out.println(e.getMessage()); }
```

**Output:**
```
-2147483648
-1702967296
2592000000
-2147483648
true
integer overflow
```

**Why:** `int` arithmetic silently **wraps around** (two's complement). The 30-day-in-milliseconds expression is computed entirely in `int` before anything else, so it overflows; making **one operand `long`** (`30L`) fixes it. The `int` range is asymmetric: `MIN_VALUE` is −2147483648 but there is no +2147483648, so `abs` and negation of `MIN_VALUE` return `MIN_VALUE` itself. `Math.addExact` / `multiplyExact` throw instead of wrapping.

**Remember:** *Java never throws on `int`/`long` overflow. Use `long`, `Math.xxxExact`, or `BigInteger` for money/ids/time.*

---

### Q321. `0.1 + 0.2`, NaN and negative zero

```java
System.out.println(0.1 + 0.2);
System.out.println(0.1 + 0.2 == 0.3);
System.out.println(new java.math.BigDecimal("0.1").add(new java.math.BigDecimal("0.2")));
double nan = Double.NaN;
System.out.println(nan == nan);
System.out.println(Double.valueOf(nan).equals(nan));
System.out.println(0.0 == -0.0);
System.out.println(Double.valueOf(0.0).equals(-0.0));
System.out.println(1.0 / 0);
System.out.println(0.0 / 0);
```

**Output:**
```
0.30000000000000004
false
0.3
false
true
true
false
Infinity
NaN
```

**Why:** `double` stores binary fractions; 0.1 and 0.2 cannot be represented exactly, so the sum is slightly off. Use `BigDecimal` (built from **Strings**) for money, or compare with a tolerance. `NaN != NaN` using `==` (IEEE 754), but `Double.equals` treats NaN as equal to itself so that collections work; likewise `0.0 == -0.0` is true but `Double.equals` says false. Floating-point division by zero does not throw: it gives `Infinity` or `NaN`.

**Remember:** *Never use `double` for money; use `Double.isNaN(x)` to test NaN; `==` and `equals` differ for doubles.*

---

### Q322. Narrowing casts: `(byte)`, `(int)` and `byte`/`short` compound assignment

```java
System.out.println((byte) 128);
System.out.println((byte) 200);
byte b = (byte) 0xFF;
System.out.println(b == 0xFF);
System.out.println(b & 0xFF);
byte big = 127;
big++;
System.out.println(big);
short s = 10;
s += 1;                 // compiles
// s = s + 1;           // does NOT compile: int cannot be assigned to short
System.out.println(s);
System.out.println((int) 3.99 + " " + (int) -3.99);
System.out.println((int) 1e20);
System.out.println((int) Double.NaN);
```

**Output:**
```
-128
-56
false
255
-128
11
3 -3
2147483647
0
```

**Why:** A `(byte)` cast keeps only the lowest 8 bits and reads them as signed, so 128 becomes −128 and 200 becomes −56. `b` holds −1; in `b == 0xFF` it is widened to the `int` −1 and compared to 255 → false; `b & 0xFF` gives the unsigned value 255. `s + 1` is an `int`, so `s = s + 1` needs a cast, but `s += 1` and `big++` have an **implicit narrowing cast** built in (and can silently overflow). Casting a double to `int` **truncates toward zero**, saturates at the `int` limits, and maps NaN to 0.

**Remember:** *`+=`, `++` hide a cast; `x & 0xFF` converts a signed byte to its unsigned value.*

---

### Q323. Initialization order with inheritance (and an overridden method called from a constructor)

```java
class Parent {
    static { System.out.println("1 Parent static"); }
    { System.out.println("3 Parent instance block"); }
    Parent() { System.out.println("4 Parent ctor"); show(); }
    void show() { System.out.println("Parent.show"); }
}
class Child extends Parent {
    static { System.out.println("2 Child static"); }
    private String name = "Alice";
    { System.out.println("5 Child instance block"); }
    Child() { System.out.println("6 Child ctor, name=" + name); }
    @Override void show() { System.out.println("Child.show name=" + name); }
}
// in main:
new Child();
System.out.println("--- second object ---");
new Child();
```

**Output:**
```
1 Parent static
2 Child static
3 Parent instance block
4 Parent ctor
Child.show name=null
5 Child instance block
6 Child ctor, name=Alice
--- second object ---
3 Parent instance block
4 Parent ctor
Child.show name=null
5 Child instance block
6 Child ctor, name=Alice
```

**Why:** Order is: parent static → child static (once, when the class loads) → parent instance initializers → parent constructor → child instance initializers and field initializers → child constructor body. The parent constructor calls `show()`, which is **virtual**, so the *child's* version runs — but the child's field `name` is not initialised yet, so it is `null`. Static blocks do not run again for the second object.

**Remember:** *Never call overridable methods from a constructor — the subclass runs before its own fields are initialised.*

---

### Q324. Static method hiding vs overriding, and field hiding

```java
class A {
    String name = "A-field";
    static String who() { return "A"; }
    String me() { return "A.me"; }
}
class B extends A {
    String name = "B-field";
    static String who() { return "B"; }
    @Override String me() { return "B.me"; }
}
// in main:
A x = new B();
System.out.println(x.who());
System.out.println(x.name);
System.out.println(x.me());
System.out.println(((B) x).name);
```

**Output:**
```
A
A-field
B.me
B-field
```

**Why:** Only **instance methods** are polymorphic (resolved at runtime by the real object type). **Static methods** and **fields** are resolved at **compile time** using the *declared* type of the variable (`A`). So `x.who()` and `x.name` use `A`'s versions; casting to `B` changes which field is read. A static method in the subclass *hides* the parent one; it does not override it (`@Override` on it would not compile).

**Remember:** *Methods are polymorphic; fields and static methods are not.*

---

### Q325. Overload resolution: `null`, widening, boxing and varargs

```java
static void f(Object o) { System.out.println("f Object"); }
static void f(String s) { System.out.println("f String"); }
static void g(long x)    { System.out.println("g long"); }
static void g(Integer x) { System.out.println("g Integer"); }
static void g(int... x)  { System.out.println("g varargs"); }
static void h(Object o)  { System.out.println("h Object"); }
static void h(int... x)  { System.out.println("h varargs"); }
// in main:
f(null);
Object o = "str";
f(o);
g(5);
g(Integer.valueOf(5));
h(5);
```

**Output:**
```
f String
f Object
g long
g Integer
h Object
```

**Why:** Overloads are chosen at **compile time** by the declared types. `null` fits both, and the **most specific** type (`String`) wins; with `f(String)` and `f(StringBuilder)` the call `f(null)` would not compile (ambiguous). `f(o)` uses the declared type `Object`, even though the object is a String. The compiler tries in three phases: **(1) widening / subtyping, no boxing → (2) boxing/unboxing → (3) varargs**. So `g(5)` picks `long` (widening) before boxing to `Integer`, and `h(5)` picks `Object` (boxing) before varargs.

**Remember:** *Overloading = compile-time, by static type. Priority: widening → boxing → varargs.*

---

### Q326. `finally` and `return`

```java
static int a() { try { return 1; } finally { return 2; } }
static int b() { int x = 1; try { return x; } finally { x = 2; } }
static int c() { try { throw new RuntimeException("boom"); } finally { return 3; } }
static StringBuilder d() {
    StringBuilder sb = new StringBuilder("hi");
    try { return sb; } finally { sb.append("!"); }
}
// in main:
System.out.println(a());
System.out.println(b());
System.out.println(c());
System.out.println(d());
```

**Output:**
```
2
1
3
hi!
```

**Why:** `finally` always runs, and a `return` inside it **overrides** the earlier return — and even **swallows an exception** (`c()` never throws). In `b()`, the return value (1) is copied *before* `finally` runs, so changing the local `x` afterwards has no effect. In `d()` the *reference* is copied, but both point to the same object, so the append is visible. (A `finally` block is skipped only if the JVM exits, e.g. `System.exit()`, or the thread dies.)

**Remember:** *Never `return` or throw from `finally`; it hides the real result or exception. Return value is captured before `finally` — but mutable objects can still change.*

---

### Q327. Exception in a static initializer

```java
class Config {
    static final int V = compute();
    static int compute() { throw new IllegalStateException("bad config"); }
    static void hello() { }
}
// in main:
try { Config.hello(); } catch (Throwable t) {
    System.out.println(t.getClass().getSimpleName() + " / cause: " + t.getCause());
}
try { Config.hello(); } catch (Throwable t) {
    System.out.println(t.getClass().getSimpleName() + ": " + t.getMessage());
}
```

**Output:**
```
ExceptionInInitializerError / cause: java.lang.IllegalStateException: bad config
NoClassDefFoundError: Could not initialize class Config
```

**Why:** The first use of the class runs the static initializer; the exception is wrapped in `ExceptionInInitializerError`. The class is then marked **unusable** for the rest of the JVM's life, so every later use throws `NoClassDefFoundError` — which hides the original cause and confuses people reading logs. Always look for the *first* error in the log.

**Remember:** *A failed static init kills the class permanently; the real cause is only in the first `ExceptionInInitializerError`.*

---

### Q328. `HashSet`/`HashMap` lose objects: mutable keys and `equals` without `hashCode`

```java
class P {                       // equals + hashCode on mutable field x
    int x;
    P(int x) { this.x = x; }
    @Override public boolean equals(Object o) { return o instanceof P p && p.x == x; }
    @Override public int hashCode() { return x; }
}
class K {                       // equals only, no hashCode
    final int id;
    K(int id) { this.id = id; }
    @Override public boolean equals(Object o) { return o instanceof K k && k.id == id; }
}
// in main:
Set<P> set = new HashSet<>();
P p = new P(1);
set.add(p);
p.x = 2;
System.out.println(set.contains(p));
System.out.println(set.contains(new P(2)));
System.out.println(set.remove(p));
System.out.println(set.size());
Map<K, String> map = new HashMap<>();
map.put(new K(1), "one");
System.out.println(map.get(new K(1)));
System.out.println(new K(1).equals(new K(1)));
```

**Output:**
```
false
false
false
1
null
true
```

**Why:** A hash collection stores the element in the bucket for its hash **at insertion time**. After `p.x = 2` the hash changes, so lookups go to a different bucket and never find it — yet it is still inside (`size()` is 1): a **memory leak**. For `K`, `equals` says two objects are equal but they inherit the identity-based `Object.hashCode()`, so they almost always land in different buckets and `get` returns `null`.

**Remember:** *Override `equals` and `hashCode` together; keep key fields immutable.*

---

### Q329. `list.remove(int)` vs `remove(Object)`

```java
List<Integer> list = new ArrayList<>(List.of(10, 20, 30, 1, 2));
list.remove(1);
System.out.println(list);
list.remove(Integer.valueOf(1));
System.out.println(list);
Set<Short> set = new HashSet<>();
for (short i = 0; i < 10; i++) {
    set.add(i);
    set.remove(i - 1);
}
System.out.println(set.size());
```

**Output:**
```
[10, 30, 1, 2]
[10, 30, 2]
10
```

**Why:** `List` has two methods: `remove(int index)` and `remove(Object o)`. A plain `int` argument matches `remove(int)` exactly (no boxing needed), so `remove(1)` removes the element at **index 1** (20). To remove the *value* 1 you must box it. In the `Set<Short>` example `i - 1` is an `int`, which gets boxed to `Integer`, never equal to a `Short`, so nothing is removed and the size is 10, not 1.

**Remember:** *On `List<Integer>`, `remove(int)` = by index, `remove(Integer.valueOf(x))` = by value. Boxed types must match exactly.*

---

### Q330. `Arrays.asList` and `List.of` surprises

```java
Integer[] arr = {1, 2, 3};
List<Integer> view = Arrays.asList(arr);
view.set(0, 99);
System.out.println(arr[0]);
arr[1] = 50;
System.out.println(view);
try { view.add(4); } catch (UnsupportedOperationException e) { System.out.println("UOE"); }
int[] prim = {1, 2, 3};
System.out.println(Arrays.asList(prim).size());
try { List.of(1, 2).contains(null); } catch (NullPointerException e) { System.out.println("NPE"); }
```

**Output:**
```
99
[99, 50, 3]
UOE
1
NPE
```

**Why:** `Arrays.asList` returns a **fixed-size view backed by the array**: `set` writes through to the array (and vice versa), but `add`/`remove` throw `UnsupportedOperationException`. With a primitive `int[]`, generics cannot hold `int`, so the whole array becomes **one element** of type `int[]` → size 1. `List.of` creates an immutable list that also rejects `null` — even in `contains(null)`.

**Remember:** *`Arrays.asList` = fixed-size view; use `new ArrayList<>(Arrays.asList(...))` for a real list. Never pass a primitive array to it.*

---

### Q331. `ConcurrentModificationException` and `subList`

```java
List<String> a = new ArrayList<>(List.of("a", "b", "c"));
for (String s : a) if (s.equals("b")) a.remove(s);
System.out.println(a);

List<String> b = new ArrayList<>(List.of("a", "b", "c", "d"));
try {
    for (String s : b) if (s.equals("b")) b.remove(s);
} catch (ConcurrentModificationException e) { System.out.println("CME"); }
b.removeIf(s -> s.equals("b"));
System.out.println(b);

List<Integer> base = new ArrayList<>(List.of(1, 2, 3, 4, 5));
List<Integer> sub = base.subList(1, 4);
sub.set(0, 99);
System.out.println(base);
base.add(6);
try { System.out.println(sub.size()); }
catch (ConcurrentModificationException e) { System.out.println("CME on subList"); }
```

**Output:**
```
[a, c]
CME
[a, c, d]
[1, 99, 3, 4, 5]
CME on subList
```

**Why:** The for-each loop uses an iterator that checks `hasNext()` (`cursor != size`) *before* it checks for modification. Removing the **second-to-last** element makes `cursor == size`, so the loop just ends — no exception, by luck. With 4 elements the same code throws CME. The safe ways are `Iterator.remove()`, `removeIf`, or iterating a copy. `subList` is a **view** of the parent: writes go through to the parent, and if the parent is changed structurally (add/remove) afterwards, the sublist is invalid and throws CME.

**Remember:** *Don't modify a collection while iterating it (CME is "fail-fast, best effort"). `subList` is a view, not a copy.*

---

### Q332. Comparator by subtraction overflows

```java
List<Integer> list = new ArrayList<>(List.of(-1, Integer.MAX_VALUE));
list.sort((a, b) -> a - b);
System.out.println(list);
list.sort(Integer::compare);
System.out.println(list);
System.out.println(Integer.MAX_VALUE - (-1));
```

**Output:**
```
[2147483647, -1]
[-1, 2147483647]
-2147483648
```

**Why:** `a - b` overflows when the numbers have very different signs: `MAX_VALUE - (-1)` wraps to a **negative** number, so the comparator claims MAX is *smaller* than −1 and the sort puts them in the wrong order. It can also violate the comparator contract and cause "Comparison method violates its general contract!" with `TimSort`. `Integer.compare` (or `Comparator.comparingInt`) never overflows.

**Remember:** *Never write a comparator as `a - b`. Use `Integer.compare`, `Comparator.comparing...`.*

---

### Q333. `switch`: fall-through, `default` placement and `null`

```java
int n = 2;
switch (n) {
    case 1: System.out.println("one");
    case 2: System.out.println("two");
    case 3: System.out.println("three"); break;
    default: System.out.println("default");
}
switch (5) {
    default: System.out.println("default");
    case 1: System.out.println("one");
}
String s = null;
try {
    switch (s) { case "a": System.out.println("a"); break; default: System.out.println("other"); }
} catch (NullPointerException e) { System.out.println("NPE"); }
switch (n) {
    case 1 -> System.out.println("one");
    case 2 -> System.out.println("two");
    default -> System.out.println("default");
}
```

**Output:**
```
two
three
default
one
NPE
two
```

**Why:** In the old `case X:` form, execution **falls through** into the next case until a `break`. So `n = 2` prints `two` and `three`. `default` need not be last; when it is matched, execution still falls through into the cases below it. Switching on a `null` String (or enum/wrapper) throws NPE. The arrow form (`->`, Java 14+) has **no fall-through**, which is why it is preferred.

**Remember:** *Forget `break` = fall-through. Prefer `case ->` and switch expressions.*

---

### Q334. Labeled `break`/`continue` and `&&` vs `&`

```java
outer:
for (int i = 0; i < 3; i++) {
    for (int j = 0; j < 3; j++) {
        if (j == 2) continue outer;
        if (i == 2) break outer;
        System.out.println(i + "," + j);
    }
}
static boolean t(String s) { System.out.print(s + " "); return true; }
static boolean f(String s) { System.out.print(s + " "); return false; }
// in main:
boolean r1 = f("A") && t("B");
System.out.println("-> " + r1);
boolean r2 = f("A") & t("B");
System.out.println("-> " + r2);
String s = null;
try {
    if (s != null & s.length() > 0) System.out.println("x");
} catch (NullPointerException e) { System.out.println("NPE"); }
```

**Output:**
```
0,0
0,1
1,0
1,1
A -> false
A B -> false
NPE
```

**Why:** `continue outer` jumps to the next iteration of the *outer* loop; `break outer` leaves both loops (at `i == 2`, before anything is printed). `&&` is **short-circuit**: if the left side is false, the right side is never evaluated. Single `&` on booleans always evaluates **both** sides, so `s.length()` runs on `null` and throws NPE.

**Remember:** *Use `&&`/`||` for conditions (safe null-guards); `&`/`|` always evaluate both sides.*

---

### Q335. Lambda capturing: locals vs fields, and `this`

```java
class Demo {
    int field = 1;
    Runnable make() {
        int local = 1;
        Runnable r = () -> System.out.println(field + " " + local);
        field = 2;
        // local = 2;   // compile error: must be (effectively) final
        return r;
    }
    void thisTest() {
        Runnable lam  = () -> System.out.println(this.getClass().getName());
        Runnable anon = new Runnable() {
            public void run() { System.out.println(this.getClass().getName()); }
        };
        lam.run();
        anon.run();
    }
    public static void main(String[] args) {
        Demo q = new Demo();
        q.make().run();
        q.thisTest();
        List<Supplier<Integer>> list = new ArrayList<>();
        for (int i : new int[]{0, 1, 2}) list.add(() -> i);
        for (Supplier<Integer> s : list) System.out.print(s.get() + " ");
        System.out.println();
    }
}
```

**Output:**
```
2 1
Demo
Demo$1
0 1 2 
```

**Why:** A lambda **copies** captured local variables, so they must be *effectively final*. A **field** is read through `this` at run time, so the lambda sees the latest value (2). In a lambda, `this` means the **enclosing** object (`Demo`); inside an anonymous class, `this` is the anonymous object itself (`Demo$1`). In an enhanced-for loop, each iteration has its own new variable `i`, so capturing it is legal; the classic `for (int i = 0; ...; i++)` variable is not effectively final and cannot be captured.

**Remember:** *Lambdas capture values of locals (effectively final) but live references to fields; lambda `this` = enclosing class.*

---

### Q336. Stream laziness and stream reuse

```java
Stream.of(1, 2, 3).peek(x -> System.out.println("peek " + x)).map(x -> x * 2);
System.out.println("after pipeline");
Optional<Integer> first = Stream.of(1, 2, 3, 4)
    .filter(x -> { System.out.println("filter " + x); return x % 2 == 0; })
    .map(x -> { System.out.println("map " + x); return x * 10; })
    .findFirst();
System.out.println(first.get());
Stream<Integer> s = Stream.of(1, 2);
System.out.println(s.count());
try { s.count(); } catch (IllegalStateException e) { System.out.println(e.getMessage()); }
```

**Output:**
```
after pipeline
filter 1
filter 2
map 2
20
2
stream has already been operated upon or closed
```

**Why:** Intermediate operations (`filter`, `map`, `peek`) are **lazy** — nothing runs until a **terminal** operation is called, so the first pipeline prints no "peek" lines. Elements flow through the whole pipeline **one at a time** (`filter 1`, then `filter 2`, `map 2`), not stage by stage, and a short-circuiting terminal op like `findFirst` stops early (3 and 4 are never touched). A stream can be consumed **only once**; a second terminal operation throws `IllegalStateException`.

**Remember:** *No terminal op = no work. One stream, one use. `peek` is for debugging, not for side effects you rely on.*

---

### Q337. `Optional.orElse` is eager, `orElseGet` is lazy

```java
static String expensive() { System.out.println("expensive() called"); return "default"; }
// in main:
System.out.println(Optional.of("value").orElse(expensive()));
System.out.println(Optional.of("value").orElseGet(() -> expensive()));
```

**Output:**
```
expensive() called
value
value
```

**Why:** `orElse(x)` takes a **value**, so Java evaluates `expensive()` *before* calling `orElse`, even when the Optional has a value. `orElseGet(supplier)` only calls the supplier if the Optional is empty. When the fallback is a DB call, remote call, or object creation, `orElse` wastes work (and can cause side effects).

**Remember:** *Cheap constant → `orElse`; anything expensive or with side effects → `orElseGet`.*

---

### Q338. `ThreadLocal`, `InheritableThreadLocal` and thread-pool leaks

```java
ThreadLocal<String> tl = new ThreadLocal<>();
InheritableThreadLocal<String> itl = new InheritableThreadLocal<>();
tl.set("main-tl");
itl.set("main-itl");
Thread t = new Thread(() -> System.out.println(tl.get() + " " + itl.get()));
t.start();
t.join();
ExecutorService pool = Executors.newSingleThreadExecutor();
pool.submit(() -> tl.set("leaked")).get();
System.out.println(pool.submit(() -> tl.get()).get());
pool.shutdown();
```

**Output:**
```
null main-itl
leaked
```

**Why:** A plain `ThreadLocal` value belongs to one thread only; a new child thread sees `null`. `InheritableThreadLocal` **copies** the parent's value into the child when the child is *created* (later parent changes are not seen). In a pool, threads are **reused**, so a value set by task 1 is still there for task 2 — a classic source of data leaking between users/requests. (Inheritable values also only get copied when the pool creates a new thread, which makes them unreliable with pools.) Always `remove()` in a `finally` block.

**Remember:** *`ThreadLocal` + thread pool = call `remove()` in `finally`. Prefer passing context explicitly (or scoped values in newer Java versions).*

---

### Q339. `start()` vs `run()`, and starting a thread twice

```java
Thread t = new Thread(
    () -> System.out.println("running in " + Thread.currentThread().getName()), "worker");
t.run();
t.start();
t.join();
try { t.start(); }
catch (IllegalThreadStateException e) { System.out.println("IllegalThreadStateException"); }
```

**Output:**
```
running in main
running in worker
IllegalThreadStateException
```

**Why:** `run()` is an ordinary method call executed by the **caller's** thread (here `main`) — no new thread. `start()` asks the JVM to create a new thread that calls `run()`. A `Thread` object can be started **only once**, even after it has finished; the second `start()` throws `IllegalThreadStateException`. To run something again, create a new `Thread` (or submit to an executor).

**Remember:** *`start()` = new thread; `run()` = just a method. A thread cannot be restarted.*

---

### Q340. Lost updates: `synchronized` on the wrong object and `volatile` counters (non-deterministic)

```java
class Counter {
    int count;
    void inc() { synchronized (new Object()) { count++; } }   // a NEW lock every call
}
static volatile int vol = 0;

// two threads each run c.inc() 1_000_000 times;
// two other threads each run vol++ 1_000_000 times
```

**Output (non-deterministic; one real run):**
```
synchronized(new Object()): 1950236
volatile++: 1449010
```
Expected 2,000,000 each — you will almost always get **less**, and a different number on every run.

**Why:** A lock only works if all threads use the **same** lock object. `synchronized (new Object())` gives each caller its own private lock, so nothing is excluded. Likewise `synchronized` instance methods on **different instances** lock different objects. `volatile` guarantees **visibility** (other threads see the latest value) but not **atomicity**: `vol++` is three steps (read, add, write), and two threads can read the same value and both write back the same result. Fix with `synchronized` on a shared lock, `AtomicInteger`, or `LongAdder`.

**Remember:** *Same shared lock, or it is not a lock. `volatile` = visibility, not atomicity. For counters use `AtomicInteger`/`LongAdder`.*

---

### Q341. `clone()` is a shallow copy

```java
class Team implements Cloneable {
    String name = "A";
    List<String> members = new ArrayList<>();
    int[] scores = {1, 2};
    @Override public Team clone() {
        try { return (Team) super.clone(); }
        catch (CloneNotSupportedException e) { throw new AssertionError(e); }
    }
}
// in main:
Team a = new Team();
Team b = a.clone();
b.members.add("x");
b.scores[0] = 99;
b.name = "B";
System.out.println(a.members + " " + a.scores[0] + " " + a.name);
System.out.println((a != b) + " " + (a.members == b.members));
```

**Output:**
```
[x] 99 A
true true
```

**Why:** `Object.clone()` copies the object's fields **bit by bit**: primitives are copied, but object fields copy only the **reference**. So the clone shares the same `List` and the same array as the original — changing them through `b` changes `a` too. Reassigning `b.name` only changes `b`'s own reference (and Strings are immutable anyway). For a **deep copy**, copy the mutable fields yourself, or use a copy constructor / factory (preferred over `Cloneable`, which is considered a flawed design).

**Remember:** *`clone()` is shallow. Prefer copy constructors; deep-copy mutable fields explicitly.*

---

### Q342. Static initialization order and lazy class loading

```java
class Singleton {
    static final Singleton INSTANCE = new Singleton();
    static int counter = 5;
    static int snapshot;
    Singleton() { counter++; snapshot = counter; }
}
class Lazy {
    static { System.out.println("Lazy loaded"); }
    static final int K = 10;
    static int v = 20;
}
// in main:
System.out.println(Singleton.counter + " " + Singleton.snapshot);
System.out.println(Lazy.K);
System.out.println("--");
System.out.println(Lazy.v);
```

**Output:**
```
5 1
10
--
Lazy loaded
20
```

**Why:** Static fields are initialised **in textual order**. `INSTANCE` is created first, while `counter` still has its default value 0, so the constructor makes it 1 and `snapshot = 1`. Then `counter = 5` runs and overwrites it, while `snapshot` has no initializer and stays 1. Moving the `INSTANCE` line below the other statics changes the result. Also, `Lazy.K` is a **compile-time constant**: the compiler inlines `10`, so the class is **not even loaded**. Reading the non-constant `Lazy.v` triggers class initialization.

**Remember:** *Static initializers run top to bottom on first use; compile-time constants don't trigger class loading.*

---

### Q343. Arrays gotchas: `char[]` printing, `Object o = new int[0]`, covariance

```java
char[] cs = {'h', 'i'};
System.out.println(cs);
System.out.println(String.valueOf(cs));
Object asObj = cs;
System.out.println(String.valueOf(asObj).startsWith("[C@"));
System.out.println(("" + cs).startsWith("[C@"));
Object o = new int[0];
System.out.println(o instanceof int[]);
System.out.println(o instanceof Object[]);
System.out.println(new Integer[0] instanceof Object[]);
System.out.println(o.getClass().getName());
Object[] objs = new String[1];
try { objs[0] = 1; }
catch (ArrayStoreException e) { System.out.println("ArrayStoreException: " + e.getMessage()); }
```

**Output:**
```
hi
hi
true
true
true
false
true
[I
ArrayStoreException: java.lang.Integer
```

**Why:** `println(char[])` and `String.valueOf(char[])` are special overloads that print the **characters**. If the same array is seen as `Object` (`String.valueOf(Object)`, or `"" + cs`), you get `Object.toString()` — a type tag and hash like `[C@1b6d3586` (the hash varies, so only the prefix is shown here). A `new int[0]` **is an `Object`**, but it is *not* an `Object[]` — primitive arrays are not arrays of objects (`[I` is its class name). **Object arrays are covariant**: a `String[]` can be used as `Object[]`, but the real array remembers its type, so storing an `Integer` fails at **runtime** with `ArrayStoreException`. (Generics are invariant and avoid this problem.)

**Remember:** *Overloads are picked by static type. Arrays are covariant and checked at runtime; generics are invariant and checked at compile time.*

---

### Q344. Generic type erasure

```java
List<String> strs = new ArrayList<>();
List<Integer> ints = new ArrayList<>();
System.out.println(strs.getClass() == ints.getClass());
System.out.println(strs.getClass().getName());
List raw = strs;          // raw type: compiles with a warning
raw.add(42);
System.out.println(strs.size());
try { String s = strs.get(0); }
catch (ClassCastException e) { System.out.println("ClassCastException"); }
```

**Output:**
```
true
java.util.ArrayList
1
ClassCastException
```

**Why:** Generics exist only at **compile time**. The compiler checks types, then **erases** them: at runtime both lists are plain `ArrayList` holding `Object`s. Using a raw type bypasses the checks and lets an `Integer` into a `List<String>` ("heap pollution"). The error only appears later, at the compiler-inserted cast in `strs.get(0)`, far from the real bug. Erasure is also why you cannot write `new T()`, `instanceof List<String>`, or overload `m(List<String>)` with `m(List<Integer>)`.

**Remember:** *Generics = compile-time safety only. Avoid raw types; pass `Class<T>` when you need the type at runtime.*

---

### Q345. `null` gotchas: `instanceof`, casts and static calls through `null`

```java
class Util { static void hi() { System.out.println("hi"); } }
// in main:
String s = null;
System.out.println(s instanceof String);
Object o = (String) null;
System.out.println(o == null);
Util u = null;
u.hi();
System.out.println(s + "x");
```

**Output:**
```
false
true
hi
nullx
```

**Why:** `instanceof` is **always false for `null`**, so a separate null check is not needed before it. Casting `null` is legal. `u.hi()` does not throw because static methods are bound to the **declared type**, not to the object — the reference is never dereferenced (bad style, but legal). String concatenation turns `null` into the text `"null"` instead of failing.

**Remember:** *`null instanceof X` is false; static calls through a null reference work; `"" + null` is `"null"`.*

---

### Q346. Division and modulo with negatives and zero

```java
System.out.println(-7 / 2);
System.out.println(-7 % 2);
System.out.println(7 % -2);
System.out.println(Math.floorMod(-7, 2));
System.out.println(Math.floorDiv(-7, 2));
System.out.println(-7 >> 1);
System.out.println(-7 >>> 28);
double d = 5 / 2;
System.out.println(d);
System.out.println(5 / 2.0);
try { System.out.println(5 / 0); }
catch (ArithmeticException e) { System.out.println(e.getMessage()); }
System.out.println(5.0 / 0);
System.out.println(5 % 0.0);
```

**Output:**
```
-3
-1
1
1
-4
-4
15
2.0
2.5
/ by zero
Infinity
NaN
```

**Why:** Integer division **truncates toward zero** (−3.5 → −3), and the remainder takes the **sign of the dividend** (left operand), so `-7 % 2 = -1`. `floorDiv`/`floorMod` round toward negative infinity and give a non-negative result for a positive divisor — use `floorMod(hash, n)` for array/bucket indexes. `>>` keeps the sign (arithmetic shift, same as floor-divide by 2 here), `>>>` shifts in zeros so a negative number becomes large positive. `5 / 2` is integer division → 2, converted to `2.0` only afterwards. Integer `/ 0` throws; floating-point `/ 0` gives `Infinity`/`NaN`.

**Remember:** *`%` can be negative in Java — use `Math.floorMod` for indexes. Integer divide by zero throws, double divide by zero doesn't.*

---

# 13. Rapid-Fire Round & Last-Minute Cheat Sheet

## Rapid-Fire Round (One-Line Answers)

> Practise saying each answer in **one breath**. If the interviewer wants more, they will ask — then expand with an example or trade-off.

### OOP & Language Basics

| Question | One-line answer |
|---|---|
| Abstract class vs interface? | Abstract class: state + constructors + partial implementation, single inheritance; interface: contract (with `default`/`static` methods), a class can implement many. |
| Overloading vs overriding? | Overloading = same name, different parameters, resolved at **compile time**; overriding = subclass redefines an instance method, resolved at **runtime**. |
| Can you override a `static` or `private` method? | No — static methods are *hidden*, private methods are not inherited. |
| What does `final` mean on a class / method / variable? | Class can't be extended; method can't be overridden; variable can't be reassigned (an object it references can still change). |
| `==` vs `equals()`? | `==` compares references (or primitive values); `equals()` compares logical equality, as defined by the class. |
| What is the `equals`/`hashCode` contract? | Equal objects must have equal hash codes; unequal hash codes mean unequal objects; keep both consistent over time. |
| Why is `String` immutable? | Safe sharing, thread safety, hash caching, String pool and security. |
| `String` vs `StringBuilder` vs `StringBuffer`? | Immutable vs mutable & fast vs mutable & synchronized (rarely needed now). |
| Pass-by-value or pass-by-reference? | Always **pass-by-value**; for objects, the value copied is the reference. |
| Composition vs inheritance? | "Has-a" is more flexible and testable than "is-a"; prefer composition unless there is a true is-a relationship. |
| What is a `record`? | Java 16+ concise immutable data carrier; auto-generates constructor, accessors, `equals`, `hashCode`, `toString`. |
| What are `sealed` classes? | Java 17+: restrict which classes may extend/implement a type (`permits`) — great with pattern matching. |
| What is the diamond problem and how does Java handle it? | Two interfaces give the same default method; the class must override and may choose with `A.super.method()`. |
| `static` nested vs inner class? | Static nested has no hidden reference to the outer instance; inner class holds one (a common memory-leak source). |

### Collections & Generics

| Question | One-line answer |
|---|---|
| How does `HashMap` work? | Array of buckets; index from `hash(key)`; collisions use a linked list that becomes a red-black tree when long (≥ 8 entries and table ≥ 64). |
| Why must `HashMap` capacity be a power of two? | The bucket index is computed by a fast bit-mask `(n - 1) & hash` instead of modulo. |
| `HashMap` vs `Hashtable` vs `ConcurrentHashMap`? | Not thread-safe + allows one null key; fully synchronized legacy, no nulls; scalable concurrent map, no nulls. |
| `HashMap` vs `TreeMap` vs `LinkedHashMap`? | Unordered O(1); sorted by key O(log n); insertion (or access) order O(1). |
| `ArrayList` vs `LinkedList`? | `ArrayList` wins almost always (cache friendly, O(1) index access); `LinkedList` only for cheap add/remove at both ends — and `ArrayDeque` beats it there. |
| `List` vs `Set` vs `Map`? | Ordered with duplicates; unique elements; key → value pairs. |
| `Comparable` vs `Comparator`? | Natural order inside the class (`compareTo`) vs external, pluggable ordering (`compare`). |
| Fail-fast vs fail-safe iterators? | Fail-fast throws `ConcurrentModificationException` on structural change; fail-safe (e.g. `CopyOnWriteArrayList`, `ConcurrentHashMap`) work on a snapshot or tolerate changes. |
| How do you make a collection read-only? | `List.of(...)` / `Collections.unmodifiableList(...)` (the latter is only a view); `List.copyOf` for an immutable copy. |
| `Set.of`/`List.of`/`Map.of` features? | Immutable, no `null`s, `Set.of`/`Map.of` throw on duplicates, and iteration order is unspecified. |
| What is type erasure? | The compiler checks generics, then removes the type arguments; at runtime they are just `Object`/bounds. |
| `List<Object>` vs `List<?>`? | `List<Object>` is *not* a supertype of `List<String>`; `List<?>` is a read-only-ish list of unknown type. |
| What is PECS? | *Producer Extends, Consumer Super*: read from `? extends T`, write into `? super T`. |
| `Collections.synchronizedMap` vs `ConcurrentHashMap`? | One lock around the whole map vs fine-grained locking/CAS; compound operations still need `computeIfAbsent`/`merge`. |
| What does `computeIfAbsent` give you? | Atomic "get or create" in a `ConcurrentHashMap`, avoiding check-then-act races. |
| Which collection for LRU cache? | `LinkedHashMap` with `accessOrder = true` and an overridden `removeEldestEntry`. |
| Which structure for a priority queue / top-K? | `PriorityQueue` (binary heap) — poll is O(log n). |

### Exceptions

| Question | One-line answer |
|---|---|
| Checked vs unchecked exceptions? | Checked must be declared/handled (`IOException`); unchecked are `RuntimeException`s and `Error`s. |
| `Error` vs `Exception`? | `Error` signals JVM-level problems (OOM, StackOverflow) — don't catch them normally. |
| `throw` vs `throws`? | `throw` raises an exception object; `throws` declares what a method may throw. |
| Does `finally` always run? | Almost: not after `System.exit()`, a JVM crash, or if the thread is killed. |
| What is try-with-resources? | Auto-closes `AutoCloseable` resources; exceptions from `close()` are attached as **suppressed**. |
| `NoClassDefFoundError` vs `ClassNotFoundException`? | Class was present at compile time but missing/failed to load at runtime vs. class not found by name during dynamic loading. |
| Should you catch `Throwable` or `Exception` broadly? | No — catch specific exceptions, and never swallow them silently; log or rethrow with cause. |
| How do you preserve the root cause? | Pass the original exception as `cause` to the new exception. |
| Why not use exceptions for flow control? | They are slow (stack trace capture) and hide the normal logic. |

### JVM, Memory & GC

| Question | One-line answer |
|---|---|
| Which memory areas does the JVM have? | Heap (objects), thread stacks (frames), Metaspace (class metadata, native memory), code cache, and direct buffers. |
| What is stored on the stack vs heap? | Stack: local variables and references per thread; heap: all objects, shared. |
| When is an object eligible for GC? | When it is no longer reachable from any GC root (thread stacks, static fields, JNI refs…). |
| What are GC roots? | Local variables on stacks, static fields, active threads, JNI references. |
| Young vs old generation? | Most objects die young (minor GC, cheap); survivors are promoted to the old generation (major/mixed GC, costlier). |
| Default GC in modern Java? | G1 (since Java 9). |
| G1 vs ZGC vs Parallel? | G1: balanced default; ZGC: very low pause times, large heaps; Parallel: max throughput, longer pauses. |
| What causes `OutOfMemoryError`? | Heap exhaustion, Metaspace exhaustion, too many threads, or direct memory — read the message after "OutOfMemoryError:". |
| What is a memory leak in Java? | Objects still reachable but no longer needed (static collections, unremoved listeners, `ThreadLocal`s, caches without eviction). |
| Strong / soft / weak / phantom references? | Never collected while strongly reachable; soft collected under memory pressure; weak collected at next GC; phantom for post-mortem cleanup. |
| How does class loading work? | Parent-first delegation: Bootstrap → Platform → Application; load, link (verify, prepare, resolve), initialize. |
| What does the JIT compiler do? | Compiles hot bytecode to native code at runtime (tiered C1 + C2) with optimizations like inlining and escape analysis. |
| Which tools diagnose a Java app? | `jcmd`, `jstack` (thread dumps), `jmap` (heap), JFR + JMC, VisualVM, and `-XX:+HeapDumpOnOutOfMemoryError`. |
| `StackOverflowError` cause? | Too deep (often infinite) recursion exhausting the thread stack. |

### Concurrency

| Question | One-line answer |
|---|---|
| `synchronized` vs `ReentrantLock`? | Built-in, auto-released monitor vs explicit lock with `tryLock`, timeouts, fairness and multiple `Condition`s. |
| What does `volatile` guarantee? | Visibility and ordering of reads/writes — **not** atomicity of compound actions like `i++`. |
| `sleep()` vs `wait()`? | `sleep` pauses the thread and keeps locks; `wait` releases the monitor and needs `notify`/timeout. |
| `Runnable` vs `Callable`? | `Callable` returns a value and can throw checked exceptions. |
| What are the four deadlock conditions? | Mutual exclusion, hold-and-wait, no preemption, circular wait — break one (e.g. always lock in the same order). |
| Thread pool task flow in `ThreadPoolExecutor`? | Core threads → queue → extra threads up to max → rejection policy. |
| Why avoid `Executors.newFixedThreadPool` blindly? | It uses an **unbounded** queue; prefer a configured `ThreadPoolExecutor` with a bounded queue. |
| `CountDownLatch` vs `CyclicBarrier`? | One-shot "wait until N events" vs reusable "all threads wait for each other". |
| `CompletableFuture` in one line? | Composable async pipeline (`thenApply`, `thenCompose`, `allOf`) — always pass your own executor and handle errors. |
| What are virtual threads? | Java 21+ lightweight JVM-managed threads for blocking, I/O-heavy code; cheap to create in large numbers. |
| What is a race condition? | Outcome depends on timing of unsynchronized access to shared mutable state. |
| What is the happens-before relationship? | Memory-model rule that guarantees one action's writes are visible to another (e.g. unlock → lock, volatile write → read, `start()`/`join()`). |
| `AtomicInteger` vs `LongAdder`? | CAS-based single value vs striped counters — better under high contention. |
| Is `ConcurrentHashMap` fully safe for "check then put"? | Individual calls are atomic, a sequence of calls is not — use `putIfAbsent`/`compute`/`merge`. |

### Streams & Functional Java

| Question | One-line answer |
|---|---|
| Intermediate vs terminal operations? | Intermediate are lazy and return a stream; terminal triggers the pipeline and produces a result. |
| `map` vs `flatMap`? | One-to-one transform vs one-to-many that flattens nested streams. |
| `Optional` best practice? | Use as a return type only; avoid in fields/parameters; use `orElseGet`, `map`, `ifPresentOrElse` instead of `get()`. |
| When to avoid parallel streams? | Small data, blocking I/O, shared mutable state, or when you can't control the common ForkJoin pool. |
| `Collectors.toMap` gotcha? | Throws `IllegalStateException` on duplicate keys unless you pass a merge function. |
| `Stream.toList()` vs `Collectors.toList()`? | Java 16+ `toList()` returns an unmodifiable list. |
| What is a functional interface? | An interface with exactly one abstract method (`@FunctionalInterface`), usable as a lambda target. |
| Method reference kinds? | Static (`Integer::parseInt`), bound instance (`obj::m`), unbound instance (`String::length`), constructor (`ArrayList::new`). |

### Java Versions

| Question | One-line answer |
|---|---|
| What are the LTS versions? | 8, 11, 17, 21, 25 (a new feature release every 6 months). |
| Main features of Java 8? | Lambdas, streams, `Optional`, default methods, `java.time`, `CompletableFuture`. |
| What came in Java 9–11? | Modules (9), `var` (10), `HttpClient`, `String` helpers like `isBlank`/`strip` (11), `List.of` (9). |
| What came in Java 14–17? | Switch expressions, text blocks, `record`s, pattern matching for `instanceof`, sealed classes (17). |
| What came in Java 21? | Virtual threads, record patterns, pattern matching for `switch`, sequenced collections, generational ZGC. |
| What happened to PermGen? | Removed in Java 8 and replaced by Metaspace (native memory). |
| What is `var`? | Local-variable type inference (Java 10); the type is still static, it is only inferred. |

### Design Patterns & Design Principles

| Question | One-line answer |
|---|---|
| Best way to write a Singleton? | Single-element `enum`, or the holder idiom; avoid double-checked locking unless the field is `volatile`. |
| Where do you see patterns in the JDK? | Decorator in `java.io` streams, Iterator in collections, Builder in `StringBuilder`, Factory in `List.of`/`Calendar.getInstance`, Strategy in `Comparator`. |
| Strategy vs State? | Strategy: caller picks the algorithm; State: the object changes its own behaviour as its state changes. |
| Builder vs telescoping constructors? | Builder gives readable, optional-parameter-friendly, immutable object construction. |
| What are SOLID principles? | Single responsibility, Open/closed, Liskov substitution, Interface segregation, Dependency inversion. |
| Dependency injection in one line? | Objects receive their collaborators instead of creating them — easier to test and swap. |
| What is an immutable class recipe? | `final` class, `private final` fields, no setters, defensive copies of mutable inputs/outputs. |

### I/O, Serialization & Testing

| Question | One-line answer |
|---|---|
| `InputStream` vs `Reader`? | Bytes vs characters (with an encoding — always specify one, e.g. UTF-8). |
| `java.io` vs `java.nio`? | Blocking stream-based vs buffers/channels/selectors (non-blocking) and the `Path`/`Files` API. |
| Best way to read a small file? | `Files.readString(path)` / `Files.readAllLines(path)`; for large files `Files.lines` or a `BufferedReader` in try-with-resources. |
| What does `transient` do? | Excludes a field from default Java serialization. |
| Why is Java serialization discouraged? | Security risks (deserialization attacks), fragile versioning, poor performance; prefer JSON/Protobuf and filter untrusted input. |
| What is `serialVersionUID`? | Version id for a serializable class; mismatch causes `InvalidClassException`. |
| JUnit 5 main parts? | Platform + Jupiter (API) + Vintage; `@Test`, `@ParameterizedTest`, `@BeforeEach`, `assertThrows`. |
| Mock vs stub vs spy? | Mock verifies interactions; stub returns canned answers; spy wraps a real object. |
| Test pyramid? | Many fast unit tests, fewer integration tests, very few end-to-end tests. |
| What makes a test good? | Fast, isolated, deterministic, readable, tests behaviour not implementation. |

---

## Numbers to Memorize

Only the values that are stable and commonly asked.

| Thing | Value |
|---|---|
| `HashMap` default capacity / load factor | 16 / 0.75 (resize doubles the table when size > capacity × load factor) |
| `HashMap` treeify / untreeify threshold | 8 / 6 (treeify only if table size ≥ 64, otherwise it resizes instead) |
| `ArrayList` default capacity / growth | 10 (allocated on first add) / grows by about 1.5× |
| `Vector` growth | doubles |
| `PriorityQueue` default initial capacity | 11 |
| `Integer` cache range | −128 to 127 (also `Byte`, `Short`, `Long`, and `Character` 0–127; `Boolean` has two cached values) |
| Primitive sizes | `byte` 8, `short` 16, `char` 16 (unsigned), `int` 32, `long` 64, `float` 32, `double` 64 bits |
| `int` range | −2,147,483,648 to 2,147,483,647 |
| `String.hashCode` multiplier | 31 |
| Default thread stack size (`-Xss`) | 1 MB on 64-bit Linux and Windows |
| Default max heap (`-Xmx`) | 1/4 of physical memory |
| Default initial heap (`-Xms`) | 1/64 of physical memory |
| G1 region size | 1 MB – 32 MB (power of 2, chosen from heap size; about 2048 regions targeted) |
| G1 default pause-time goal | 200 ms (`-XX:MaxGCPauseMillis`) |
| `MaxTenuringThreshold` (max age) | 15 |
| Metaspace default max size | Unlimited (grows in native memory unless `-XX:MaxMetaspaceSize` is set) |
| Compressed oops | Used for heaps up to roughly 32 GB |
| Default GC | G1 (Java 9+, on "server-class" machines) |
| LTS releases | 8, 11, 17, 21, 25 |
| Feature release cadence | Every 6 months (March and September) |
| Parallel stream default parallelism | Common ForkJoinPool with (cores − 1) workers |

---

## Complexity Cheat Sheet (Collections)

| Structure | get / contains | add | remove | Notes |
|---|---|---|---|---|
| `ArrayList` | O(1) by index / O(n) search | O(1) amortized at end; O(n) in the middle | O(n) | Contiguous memory, cache friendly |
| `LinkedList` | O(n) | O(1) at ends | O(1) at ends / O(n) to find a middle element | Rarely the best choice |
| `ArrayDeque` | O(1) at ends | O(1) at ends | O(1) at ends | Best stack/queue; no nulls |
| `HashMap` / `HashSet` | O(1) average | O(1) average | O(1) average | Worst case O(log n) with treeified buckets |
| `LinkedHashMap` / `LinkedHashSet` | O(1) average | O(1) average | O(1) average | Keeps insertion/access order |
| `TreeMap` / `TreeSet` | O(log n) | O(log n) | O(log n) | Sorted; `floor`, `ceiling`, range views |
| `PriorityQueue` | `peek` O(1); `contains` O(n) | `offer` O(log n) | `poll` O(log n); `remove(Object)` O(n) | Binary heap, iteration is not sorted |
| `ConcurrentHashMap` | O(1) average | O(1) average | O(1) average | No `null` keys/values |
| `CopyOnWriteArrayList` | O(1) by index | O(n) (copies the array) | O(n) | For read-mostly data |
| `EnumMap` / `EnumSet` | O(1) | O(1) | O(1) | Array/bitset based, very fast |

**Sorting:** `Arrays.sort(int[])` is a dual-pivot quicksort; `Collections.sort` / `List.sort` / `Arrays.sort(Object[])` use TimSort — stable, O(n log n), close to O(n) for nearly sorted data. Binary search is O(log n) and needs sorted input.

---

## Always-True One-Liners (Core Java)

- Override `equals` and `hashCode` together; never put mutable fields into either.
- Compare objects with `equals()`, not `==` (and compare wrappers carefully).
- Prefer immutability: `final` fields, unmodifiable collections, records.
- Program to interfaces (`List`, `Map`), not implementations.
- Prefer composition over inheritance; keep inheritance shallow.
- Never call overridable methods from a constructor.
- Use `BigDecimal` (from String) for money, never `double`.
- Use try-with-resources for every `AutoCloseable`.
- Never swallow exceptions; log once, or rethrow with the cause.
- Never return `null` for a collection — return an empty one; use `Optional` for "maybe a single value" return types.
- `StringBuilder` for string building in loops; never `+=`.
- Always specify the charset (`StandardCharsets.UTF_8`) and the `Locale` when it matters.
- Collections are not thread-safe by default; share only immutable or properly synchronized state.
- `volatile` is for visibility, not atomicity; use `Atomic*` or locks for compound actions.
- Acquire locks in a consistent order; keep critical sections small; never call alien code while holding a lock.
- Use bounded thread pools and bounded queues; name your threads.
- Always `remove()` a `ThreadLocal` when threads are pooled.
- Don't use `finalize()`; use try-with-resources or `Cleaner`.
- Streams: no side effects in lambdas, and no parallel streams for blocking work.
- Measure with a profiler/JMH before micro-optimizing; fix algorithms and I/O first.
- Heap dump on OOM (`-XX:+HeapDumpOnOutOfMemoryError`) and GC logs should be on in production.
- Don't rely on `System.gc()`, `hashCode()` order, or `HashMap` iteration order.
- Use `java.time`, never `Date`/`SimpleDateFormat` (the latter is not thread-safe).
- Prefer `enum` and sealed types over magic constants and `instanceof` chains.

---

## Top 25 Questions Most Likely Asked to a 9+ Year Java Developer

Each line names the topic; be ready to go **deep** (internals, trade-offs, a production story).

1. **`HashMap` internals** — hashing, buckets, collisions, treeification, resize, why `equals`/`hashCode` matter, null handling.
2. **`HashMap` vs `ConcurrentHashMap`** — how thread safety is achieved (CAS + bin-level locking), what is still not atomic, and why `Hashtable` is obsolete.
3. **`equals`/`hashCode` contract** — and the bugs from mutable or missing `hashCode`.
4. **Immutability and `String`** — pool, `intern`, why immutable, how to design an immutable class.
5. **Java Memory Model** — happens-before, `volatile`, visibility vs atomicity, safe publication.
6. **`synchronized` vs locks vs atomics** — when to use which, contention, lock ordering, deadlock detection.
7. **Thread pools and `ExecutorService`** — sizing for CPU vs I/O work, queue types, rejection policy, shutdown.
8. **`CompletableFuture` and async design** — composition, error handling, executors, timeouts.
9. **Virtual threads (Java 21+)** — what problem they solve, pinning, when *not* to use them.
10. **Garbage collection** — generations, G1 vs ZGC vs Parallel, tuning goals, reading GC logs.
11. **Memory leaks and `OutOfMemoryError`** — how you found one (heap dump, MAT), typical causes.
12. **Performance troubleshooting** — thread dumps, CPU spikes, high latency, profiling tools, JFR.
13. **JVM internals** — class loading, JIT compilation, escape analysis, memory areas.
14. **Exception handling strategy** — checked vs unchecked, custom exceptions, global handling, preserving causes.
15. **Streams and lambdas** — laziness, collectors, `flatMap`, parallel-stream pitfalls, readability vs loops.
16. **Generics** — wildcards, PECS, type erasure, bounded types.
17. **Java 8 → 17 → 21 features** — records, sealed classes, pattern matching, text blocks, switch expressions.
18. **OOP and SOLID in real code** — Liskov violations, abstract class vs interface, composition over inheritance.
19. **Design patterns you actually used** — Singleton (and its pitfalls), Builder, Strategy, Factory, Decorator, Observer.
20. **Immutability, `Optional` and null-safety** — API design that avoids `NullPointerException`s.
21. **Serialization and I/O** — Java serialization risks, NIO vs IO, large-file processing, charset handling.
22. **Testing strategy** — unit vs integration, mocking, testing concurrent code, flaky tests, test pyramid.
23. **Tricky output questions** — string pool, Integer cache, init order, overload resolution, finally.
24. **Designing for production** — logging, metrics, graceful shutdown, timeouts, retries, backpressure.
25. **Your biggest technical achievement / failure** — a real project, your decisions, trade-offs, measurable result (STAR).

---

## Questions to Ask the Interviewer

- Which Java version and GC are you running in production, and what is the upgrade plan (8 → 17 → 21 or later)?
- What does the on-call rotation and incident process look like?
- How do you handle performance, observability and load testing today?
- What is the test strategy and code-review culture? Is there a coverage or quality gate?
- How often do you deploy, and how long does a change take to reach production?
- What is the biggest piece of technical debt the team lives with, and how is it prioritised?
- How are architectural decisions made and recorded?
- What would success look like for me in the first 3 and 6 months?
- How does the team mentor juniors and share knowledge (design reviews, tech talks)?

---

## Final Reminders

1. **Structure every answer:** definition → how it works → trade-offs → a real example from your projects. Keep each to about two minutes.
2. **Think out loud.** At senior level, your reasoning is the answer.
3. **Clarify requirements first** for any design or scenario question; state your assumptions.
4. **Always give trade-offs**, never a single "best" answer: "It depends on X, and here is how I would decide."
5. **Use real project experience:** name the problem, what you tried, what you measured, and what you would do differently.
6. **Quantify:** latency, throughput, memory, cost, incident count — numbers make stories credible.
7. For tricky-output questions, **say the rule**, not just the output.
8. **Say "I don't know, but here's how I'd find out"** instead of bluffing; seniors are tested for honesty and approach.
9. Don't badmouth previous employers or teammates; show ownership and what you learned.
10. Revise the night before: this cheat sheet, the tricky-output section, your two best project stories — then sleep.

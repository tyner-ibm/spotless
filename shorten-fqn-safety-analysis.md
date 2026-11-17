# Analysis: Why Spotless `shortenFullyQualifiedTypes` Skips Classes with External Inheritance

## Executive Summary

The observed behavior—where Spotless does not shorten fully-qualified type names (FQNs) inside classes that `extend` or `implement` types declared in other files—is **not a bug**. It is an intentional safety mechanism designed into Spotless under Issue [#3117](https://github.com/diffplug/spotless/issues/3117) (commit `0d2e013ea`) to enforce Spotless's zero-risk safety invariant: **never introduce a compile error**.

---

## 1. Spotless Core Design Principle

Spotless operates as a source-code formatter and linter. Unlike full IDEs or compilers, Spotless steps (including `shortenFullyQualifiedTypes`) execute using **AST parsing without a compiled classpath** or semantic type-resolution engine.

Because Spotless has no access to external `.class` files or dependency jars:
- The formatter cannot query the member declarations or type hierarchies of external types.
- The primary contract stated in `ShortenQualifiedTypesFormatterFunc.java` and `ShortenFullyQualifiedTypesStepTest.java` is:
  > **"Safety contract: never introduce a compile error."**  
  > *"When in doubt, leave the reference qualified. An un-shortened reference is merely verbose; a wrongly-shortened one breaks the build."*

---

## 2. The Technical Problem: Shadowing via Inherited Member Types (JLS §6.5)

Under Java Language Specification rules for scope and unqualified identifier resolution (JLS §6.5.5.1 and §6.5.6.1):
1. **Inherited Member Scope Precedence**: Member types inherited from superclasses or superinterfaces (e.g., `Base.Type`) are directly in scope in the subclass body.
2. **Shadowing of Single-Type Imports**: An inherited member type directly **shadows / takes precedence over** a single-type import (`import external.Type;`).

### Concrete Example of the Failure Mode

Suppose an external library or separate file defines:

```java
// Base.java (external or library)
public class Base {
    public static class CloudEntityInfo {
        // ...
    }
}
```

And a subclass declares:

```java
// Subclass.java
public class Subclass extends Base {
    com.vmturbo.common.protobuf.action.ActionDTO.CloudEntityInfo getInfo() {
        return ...;
    }
}
```

If Spotless shortens the FQN, it rewrites the file to:

```java
import com.vmturbo.common.protobuf.action.ActionDTO.CloudEntityInfo;

public class Subclass extends Base {
    CloudEntityInfo getInfo() { // COMPILE ERROR: Resolves to Base.CloudEntityInfo!
        return ...;
    }
}
```

Because `Base` has a member type `CloudEntityInfo`, Java resolves the unqualified identifier `CloudEntityInfo` to `Base.CloudEntityInfo`, **not** the imported `ActionDTO.CloudEntityInfo`. This results in a compilation failure or silent semantic change.

---

## 3. Why Spotless Must Be Conservative Without a Classpath

Without a classpath:
1. Spotless cannot inspect `Base` or `Other` if they are defined in another file, jar, or library.
2. Spotless cannot determine whether `Base` or `Other` declares an inner class, enum, record, or interface with any given simple name.
3. Therefore, any class with an external supertype or implemented interface has **unknown inheritance**.

To uphold the absolute guarantee of **100% compile safety**, Spotless deliberately treats classes with unknown inheritance as unsafe for unqualified type substitution and leaves their FQNs qualified.

---

## 4. Current Hierarchy Resolution Rules in Spotless

Spotless inspects inheritance using `typesWithUnknownInheritance`:

| Scenario | Behavior | Reason |
|---|---|---|
| `class Foo { ... }` (no supertypes) | Shortened | Inheritance is fully known (implicit `java.lang.Object` has no member types). |
| `class Foo extends Object { ... }` | Shortened | `java.lang.Object` is explicitly known to have no member types. |
| `class Foo extends Base { ... }` where `Base` is in the same file | Shortened (except for types clashing with members in the file) | Spotless parses `Base` in the same compilation unit and knows all declared nested types. |
| `class Foo implements Runnable` or `class Foo extends ExternalBase` | Left qualified | Supertype is external; Spotless cannot prove what member types it inherits. |

---

## 5. Potential Future Improvements (Preserving 100% Safety)

While the current behavior is strictly correct according to Spotless's design invariant, the rule could be refined upstream in ways that preserve 100% compile safety:

1. **Allowlist Standard JDK Types**:
   Types defined in the Java Platform Standard (e.g., `java.io.Serializable`, `java.lang.Runnable`, `java.lang.Comparable`, `java.lang.AutoCloseable`, `java.util.EventListener`) have fixed, standardized specifications known never to declare conflicting member types. Spotless could safely treat these standard types as having known inheritance.

2. **Opt-in Strictness Flag**:
   Provide a configuration setting such as `<ignoreUnknownInheritance>true</ignoreUnknownInheritance>` for users who want aggressive FQN shortening across all classes and rely on their compiler to catch any rare collisions.

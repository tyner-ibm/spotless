# Spotless `shortenFullyQualifiedTypes` silently does nothing for classes that `implements` a type from another file

Status: **not yet filed upstream**. Findings below come from local experiments on 2026-11-16. The upstream issue tracker has not been searched for duplicates, and the behaviour has not been checked against newer releases.

## Summary

With `<shortenFullyQualifiedTypes />` configured, Spotless does not shorten fully qualified type names (FQNs) inside a class whose declaration has an `implements` (or `extends`) clause naming a type that is not declared in the same file. The step succeeds and reports no warning. It leaves the FQNs as they are.

Classes with no supertypes, with only `extends Object`, or with supertypes declared in the same file are shortened normally.

The behaviour appears deliberate (see "Likely cause"), but it is undocumented, silent, and makes the step ineffective for most real-world classes.

## Environment

| Item | Value |
|---|---|
| spotless-maven-plugin | 3.11.0 |
| JavaParser (pulled by the step) | `com.github.javaparser:javaparser-core:3.28.2` (taken from the `MAVEN_COORDINATES` constant in `ShortenFullyQualifiedTypesStep`) |
| spotless-lib | 4.11.0 was the jar inspected. It was assumed, not verified, to be the version 3.11.0 of the plugin resolves |
| Maven | 3.10.0 (also reproduced via `mvnd` in the real project) |
| Java | 17 (Amazon Corretto 17.0.17) |
| OS | macOS (arm64) |

## Minimal reproduction

Create a project with two files.

`pom.xml`:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">
  <modelVersion>4.0.0</modelVersion>
  <groupId>t</groupId>
  <artifactId>t</artifactId>
  <version>1</version>
  <build>
    <plugins>
      <plugin>
        <groupId>com.diffplug.spotless</groupId>
        <artifactId>spotless-maven-plugin</artifactId>
        <version>3.11.0</version>
        <configuration>
          <java>
            <includes>
              <include>src/**/*.java</include>
            </includes>
            <shortenFullyQualifiedTypes/>
          </java>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

`src/main/java/a/T.java`:

```java
package a;

import java.util.Optional;

class T implements Other {
    Optional<com.vmturbo.common.protobuf.action.ActionDTO.CloudEntityInfo> f() {
        return Optional.empty();
    }
}
```

Run:

```
mvn spotless:apply
```

(No `Other` source is needed. The file does not have to compile, since Spotless only parses it.)

### Expected

```java
import java.util.Optional;
import com.vmturbo.common.protobuf.action.ActionDTO.CloudEntityInfo;

class T implements Other {
    Optional<CloudEntityInfo> f() {
```

### Actual

The file is unchanged. The build reports `BUILD SUCCESS` with no warning, and the FQN is still in the file.

## Experiment matrix

Each row uses the same body: a method returning `Optional<com.vmturbo.common.protobuf.action.ActionDTO.CloudEntityInfo>`. Only the class declaration changes. "Shortened" means the FQN was replaced and an import was added.

| Class declaration | Result |
|---|---|
| `class T {` | shortened |
| `abstract class T {` | shortened |
| `final class T {` | shortened |
| `interface T {` | shortened |
| `class T extends Object {` | shortened |
| `class T implements Runnable {` | **not shortened** |
| `class T implements Cloneable, Comparable<T> {` | **not shortened** |
| `class T implements java.io.Serializable {` | **not shortened** |
| `class T implements Other {` (`Other` not in file) | **not shortened** |
| `class T implements Base {` plus `interface Base {}` in the same file | shortened |
| `class T { public void run() {} ... }` (no supertypes, other members present) | shortened |

Other things ruled out. They made no difference in separate runs:
- A trailing `// fqn` line comment.
- A Javadoc `{@link <FQN>}`.
- `@Override` and `@Nonnull` annotations.
- Other imports from the same outer class (`ActionDTO.*`).
- A `private static final` field.
- Using `importOrder` or `removeUnusedImports` alongside the step.

Real-world instance: `DatabaseScaleTranslator implements EntityScaleTranslator`, where `EntityScaleTranslator` is a separate file. Its FQNs were never shortened. Copying that file into the minimal project, with the class renamed, gave the same result.

## Likely cause (inferred from bytecode, not from source or docs)

This comes from running `javap -c` on the classes in `spotless-lib-4.11.0.jar`, specifically `com.diffplug.spotless.glue.javaparser.ShortenQualifiedTypesFormatterFunc`. The source and the upstream documentation were not read, so treat it as a hypothesis.

The class has these private helpers:

- `typesWithUnknownInheritance(CompilationUnit)`
- `hasUnknownInheritance(TypeDeclaration, ...)`
- `supertypes(TypeDeclaration)` (reads `getExtendedTypes()` / `getImplementedTypes()`)
- `isJavaLangObject(ClassOrInterfaceType, ...)`
- `resolveInFile(ClassOrInterfaceType, ...)`
- `insideUnknownInheritance(Node, Set<TypeDeclaration>)`

This is consistent with the matrix above:
- `extends Object` is recognised by `isJavaLangObject`, so it is safe.
- A supertype declared in the same file is found by `resolveInFile`, so it is safe.
- Any other supertype cannot be resolved without the classpath, so the whole type is treated as having "unknown inheritance" and its body is skipped. This probably protects against a shortened name being shadowed by a nested type inherited from an unresolvable supertype.

## What the reporter should check first

1. Does it still reproduce on the latest spotless-maven-plugin?
2. Search upstream issues and the docs for "unknown inheritance", `shortenFullyQualifiedTypes`, and `implements`, to check whether this is known or documented.
3. Read `ShortenQualifiedTypesFormatterFunc` in the Spotless source to confirm or refute the hypothesis above.

## Suggested ask for upstream

Any of these would help:

- **Document the limitation** on the `shortenFullyQualifiedTypes` page: classes with unresolved supertypes are skipped.
- **Log a message** (at least at debug level) when a type is skipped for unknown inheritance, so the no-op is not silent.
- **Offer a way to opt in** for projects that accept the shadowing risk. Examples: a flag to ignore inheritance, or a way to provide resolvable supertypes.
- **Narrow the skip.** Where the FQN's first segment is lowercase and package-like, shadowing by an inherited nested type is not plausible. Such FQNs could be shortened safely.

## Impact on this repo

`shortenFullyQualifiedTypes` is configured in [`pom.xml`](../pom.xml) and [`com.vmturbo.cloud.optimizer.component/pom.xml`](../com.vmturbo.cloud.optimizer.component/pom.xml). It is effectively inert for any class that implements or extends a type from another file, which is most of this codebase. FQNs in such classes have to be fixed by hand.

## Side note found while investigating (separate problem)

The Spotless config originally had two `<formats>` elements inside `<configuration>`. Maven keeps only one, so the Java format was silently dropped. The fix is to use the dedicated `<java>` element for Java and a single `<formats>` for everything else. This is unrelated to the bug above, but it was the first reason the rule appeared not to work.

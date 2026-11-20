# Module replacement and version reselection assertion failure

Standalone reproducer using stock Gradle 9.7.1, no plugins, and no external dependencies. Tested with Java 17 on Linux. The `repo/` directory contains a static Maven repository with POMs and empty JARs; dependency resolution requires no network. The wrapper needs network access to download Gradle on its first run.

## Reproduce

```sh
./gradlew reproduce --no-daemon -Dorg.gradle.jvmargs='-Xmx512m -ea' --stacktrace
```

Expected: successful resolution to `example:bridge:1` and `example.modern:library:3`, with the legacy library replaced.

Actual: build fails with:

```text
Caused by: java.lang.AssertionError
    at org.gradle.api.internal.artifacts.ivyservice.resolveengine.graph.builder.ModuleResolveState.resolveModuleConflict(ModuleResolveState.java:236)
    at org.gradle.api.internal.artifacts.ivyservice.resolveengine.graph.conflicts.DefaultModuleConflictHandler.resolveNextConflict(DefaultModuleConflictHandler.java:101)
```

Assertions are explicitly enabled in the Gradle daemon JVM, where resolution happens.

## Dependency graph

The root configuration requests:

- `example.modern:library:1`
- `example.modern:library:2`
- `example:bridge:1`, which depends on `example.modern:library:3` and `example.legacy:library:1`

A standard module replacement rule declares `example.legacy:library` replaced by `example.modern:library`.

The two initial requests cause version selection before a transitive path introduces version 3 and the legacy module. This exercises an inconsistency between replacement conflict candidates and the module's current selection. The exact internal sequence remains a hypothesis until instrumented.

## Assertions-disabled comparison

```sh
./gradlew reproduce --no-daemon -Dorg.gradle.jvmargs='-Xmx512m -da'
```

Observed: succeeds and prints `bridge-1.jar` and `library-3.jar`. This minimal case establishes an assertion failure, not an incorrect final artifact version with assertions disabled. It does not prove that all affected graphs resolve correctly without assertions.

No internal repositories, dependencies, plugins, credentials, or services are involved. The build reads the bundled repository without generating or publishing any artifacts. Run without configuration cache.

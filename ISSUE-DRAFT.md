# YouTrack issue draft — DO NOT auto-post

File manually at <https://youtrack.jetbrains.com/newIssue?project=AMPER>
(issues get a `KTC-` ID). Everything below the rule is the issue text.

---

**Summary:** Documented `generated.*.fragment.modifier` form (without `@`) crashes model reading with internal NoSuchElementException in selectFragmentByDescriptor

## Environment

- Kotlin Toolchain 0.11.1 (801e9d4, 2026-06-05), standard vendored wrapper
- Reproduced on macOS 26.3.1 (arm64) and Linux x86-64 (Fedora)
- Still reproduces identically on the latest dev build, 0.12.0-dev-4213
  (26ef291, 2026-08-04), where the workaround also still works

## Description

The plugin.yaml schema documentation for `generated.<kind>.fragment.modifier`
says the value is the fragment qualifier **without** the `@` symbol:

- `FragmentDescriptor` schema doc in
  [`pluginYamlSchema.kt#L154-L161` (v0.11.1)](https://github.com/JetBrains/kotlin-toolchain/blob/v0.11.1/sources/frontend-api/src/org/jetbrains/amper/frontend/plugins/pluginYamlSchema.kt#L154-L161):
  "The fragment qualifier without the `@` symbol (e.g. `jvm`, `ios`, etc.)"
- [`docs/src/user-guide/plugins/topics/tasks.md#L216-L226` (v0.11.1)](https://github.com/JetBrains/kotlin-toolchain/blob/v0.11.1/docs/src/user-guide/plugins/topics/tasks.md?plain=1#L216-L226)
  likewise documents `modifier: ios` (no `@`), and the shorthand
  `fragment: native`.

Following that documentation crashes `./kotlin build` during model reading:

```
Internal error: java.util.NoSuchElementException: Collection contains no element matching the predicate.
```

```
java.util.NoSuchElementException: Collection contains no element matching the predicate.
	at org.jetbrains.amper.frontend.aomBuilder.plugins.ApplyPluginsKt.selectFragmentByDescriptor(applyPlugins.kt:359)
	at org.jetbrains.amper.frontend.aomBuilder.plugins.ApplyPluginsKt.collectGeneratedMarks(applyPlugins.kt:198)
	at org.jetbrains.amper.frontend.aomBuilder.plugins.ApplyPluginsKt.applyPlugins(applyPlugins.kt:55)
	at org.jetbrains.amper.frontend.aomBuilder.plugins.BuildAndApplyPluginsKt.buildAndApplyPlugins(buildAndApplyPlugins.kt:34)
	at org.jetbrains.amper.frontend.aomBuilder.BuildKt.doReadProjectModel(build.kt:118)
	at org.jetbrains.amper.frontend.aomBuilder.BuildKt.readProjectModel(build.kt:86)
	at org.jetbrains.amper.cli.project.ProjectKt.preparePluginsAndReadModel(project.kt:126)
	at org.jetbrains.amper.cli.commands.AmperModelAwareCommand.run(AmperModelAwareCommand.kt:17)
	at org.jetbrains.amper.cli.MainKt.main(main.kt:145)
```

## Steps to reproduce

Minimal reproduction project:
<https://github.com/Serisium/kotlin-toolchain-fragment-modifier-repro>

1. `git clone https://github.com/Serisium/kotlin-toolchain-fragment-modifier-repro`
2. `./kotlin build`

The project is a `kmp/lib` module (platforms `[ jvm, linuxX64 ]`) plus a
minimal local `jvm/amper-plugin` whose only task writes one file into an
`@Output` directory, contributed via:

```yaml
generated:
  resources:
    - directory: ${tasks.generateResource.action.outputDir}
      fragment:
        modifier: jvm   # documented form
```

The shorthand `fragment: jvm` crashes identically.

## Expected behavior

The build succeeds and the generated directory is attached to the `jvm`
fragment's resources — or, if the value is considered invalid, a proper
error message pointing at the plugin.yaml location instead of an internal
crash.

## Root cause

Fragments store their modifier **with** the `@` prefix:
[`fragmentSeeds.kt#L97` (v0.11.1)](https://github.com/JetBrains/kotlin-toolchain/blob/v0.11.1/sources/frontend/schema/src/org/jetbrains/amper/frontend/aomBuilder/fragmentSeeds.kt#L97)
builds `"@${hierarchyPlatform.pretty}"`. But `selectFragmentByDescriptor` in
[`applyPlugins.kt#L149-L156` (v0.11.1)](https://github.com/JetBrains/kotlin-toolchain/blob/v0.11.1/sources/frontend/schema/src/org/jetbrains/amper/frontend/aomBuilder/plugins/applyPlugins.kt#L149-L156)
compares the user-supplied descriptor string verbatim:

```kotlin
.first { it.isTest == descriptor.isTest && it.modifier == descriptor.modifier }
```

`"jvm" != "@jvm"`, so the documented form never matches and `first` throws.
The call site even carries a FIXME noting that `first` will crash on
incorrect user input. (The stacktrace's `applyPlugins.kt:359` frame is a
debug-info artifact of the shipped dist — the v0.11.1 source file, build
commit `801e9d4`, is only 258 lines.)

The code is unchanged as of the latest dev build's commit:
[`applyPlugins.kt#L149-L156` @ 26ef291](https://github.com/JetBrains/kotlin-toolchain/blob/26ef291/sources/frontend/schema/src/org/jetbrains/amper/frontend/aomBuilder/plugins/applyPlugins.kt#L149-L156).

So two fixes seem warranted:

1. Reconcile the convention: either normalize the descriptor (accept the
   documented bare form, e.g. strip/add `@` before comparing) or change the
   docs/schema to require the `@`-prefixed form.
2. Per the existing FIXME, replace `first` with a real diagnostic so an
   unmatched fragment descriptor reports the offending plugin.yaml value
   instead of an internal `NoSuchElementException`.

## Workaround

Write the modifier with the `@` prefix, contradicting the docs:

```yaml
      fragment:
        modifier: "@jvm"   # or shorthand: fragment: "@jvm"
```

With that change the same project builds successfully and the generated
resource lands on the jvm classpath.

## Prior search

No existing report found for this crash (searched YouTrack for
`selectFragmentByDescriptor`, `NoSuchElementException`,
"Collection contains no element", `fragment modifier`, `plugin.yaml`,
`generated fragment`). Related but distinct: KTC-4773 (attach plugin.yaml
to model-reading exception reports — this crash is exactly that category),
KTC-4904 (unclear message when no valid task actions), KTC-3619 (`@ios`
settings propagation, resolved 2024).

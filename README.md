# Kotlin Toolchain 0.11.1: documented `generated.*.fragment.modifier` form crashes model reading

> **Filed as [KTC-5646](https://youtrack.jetbrains.com/issue/KTC-5646)** (2026-08-05).

Minimal reproduction for a Kotlin Toolchain (Amper) 0.11.1 bug: writing a
build plugin's `generated.<kind>.fragment.modifier` the way the docs say
(fragment qualifier **without** the `@` symbol, e.g. `jvm`) crashes
`./kotlin build` during model reading with an internal
`NoSuchElementException`. The issue text as filed is in
[ISSUE-DRAFT.md](ISSUE-DRAFT.md).

## Layout

- `lib/` — trivial `kmp/lib` module, platforms `[ jvm, linuxX64 ]`
- `build-plugins/repro-plugin/` — trivial `jvm/amper-plugin` with one task
  that writes a single file to an `@Output` directory, contributed back to
  the build via `generated.resources` with `fragment: { modifier: jvm }`
  (the documented form — [plugin.yaml](build-plugins/repro-plugin/plugin.yaml))
- `kotlin` / `kotlin.bat` — the standard 0.11.1 wrapper

## Reproduce

```bash
./kotlin build
```

Observed (0.11.1, macOS 26.3.1 arm64; also reproduced on Linux x86-64):

```
Internal error: java.util.NoSuchElementException: Collection contains no element matching the predicate.

Please check the build logs for the full stacktrace, and if possible file a bug report at https://youtrack.jetbrains.com/newIssue?project=AMPER
```

Stacktrace top (from `build/logs/.../debug.log`):

```
java.util.NoSuchElementException: Collection contains no element matching the predicate.
	at org.jetbrains.amper.frontend.aomBuilder.plugins.ApplyPluginsKt.selectFragmentByDescriptor(applyPlugins.kt:359)
	at org.jetbrains.amper.frontend.aomBuilder.plugins.ApplyPluginsKt.collectGeneratedMarks(applyPlugins.kt:198)
	at org.jetbrains.amper.frontend.aomBuilder.plugins.ApplyPluginsKt.applyPlugins(applyPlugins.kt:55)
	...
```

The shorthand form `fragment: jvm` crashes identically.

Also reproduces unchanged on the latest dev build, **0.12.0-dev-4213**
(`26ef291`, 2026-08-04): same crash and stacktrace with the documented form,
and the `"@jvm"` workaround below still builds. (To retest against a dev
build, point `kotlin_cli_version`/`kotlin_cli_sha256` in the `kotlin`
wrapper at a version from the
[dev Maven repo](https://packages.jetbrains.team/maven/p/amper/amper/org/jetbrains/kotlin/kotlin-cli/).)

## Expected

The build succeeds and `generated.txt` lands on the jvm classpath — which is
exactly what happens with the undocumented `@`-prefixed workaround:

```yaml
      fragment:
        modifier: "@jvm"   # works; documented bare form crashes
```

Edit [build-plugins/repro-plugin/plugin.yaml](build-plugins/repro-plugin/plugin.yaml)
accordingly and rerun `./kotlin build` to see `Build successful`
(`generated.txt` appears in `build/artifacts/CompiledJvmArtifact/libjvm/resources-output/`).

## Root cause (0.11.1 sources)

- The schema doc
  ([`pluginYamlSchema.kt` — `FragmentDescriptor.modifier`](https://github.com/JetBrains/kotlin-toolchain/blob/v0.11.1/sources/frontend-api/src/org/jetbrains/amper/frontend/plugins/pluginYamlSchema.kt#L154-L161))
  and the user guide
  ([`tasks.md` — "Referencing module fragments"](https://github.com/JetBrains/kotlin-toolchain/blob/v0.11.1/docs/src/user-guide/plugins/topics/tasks.md?plain=1#L216-L226))
  both say the modifier is the fragment qualifier *without* the `@`
  symbol (e.g. `modifier: ios`).
- But fragments store their modifier *with* the prefix:
  [`fragmentSeeds.kt:97`](https://github.com/JetBrains/kotlin-toolchain/blob/v0.11.1/sources/frontend/schema/src/org/jetbrains/amper/frontend/aomBuilder/fragmentSeeds.kt#L97)
  builds `"@${hierarchyPlatform.pretty}"`.
- [`selectFragmentByDescriptor` (`applyPlugins.kt:149-156`)](https://github.com/JetBrains/kotlin-toolchain/blob/v0.11.1/sources/frontend/schema/src/org/jetbrains/amper/frontend/aomBuilder/plugins/applyPlugins.kt#L149-L156)
  compares the user-supplied string verbatim —
  `.first { it.isTest == descriptor.isTest && it.modifier == descriptor.modifier }` —
  and carries a FIXME noting that `first` will crash on incorrect user
  input. `"jvm" != "@jvm"`, so the documented form matches nothing and
  `first` throws. (The stacktrace's `applyPlugins.kt:359` frame exceeds
  the 258-line source file — a debug-info artifact of the shipped dist;
  tag `v0.11.1` is the dist's build commit `801e9d4`.)

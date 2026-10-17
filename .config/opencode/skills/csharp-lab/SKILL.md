---
name: csharp-lab
description: Creates temporary C# scripts for quick investigation, debugging,
  and validation without modifying the main codebase. Use this skill instead of
  creating files in temporary directories.
version: 1.1.0
license: MIT
tags:
  - csharp
  - dotnet
  - debugging
  - testing
---

# Purpose

Use this skill for quick, temporary C# investigation, including:

- reproducing bugs
- isolating root causes
- verifying API behavior
- checking language/runtime behavior
- testing edge cases
- validating assumptions before editing real code
- comparing two or more implementation approaches
- checking LINQ, async, nullable, generics, reflection, serialization, or DI behavior
- creating minimal examples from production logic
- testing existing project logic through a project-reference directive when appropriate
- smoke-checking performance characteristics only when useful and cheap
- confirming whether a proposed fix is plausible before applying it

This skill is NOT for:

- creating production artifacts
- scaffolding full applications
- leaving temporary files behind unless the workspace is intentionally persistent
- long-running services
- benchmark suites requiring statistical rigor
- modifying user code unless explicitly asked
- adding dependencies unless explicitly asked

# Activation gate

Before using this skill, verify that shell execution is available.

If shell execution is unavailable:

- do not use this skill
- do not create files under `csharp-lab/`
- tell the user: `C# Lab skill skipped: shell execution is unavailable.`
- continue without this skill unless the user asks otherwise

Then check the .NET version:
```bash
dotnet --version
```

Parse the major version from the output.

Only use this skill if:

- the `dotnet` command exists
- the major version is `10` or higher

If `dotnet` is missing, the version cannot be determined, or the major version is less than `10`:

- do not use this skill
- do not create files under `csharp-lab/`
- do not create a `.csproj`
- tell the user: `C# Lab skill skipped: requires .NET 10 or above.`
- continue without this skill unless the user asks otherwise

If `dotnet run <file>.cs` fails because direct file execution is unavailable:

- stop using this skill
- report the issue
- do not fall back to creating a `.csproj` unless the user explicitly asks

# Core principles

## Keep it cheap

- Prefer tiny examples.
- Avoid large data, long loops, network calls, or heavyweight setup unless necessary.

## Keep it disposable unless the workspace is intentionally persistent

- Generated files are temporary by default.
- They are not deliverables unless the user explicitly asks to keep them.

## Keep it evidence-based

- Prefer running code over speculating.
- Show expected vs actual behavior where useful.
- Use clear PASS/FAIL output when appropriate.

## Keep it isolated

- Do not modify the user's real source files unless asked.
- Do not affect the user's build output or repository state more than necessary.
- Do not create project scaffolding.

## Keep it JIT by default

File-based C# apps run in AOT by default.

AOT is not ideal for many investigations, especially when testing:

- JSON serialization or deserialization
- API response mapping
- reflection
- dynamic behavior
- some generic patterns
- DI or framework behavior that may differ under AOT

Therefore, generated runnable files must disable AOT by default:

```csharp
#:property PublishAot=false
```

Only allow AOT when:

- AOT behavior itself is being investigated
- the user explicitly asks for AOT
- the user's request clearly implies that AOT behavior matters

If AOT is intentional, omit the disable directive or explicitly enable AOT if needed, and mention that the test is running under AOT.

# Workspace rules

Create `csharp-lab/` relative to the current workspace/project root unless the user specifies another location.

Prefer a unique run folder:

```text
csharp-lab/<timestamp-or-slug>/
```

Examples:

```text
csharp-lab/auth-token-repro/
csharp-lab/linq-orderby-check/
csharp-lab/json-nullability/
```

Inside that folder, create small runnable `.cs` files.

Good examples:

```text
01-repro.cs
02-minimal-case.cs
03-alternative-approach.cs
04-edge-case.cs
05-fix-check.cs
```

Each `.cs` file should be directly runnable with:

```bash
dotnet run csharp-lab/<folder>/<file>.cs
```

Example:

```bash
dotnet run csharp-lab/auth-token-repro/01-repro.cs
```

# File rules

Each runnable `.cs` file must be:

- self-contained
- small
- focused on one question or one scenario
- directly executable
- free of external dependencies unless the user explicitly asks
- prefixed with the required AOT-disable directive unless AOT is intentional

Each file should begin with directives before any C# code:

```csharp
#:property PublishAot=false
```

Then include:

- the minimal types/helpers needed
- top-level statements or an entrypoint
- clear console output
- explicit PASS/FAIL where appropriate
- a non-zero exit code on failure when useful

Example:

```csharp
#:property PublishAot=false

int Add(int a, int b) => a + b;

var expected = 5;
var actual = Add(2, 3);

if (actual == expected)
{
    Console.WriteLine($"PASS: Add(2, 3) == {expected}");
}
else
{
    Console.WriteLine($"FAIL: expected {expected}, got {actual}");
    Environment.Exit(1);
}
```

If multiple scenarios are needed, create multiple files instead of one large file.

If shared helper code is needed, prefer duplicating the minimal helper into each runnable file so each file remains independently runnable.

# Project-reference directives

When the logic under test already exists in a nearby `.csproj`, the agent should prefer referencing that project directly instead of duplicating the code.

Use a project-reference directive at the top of the runnable `.cs` file:

```csharp
#:project ../MyProject/MyProject.csproj
```

The path is relative to the runnable `.cs` file.

This allows the lab file to use the target project's types directly while the lab file itself remains the entry point.

Example:

```csharp
#:property PublishAot=false
#:project ../../src/MyApp/MyApp.csproj

using MyApp.Services;

var service = new TokenService();

var expected = "Bearer abc123";
var actual = service.BuildAuthToken("abc123");

if (actual == expected)
{
    Console.WriteLine($"PASS: BuildAuthToken returned expected value");
}
else
{
    Console.WriteLine($"FAIL: expected {expected}, got {actual}");
    Environment.Exit(1);
}
```

When using a project reference:

- the lab file should usually use top-level statements
- do not declare a namespace, `Program` class, or `Main` method unless required
- do not duplicate logic from the referenced project unless isolation requires it
- do not modify the referenced project
- do not create a new `.csproj` just to make the reference work

Use a project reference when:

- the referenced project already exists
- no changes to the referenced project are required
- the project can be built cheaply and safely
- its existing dependencies are acceptable
- the goal is to test existing implementation logic directly

Do not use a project reference when:

- referencing the project would require modifying it
- the project is heavy, slow to build, or otherwise unsuitable
- the test requires a fully isolated reproduction
- the needed types are inaccessible
- referencing it would dirty the repository state in a way that is not acceptable

If a project reference is not suitable, fall back to duplicating the minimal logic into the lab file.

Using an existing project through a `#:project` directive is allowed. Creating a new project file is not.

# Project-file prohibition

Do NOT create any project files unless the user explicitly asks.

Specifically, do not create:

```text
*.csproj
*.sln
global.json
Directory.Build.props
Directory.Packages.props
nuget.config
```

Do NOT use these commands for this skill:

```bash
dotnet new
dotnet build
dotnet test
dotnet publish
```

Only use:

```bash
dotnet run <path-to-file.cs>
```

# Diagnosis workflow

When using this skill to find an issue:

1. State the likely question or hypothesis.

   Example:

   - "Does this method throw when input is null?"
   - "Does this LINQ query defer execution?"
   - "Does this serializer omit null values?"

2. Create the smallest possible reproduction.

3. Run it.

4. If the behavior is surprising, reduce the example further.

5. Add focused checks.

6. If useful, create one or more alternative examples.

7. Identify the root cause or likely root cause.

8. Recommend the next action.

Examples:

- apply a specific fix
- add a guard
- change an API contract
- add a regression test
- investigate a framework limitation

# Implementation-support workflow

When the user asks to implement something and this skill is useful:

1. Use the scratch lab to validate the approach first.
2. Keep the temporary code minimal.
3. Identify edge cases early.
4. Confirm the expected behavior.
5. Apply cleanup rules.
6. Then implement or propose the real change in the actual codebase if the user asked for that.

Do not treat the temporary C# files as the final implementation.

# Comparison workflow

When comparing approaches:

Create separate files:

```text
01-approach-a.cs
02-approach-b.cs
03-comparison.cs
```

Each file should print:

- what approach is being used
- the result
- whether the result is correct
- any relevant exception or behavioral difference

Only mention performance if the difference is obvious, requested, or diagnostically important.

# Safety rules

While using this skill:

- do not modify files outside `csharp-lab/` unless explicitly asked
- do not modify referenced projects unless explicitly asked
- do not delete user files
- do not read or print secrets
- do not commit anything
- do not start long-running processes unless explicitly asked
- do not make network calls unless necessary and the user's request implies it
- do not install global tools
- do not add NuGet packages unless explicitly requested

If an experiment requires something unsafe, expensive, or irreversible, ask first.

# Output rules

After running the skill, report:

- what was tested
- files created
- commands run
- observed results
- root cause or conclusion
- recommended next step
- cleanup status

Keep the report concise.

Example:

```text
Created temporary files under csharp-lab/json-nullability/.

Ran:
- dotnet run csharp-lab/json-nullability/01-repro.cs
- dotnet run csharp-lab/json-nullability/02-fix-check.cs

Result:
The serializer omitted null properties when DefaultIgnoreCondition was set to WhenWritingNull.

Conclusion:
The behavior is caused by JSON serializer options, not the model.

Cleanup:
Left files in csharp-lab/ because it is Git-ignored.
```

# Cleanup

Cleanup depends on whether `csharp-lab/` is Git-ignored.

Before cleaning up, determine whether `csharp-lab/` is ignored by Git.

If inside a Git repository, use:

```bash
git check-ignore -q csharp-lab
```

Interpret the result:

- exit code `0`: `csharp-lab/` is Git-ignored
- non-zero exit code: `csharp-lab/` is not Git-ignored, or Git status could not be determined

Cleanup rules:

If `csharp-lab/` is Git-ignored:

- do NOT automatically clean up
- leave the generated files in place
- tell the user where they are

Example:

```text
Left C# Lab files in csharp-lab/ because it is Git-ignored.
```

If `csharp-lab/` is not Git-ignored:

- clean up after the skill is finished
- delete all files created by this skill run
- if using a run folder, delete the entire run folder
- if this skill created `csharp-lab/` and it is now empty, delete the directory too

If Git status cannot be determined:

- treat the workspace as not Git-ignored
- clean up after the skill is finished

User overrides:

- if the user explicitly asks to keep the files, keep them
- if the user explicitly asks to clean up, clean up even if `csharp-lab/` is Git-ignored

Never delete unrelated user files.

Example cleanup confirmation when not ignored:

```text
Cleaned up temporary C# Lab files.
```

Example cleanup status when ignored:

```text
Left C# Lab files in csharp-lab/ because it is Git-ignored.
```
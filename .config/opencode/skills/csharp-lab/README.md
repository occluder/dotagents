# csharp-lab

A disposable C#/.NET scratch lab skill for coding agents.

Use it to answer C# questions by running tiny, temporary programs instead of guessing.

## Requirements

- .NET SDK 10 or higher
- Support for direct file execution via:
  ```bash
  dotnet run path/to/file.cs -p:PublishAot=false
  ```
- A coding agent/harness that supports Markdown skills

## What it does

This skill tells the agent to:

- create temporary C# files under `csharp-lab/`
- run small, focused experiments
- avoid modifying real project files
- avoid creating `.csproj`, `.sln`, or other project scaffolding
- clean up unless `csharp-lab/` is Git-ignored
- report evidence-based results

## Install for OpenCode

Project-local:

```bash
git clone https://github.com/YOUR_USERNAME/csharp-lab .opencode/skills/csharp-lab
```

Global on macOS/Linux:

```bash
git clone https://github.com/YOUR_USERNAME/csharp-lab ~/.config/opencode/skills/csharp-lab
```

Then restart OpenCode or start a new session.

## Install for other harnesses

Copy the skill folder into your harness’s skills directory.

For example:

```text
<skills-directory>/csharp-lab/SKILL.md
```

Consult your harness documentation for the exact skills location.


## Usage

You usually do not need to invoke this skill manually by explicitly mentioning it. If your request involves checking, reproducing, or comparing C#/.NET behavior, the agent should automatically consider using it when the environment supports it.

For example, this may trigger the skill:

```text
Can you check whether this LINQ query is deferred?
```

## License

```text
MIT License

Copyright (c) 2026 Your Name

Permission is hereby granted, free of charge, to any person obtaining a copy...
```
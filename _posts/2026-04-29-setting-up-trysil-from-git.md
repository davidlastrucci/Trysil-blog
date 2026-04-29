---
title: "Setting up Trysil from git"
date: 2026-04-29 16:30:00 +0200
categories: [Tutorials]
tags: [getting-started, installation, build, tutorial]
---

You found Trysil on GitHub and want to try it. This post walks through everything between `git clone` and a working Delphi project that links against Trysil's packages: cloning the repo, building for your Delphi version, configuring the search path, and one section that surprises Delphi newcomers — which packages your edition can actually compile.

GetIt is also an option if you prefer the package-manager route. Going from git is what I'd recommend if you want to track tip of `master`, pin to a tag, or contribute back. Everything below assumes a fresh clone.

## 1. Clone

```bash
git clone https://github.com/davidlastrucci/Trysil.git
```

This gives you the full source tree. The relevant directories:

- `Trysil/` — core ORM
- `Trysil.JSon/` — JSON serialization module
- `Trysil.Http/` — REST/HTTP module
- `Packages/` — `.dproj` and `.dpk` files for each supported Delphi version, plus the build batches
- `Demos/` — runnable examples
- `Lib/` — output directory, created by the build

## 2. Build the packages

Each supported Delphi version has its own `Build<ver>.bat` in `Packages/`:

| Version | Batch file | Delphi |
|---|---|---|
| `370` | `Build370.bat` | 13 Florence |
| `290` | `Build290.bat` | 12 Athens |
| `280` | `Build280.bat` | 11 Alexandria |
| `270` | `Build270.bat` | 10.4 Sydney |
| `260` | `Build260.bat` | 10.3 Rio |

Open a command prompt in `Packages/` and run the one that matches your Delphi:

```bat
Build370.bat
```

Each `Build<ver>.bat` is a one-liner that delegates to `MainBuild.bat` with three arguments — Studio version, package version, and target platforms:

```bat
@CALL "%~dp0MainBuild.bat" 37.0 370 "Win32 Win64"
```

`Win32` and `Win64` in both `Debug` and `Release` is the default. If you also want `Linux64` (or `OSX64`, `iOSDevice64`, etc.), open the batch file for your version — it's literally one line — and append the platform to the quoted list:

```bat
@CALL "%~dp0MainBuild.bat" 37.0 370 "Win32 Win64 Linux64"
```

Compiled output lands under `Lib/<ver>/$(Platform)/$(Config)/` as `.bpl`, `.dcp`, and `.dcu` files. On a clean run you'll see one `[OK]` line per package/config/platform and a final `BUILD SUCCESS` in green. If anything fails, the MSBuild output for that single invocation is printed inline — easier to read than scrolling through a wall of green-on-success noise.

## 3. Edition compatibility

This is the part that surprises people new to Delphi. Not every package compiles on every edition: FireDAC's database drivers are gated by the edition Embarcadero licensed you.

| Driver | Community | Professional | Enterprise | Architect |
|---|---|---|---|---|
| `Trysil.SQLite` | ✓ | ✓ | ✓ | ✓ |
| `Trysil.PostgreSQL` | ✓ (localhost only) | ✓ (localhost only) | ✓ | ✓ |
| `Trysil.FirebirdSQL` | ✓ (localhost / embedded) | ✓ (localhost / embedded) | ✓ | ✓ |
| `Trysil.SqlServer` | ✗ | ✗ | ✓ | ✓ |

Two practical consequences:

- **SQL Server requires Enterprise or Architect.** On Community and Professional, the `Trysil.SqlServer<ver>` package fails to build because the FireDAC SQL Server units (`FireDAC.Phys.MSSQL`) ship only with the higher editions. The fix on Pro is to remove `Trysil.SqlServer` from the package list in `MainBuild.bat`, or just accept the single failure: the build continues regardless, every other driver still produces its `.bpl` and `.dcu`.
- **PostgreSQL and Firebird connect to local servers only on Community/Professional.** The packages compile and the units are usable; FireDAC simply restricts the runtime connection to `localhost` (or embedded mode for Firebird). To talk to a remote database server you need Enterprise or Architect.

SQLite has no restriction — it's file-based, "local" by definition, and works on every edition. That's why every Trysil tutorial post starts there.

If your goal is just to follow along with the documentation, none of this affects you.

## 4. Configure your project

Two settings in your consumer project and you're done.

### Environment variable `$(Trysil)`

Point an environment variable named `Trysil` at the `Lib/<ver>/` directory. For Delphi 13:

```
Trysil = C:\path\to\Trysil\Lib\370
```

You can set it system-wide (Windows → Environment Variables) or per-IDE in **Tools → Options → IDE → Environment Variables**. The latter keeps the variable scoped to Delphi and out of your shell.

### Search path

In your project's **Project Options → Building → Delphi Compiler → Search path**, add:

```
$(Trysil)\$(Platform)\$(Config)
```

`$(Platform)` and `$(Config)` are Delphi's own build-time macros (`Win64`, `Debug`, etc.). One entry covers every platform/config combination — you don't need a separate path per target.

## 5. Verify

If everything is set up correctly:

- `Build370.bat` ended with `BUILD SUCCESS` in green
- `Lib/370/Win64/Release/Trysil370.bpl` exists (or the path matching your version and target)
- A new Delphi project with

  ```pascal
  uses
    Trysil.Data,
    Trysil.Data.FireDAC.SQLite,
    Trysil.Context;
  ```

  compiles without complaining about missing units

You're ready for the next step. The natural follow-up is [your first Trysil entity]({% post_url 2026-04-21-your-first-trysil-entity %}) — a tiny SQLite program that maps a class and runs read, insert, update, delete.

Happy coding.

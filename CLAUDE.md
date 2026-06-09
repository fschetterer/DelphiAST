# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

DelphiAST is an Abstract Syntax Tree parser for Delphi/Object Pascal source code. It parses one unit at a time without requiring a symbol table, producing an XML (or binary) representation of the code structure. Compatible with both Delphi and FreePascal/Lazarus.

## Build & Test

This is a native Object Pascal project. There is no command-line build/test pipeline — it uses:
- **Delphi IDE**: Open `.dproj` files (requires Delphi with VCL)
- **Lazarus IDE**: Open `.lpi`/`.lpr` files (FreePascal)

Key project files:
- `Test/DelphiASTTest.dproj` / `Test/DelphiASTTest.lpr` — GUI test application
- `Demo/Parser/ParserDemo.dproj` / `Demo/Parser/ParserDemo.lpi` — Parser demo
- `Demo/ProjectIndexer/ProjectIndexerResearch.dproj` — Project indexer demo

Testing is done via the GUI test app which processes all `.pas` files in `Test/Snippets/` (34 test cases covering various Delphi language features) and reports OK/FAILED for each.

## Architecture

**Two-stage parsing pipeline:**

1. **Lexer** (`Source/SimpleParser/SimpleParser.Lexer.pas`) — Tokenizes Delphi source into token stream
2. **Parser** (`Source/SimpleParser/SimpleParser.pas`) — Base recursive descent parser consuming tokens
3. **AST Builder** (`Source/DelphiAST.pas`) — `TPasSyntaxTreeBuilder` extends the parser, builds the AST using a `TNodeStack` (stack-based node construction)

**Core types:**
- `TSyntaxNode` / `TCompoundSyntaxNode` (`Source/DelphiAST.Classes.pas`) — AST node types with attributes, children, and position info
- `DelphiAST.Consts.pas` — 150+ node type constants (`ntMethod`, `ntIdentifier`, `ntAssign`, etc.)
- `DelphiAST.Writer.pas` — `TSyntaxTreeWriter.ToXML()` for XML output
- `DelphiAST.SimpleParserEx.pas` — `TmwSimplePasParEx` bridges SimpleParser to the AST builder
- `DelphiAST.ProjectIndexer.pas` — Analyzes entire Delphi projects across multiple units
- `DelphiAST.Serialize.Binary.pas` — Binary serialization (Delphi only)

**Entry point:** `TPasSyntaxTreeBuilder.Run(FileName, IncludeHandler)` in `DelphiAST.pas`

**FreePascal compatibility:** `Source/FreePascalSupport/` provides polyfills (Diagnostics, IOUtils) and uses git submodules for generic collections and StringBuilder.

## Cross-Compiler Conditionals

Source files use `{$IFDEF FPC}` extensively for FreePascal vs Delphi compatibility. The include file `Source/SimpleParser/SimpleParser.inc` centralizes compiler directives.

## Compiler Version & Conditional Defines

The lexer (`TmwBasePasLex`) has a complete conditional compilation engine. It handles `{$IFDEF}`, `{$IFNDEF}`, `{$IF}`, `{$ELSE}`, `{$ELSEIF}`, `{$ENDIF}`, `{$IFEND}`, `{$DEFINE}`, `{$UNDEF}`, and correctly skips inactive branches. `{$IF}` supports `DEFINED()`, `NOT DEFINED()`, `COMPILERVERSION`, `RTLVERSION`, and `AND`/`OR` chaining.

### Problem

By default, `TPasSyntaxTreeBuilder.Run` calls `InitDefinesDefinedByCompiler`, which seeds the lexer's `FDefines` list with the `VERxxx` and platform symbols of the compiler that **built the parser** — not the compiler version of the code being parsed. This means:

- Parsing D13 code with a D12-built parser incorrectly resolves `{$IF DEFINED(VER370)}` as false
- Code with `{$IF DEFINED(VER210)}...{$ELSEIF DEFINED(VER350)}...{$ELSE} Unsupported {$IFEND}` chains hits the `{$ELSE}` because the parser only has `VER360` (or whatever compiled it), not the versions the code checks for

### Solution

Two additions to `TPasSyntaxTreeBuilder` (in `Source/DelphiAST.pas`):

**`InitDefinesForCompilerVersion(AVersion: Double)`** — clears all defines and synthesizes the correct set for a target compiler version:
- All `VERxxx` symbols from VER90 (Delphi 2) through the target version (e.g. VER370 for D13)
- Standard Windows 64-bit platform defines: `WIN64`, `WIN32`, `MSWINDOWS`, `CPUX64`, `CPUX86`, `CPU64BITS`, `CPU32BITS`, `CONDITIONALEXPRESSIONS`, `UNICODE`, `NATIVECODE`, `CONSOLE`, etc.

**`Run(FileName, InterfaceOnly, IncludeHandler, CompilerVersion: Double)`** — overloaded class method. When `CompilerVersion > 0`, calls `InitDefinesForCompilerVersion` instead of `InitDefinesDefinedByCompiler`.

### Usage

```pascal
// Target Delphi 12 Athens (CompilerVersion 36.0)
Node := TPasSyntaxTreeBuilder.Run('MyUnit.pas', False, IncludeHandler, 36.0);

// Target Delphi 13 Florence (CompilerVersion 37.0)
Node := TPasSyntaxTreeBuilder.Run('MyUnit.pas', False, IncludeHandler, 37.0);

// Use the parser's own compiler version (legacy behavior)
Node := TPasSyntaxTreeBuilder.Run('MyUnit.pas', False, IncludeHandler);
// or explicitly with 0:
Node := TPasSyntaxTreeBuilder.Run('MyUnit.pas', False, IncludeHandler, 0.0);
```

### Instance-level usage

When creating a builder instance directly instead of using the class method:

```pascal
Builder := TPasSyntaxTreeBuilder.Create;
try
  Builder.InitDefinesForCompilerVersion(37.0);  // D13
  Builder.IncludeHandler := IncludeHandler;
  Result := Builder.Run(Stream);
finally
  Builder.Free;
end;
```

### Version mapping

| CompilerVersion | Delphi version | VERxxx |
|-----------------|---------------|--------|
| 36.0 | 12 Athens | VER360 |
| 37.0 | 13 Florence | VER370 |

The formula is `VERxxx` where `xxx = Round(10 * CompilerVersion)`. All versions from VER90 upward are added; the full list is in the `AllVersions` array in `InitDefinesForCompilerVersion`.

### Limitations

- `{$IF COMPILERVERSION >= xxx}` still evaluates against the parser's own `CompilerVersion` constant (compile-time), not the target. Use `{$IF DEFINED(VERxxx)}` patterns instead — these work correctly with the synthesized defines.
- Platform defines are hardcoded to Windows 64-bit. Cross-platform parsing would need a `platform` parameter (not yet implemented).
- `{$IFOPT}` is not supported and always evaluates to false.

## Git Submodules

- `Source/FreePascalSupport/Generics.Collection`
- `Source/FreePascalSupport/FPC_StringBuilder`

Run `git submodule update --init` after cloning if building with FreePascal.

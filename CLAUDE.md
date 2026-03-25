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

## Git Submodules

- `Source/FreePascalSupport/Generics.Collection`
- `Source/FreePascalSupport/FPC_StringBuilder`

Run `git submodule update --init` after cloning if building with FreePascal.

# AGENTS.md – Repository Agent Guidelines

This document provides the standard for how automated coding agents should operate inside this repository. It is designed to be language-agnostic while still offering practical, implementation-oriented guidance for teams that rely on agent-based execution.

Table of contents:
- Build, lint, and test commands
- Code style guidelines
- Import, formatting, and naming conventions
- Error handling and resilience
- Test strategies and running a single test
- Collaboration, notetaking, and verification
- Cursor and Copilot rules (if present)
- Notepad integration and provenance

Note: If a Cursor rules directory (.cursor/rules/ or .cursorrules) or Copilot instructions (.github/copilot-instructions.md) exist, copy and adapt their guidance into the relevant sections below.

---

## 1) Build, lint, test commands
- Detect and use the project’s package/build system automatically where possible.
- When in doubt, prefer conventional, widely-supported commands that work across CI and local dev machines.
- The following commands cover common environments. Customize for your stack where necessary.

- Build commands (choose the most appropriate):
  - If a Makefile exists: make -j$(nproc) all
  - If a CMakeLists.txt exists: cmake -S . -B build && cmake --build build -- -j$(nproc)
  - If a Cargo.toml exists: cargo build
  - If a package.json exists: npm run build || yarn build
  - If a go.mod exists: go build ./...
  - If a meson.build or ninja is used: meson setup build && ninja -C build

- Lint commands (best effort per language):
  - JavaScript/TypeScript: npm run lint || yarn lint
  - Python: pytest -m "lint" or flake8 .
  - C/C++: clang-tidy and/or cppcheck on changed files
  - Go: go vet ./... && golangci-lint run

- Test commands (prefer single-test invocation when possible):
  - JavaScript/TypeScript (Jest): npm test -- --testNamePattern="YourTestName"
  - Python (pytest): pytest -k "your_test_name" -q
  - Go: go test -run YourTestName ./...
  - Rust: cargo test your_test_name
  - Java (JUnit/Maven): mvn -Dtest=YourTestName test
  - C/C++ (Catch2 / GoogleTest): ./build/tests --gtest_filter=YourTestName

- Quick single-test helper (macOS/Linux):
  - If using Jest: npx jest -t "YourTestName" --runInBand
  - If using PyTest: pytest -k "YourTestName" -q
  - If using Go: go test -run YourTestName ./...

- Running a single targeted test from CI helpers:
  - CI: CI=true npm test -- --testNamePattern="YourTestName"

---

## 2) Code style guidelines
- Always follow the repository’s existing style unless a documented exception exists.
- General rules (language-agnostic):
  - Keep modules small and focused; prefer small, well-scoped functions.
  - Prefer explicit error handling and clear failure paths; avoid swallowing errors.
  - Write self-describing identifiers; avoid cryptic abbreviations.
  - Document non-obvious decisions with inline comments or notes in notepads.

- Language-specific conventions (adapt as needed):
  - JavaScript/TypeScript: use ESLint + Prettier; 2-space indentation; camelCase for variables and functions; PascalCase for types/classes; prefer async/await; avoid console.log in production code.
  - Python: follow PEP8; max line length 88; use type hints where helpful; docstrings for public APIs.
  - C/C++: use 4-space indentation; include guards; prefer static inline where appropriate; meaningful variable names; avoid NULL Math. Use explicit casts; handle resources with RAII.
  - Go: gofmt/go vet; explicit error returns; avoid panics in library code; comment exported symbols; idiomatic error wrapping.
- Import/namespacing rules are language-specific but the spirit is universal: avoid circular imports, use stable naming, and group imports logically.
- Types and interfaces: define small, composable interfaces; prefer narrow interfaces; document expectations in notepads.
- Formatting: always run the formatting pass before committing; include a dev-only formatting check in CI if feasible.
- Error semantics: do not use exceptions for control flow; return error values or error objects; propagate errors with context.
- Accessibility and security: do not expose secrets; use environment/config patterns; validate inputs; guard against common injection vectors in all languages.
- Tests: tests should be deterministic, isolated, and fast. Prefer table-driven tests where applicable.

- Naming conventions (quick reference):
  - Go: CamelCase for exported identifiers, mixedCaps for non-exported; short, descriptive names.
  - JS/TS: camelCase for variables and functions, PascalCase for types, UPPER_SNAKE for constants.
  - Python: snake_case for functions/vars, CapWords for classes.
  - C/C++: snake_case or camelCase depending on project conventions; use ALL_CAPS for macros.

- Documentation: always accompany public APIs with minimal docs; link to notepad references when decisions are non-obvious.
- Versioning: follow the repo’s versioning policy (SemVer is common); tag releases when appropriate.

---

## 3) Import, formatting, and naming specifics
- Import ordering per language guidelines; keep standard library imports grouped first, third-party after, local packages last.
- Use linting rules to enforce import ordering, unused imports, and naming conventions.
- For TypeScript, enable noImplicitAny and strict mode; prefer explicit types over any.
- For C/C++, prefer header inclusions guards and forward declarations when possible.
- For Go, organize imports with gofmt; avoid blank imports unless required.
- Maintain consistent file headers documenting purpose and license where applicable.
- Add or update module-level comments for non-trivial files.
- If you introduce new public APIs, update the corresponding Notepad entries to reflect usage expectations.

---

## 4) Error handling and resilience
- Do not crash on recoverable errors; return sensible error messages and codes.
- When critical failures occur, surface a clear message to the user or caller; propagate as part of the error chain.
- Where resources are acquired (files, sockets, memory), ensure deterministic cleanup via RAII, defer/finally, or equivalent.
- Use retries with exponential backoff for transient external calls where appropriate.
- Validate inputs early and fail fast with actionable error messages.
- In tests, verify error paths alongside success paths.

---

## 5) Running a single targeted test (quick reference)
- Jest: npm test -- --testNamePattern="YourTestName"
- PyTest: pytest -k "YourTestName" -q
- Go: go test -run YourTestName ./...
- Rust: cargo test YourTestName
- Java: mvn -Dtest=YourTestName test
- C/C++ (Catch2): ./build/tests --test YourTestName if supported; otherwise use --gtest_filter=YourTestName
- Misc: consult project docs for framework-specific single-test invocations

Tips:
- In CI, set CI=true to disable interactive prompts and enable predictable outputs.
- When possible, run a single-test target locally to speed up iteration.
- Use descriptive test names that map to the feature being tested.

---

## 6) Notepad integration, provenance, and verification
- Notepad conventions:
  - Read: .sisyphus/notepads/{plan-name}/*.md before delegation to extract context.
  - Write: Append to the appropriate category; never overwrite existing notes.
- Plan provenance:
  - Record decisions, trade-offs, and blockers in notepads to facilitate traceability for agents.
- Verification:
  - After each agent action, verify with a lightweight QA pass against changed files and repository state.
- Documentation hygiene:
  - Link AGENTS.md updates to related plan notes to keep the chain of thought coherent.
- Cursor/Copilot presence:
  - If .cursor/rules/ or .cursorrules exist, mirror their guidance here. Same for .github/copilot-instructions.md; include or reference relevant rules.
- Dependencies:
  - This document assumes no external runtime dependencies beyond the repository’s existing tooling.
- End state:
  - This file serves as a contract for agents and as a reference for future onboarding of new team members.

---

## Inherited Wisdom (none available for this plan)
- If you have access to plan notes, copy any explicit conventions here.

---

## Notepad integration notes
- Notepad path template for this plan: .sisyphus/notepads/{plan-name}/
- Current plan name: default-plan (created ad hoc for agent-driven tasks)
- Use these notes to enrich future task prompts and to justify deviations from default patterns.

---

## Version and maintenance
- This file is part of the repository’s agent tooling. Update it as the team’s conventions evolve.
- Do not remove referenced sections unless the conventions have changed; instead, deprecate with a changelog entry in the Notepad.

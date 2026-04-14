---
name: golang-testing
description: "Production-ready Golang testing guide with ByteDance unit-test conventions. Enforces one TestXxx entry per tested method, scenario cases organized inside PatchConvey, strict unit-boundary isolation with external dependencies mocked by github.com/bytedance/mockey, and unified assertions with github.com/smartystreets/goconvey/convey (So + Should*). Covers unit/integration tests, benchmarks, fuzzing, coverage, race checks, goleak, and CI test execution. Use for ANY Go testing task."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: samber
  version: "1.2.0"
  openclaw:
    emoji: "🧪"
    homepage: https://github.com/samber/cc-skills-golang
    requires:
      bins:
        - go
        - gotests
    install:
      - kind: go
        package: github.com/cweill/gotests/gotests@latest
        bins: [gotests]
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent Bash(gotests:*) AskUserQuestion
---

**Persona:** You are a Go engineer who treats tests as executable specifications. You write tests to constrain behavior, not to hit coverage targets.

**Thinking mode:** Use `ultrathink` for test strategy design and failure analysis. Shallow reasoning misses edge cases and produces brittle tests that pass today but break tomorrow.

**Modes:**

- **Write mode** — generate tests for existing or new code. Build one `TestXxx` per method, organize scenarios with `PatchConvey` case blocks, isolate external dependencies with `mockey`, and assert with `So` + `Should*`.
- **Review mode** — review PR test changes. Focus on one-entry test structure, unit-boundary isolation, mock correctness, assertion consistency, and flaky patterns.
- **Audit mode** — audit a test suite for structural and quality gaps. Prioritize: (1) one-entry-per-method and case organization, (2) dependency isolation quality, (3) goroutine leaks/race risks.
- **Debug mode** — failing/flaky tests. Reproduce first, isolate one failing case, validate mock lifecycle, trace root cause in production code or test setup.

> **Community default.** A company skill that explicitly supersedes `samber/cc-skills-golang@golang-testing` skill takes precedence.

# Go Testing Best Practices (ByteDance Conventions)

This skill defines production-ready Go testing rules with ByteDance unit-test conventions.

## Mandatory Rules

1. Each tested method MUST have exactly one `TestXxx` entry function.
2. Different scenarios MUST be organized as cases inside that single `TestXxx` (recommended: nested `PatchConvey("case: ...")`).
3. Unit tests MUST keep clear unit boundaries: test only the current logic block.
4. Dependencies outside the current logic block SHOULD be mocked (other flows/modules/packages, external systems, clocks, random sources, IO).
5. Mocking SHOULD primarily use `github.com/bytedance/mockey`.
6. Assertions SHOULD primarily use `github.com/smartystreets/goconvey/convey` (`So` + `Should*`, including `ShouldResemble`).
7. Do NOT mix multiple assertion libraries in the same test file unless migration is unavoidable.
8. Test structure SHOULD clearly show: setup data -> mock dependencies -> execute logic -> assert results.
9. Tests MUST NOT depend on execution order.
10. Tests using Mockey SHOULD NOT use `t.Parallel()`.
11. Keep unit tests fast and deterministic.
12. Integration tests MUST use build tags (`//go:build integration`).
13. Concurrency-heavy packages SHOULD use `goleak.VerifyTestMain` in `TestMain`.
14. CI SHOULD run tests with race detection.

## Canonical Unit Test Template

Use this as the default organization template:

```go
package user_test

import (
    "errors"
    "testing"

    . "github.com/bytedance/mockey"
    . "github.com/smartystreets/goconvey/convey"
)

func TestUserService_CreateUser(t *testing.T) {
    PatchConvey("TestUserService_CreateUser", t, func() {
        // case 1: setup -> mock -> execute -> assert
        PatchConvey("case: valid input returns nil", func() {
            input := &User{Name: "alice"}

            Mock(validateUser).Return(nil).Build()
            Mock(saveUser).Return(nil).Build()

            err := CreateUser(input)

            So(err, ShouldBeNil)
        })

        // case 2: setup -> mock -> execute -> assert
        PatchConvey("case: validation error is returned", func() {
            input := &User{Name: ""}
            wantErr := errors.New("invalid user")

            Mock(validateUser).Return(wantErr).Build()

            err := CreateUser(input)

            So(err, ShouldResemble, wantErr)
        })
    })
}
```

## Test Structure and Naming

### File Conventions

```go
// package_test.go - tests in same package (white-box)
package mypackage

// mypackage_test.go - tests in external package (black-box, preferred for public API)
package mypackage_test
```

### Naming Conventions

```go
func TestAdd(t *testing.T) { ... }                // one function/method -> one test entry
func BenchmarkAdd(b *testing.B) { ... }           // benchmark
func ExampleAdd() { ... }                         // example
```

Case naming guidelines:

- Use direct, behavior-focused names: `case: empty input returns ErrInvalid`.
- Include condition and expected result.
- Avoid redundant prefixes (`test`, `scenario`, `should`) in every case.

## Unit Boundary Guidance

For a unit test, only verify current logic behavior. Mock dependencies that are outside current logic scope:

- Other package functions/methods
- External systems (DB/cache/MQ/network/filesystem)
- Time/randomness/global state providers
- Side-effectful helper paths not under current assertion target

Do not turn a unit test into an integration flow.

## Mocking with Mockey

### Recommended imports

```go
import (
    . "github.com/bytedance/mockey"
    . "github.com/smartystreets/goconvey/convey"
)
```

### Core patterns

- Basic replacement: `Mock(target).Return(...).Build()`
- Hook behavior: `Mock(target).To(func(...) ... { ... }).Build()`
- Conditional behavior: `Mock(target).When(func(...) bool { ... }).Return(...).Build()`
- Lifecycle management: `PatchConvey(...)` (preferred with GoConvey), `PatchRun(...)` (lightweight alternative)

`PatchConvey` and `PatchRun` automatically release mocks when scope exits.

### Runtime requirements for Mockey

Mockey requires compiler optimizations disabled during test execution.

```bash
go test -gcflags="all=-l -N" ./...
```

Notes:

- If mock does not take effect, first check whether `Build()` is called.
- If mock does not take effect, check target signature/type matching.
- Avoid mocking critical runtime/system internals directly.

## Assertions with GoConvey

Use `So` with `Should*` assertions consistently:

```go
So(err, ShouldBeNil)
So(got, ShouldEqual, want)
So(gotObj, ShouldResemble, wantObj)
So(items, ShouldHaveLength, 3)
```

Keep assertion style uniform in one file. Prefer one assertion family throughout the test.

## Parallel Policy

- For tests that use Mockey: do not use `t.Parallel()`.
- For pure-function tests without shared state and without patching: `t.Parallel()` can be used when helpful.

## Goroutine Leak Detection with goleak

Use `go.uber.org/goleak` to detect leaked goroutines.

```go
import (
    "testing"
    "go.uber.org/goleak"
)

func TestMain(m *testing.M) {
    goleak.VerifyTestMain(m)
}
```

## Test Timeouts

For tests that may hang, use a timeout helper that panics with caller location. See [Helpers](./references/helpers.md).

## Integration Tests

Use build tags to separate integration tests from unit tests:

```go
//go:build integration

package mypackage

func TestDatabaseIntegration(t *testing.T) {
    // real dependency integration test
}
```

Run integration tests explicitly:

```bash
go test -tags=integration ./...
```

For Docker Compose fixtures and schemas, see [Integration Testing](./references/integration-testing.md).

## Benchmarks

See `samber/cc-skills-golang@golang-benchmark` for advanced benchmark workflows.

Benchmark basics:

- Use `b.ReportAllocs()` where allocation tracking matters.
- Cover meaningful input sizes with `b.Run("size=...", ...)`.
- Exclude setup work from timed loops.

## Fuzzing

Use fuzz tests for critical parsing/sanitization/serialization logic.

```go
func FuzzReverse(f *testing.F) {
    f.Add("hello")
    f.Add("")
    f.Fuzz(func(t *testing.T, input string) {
        out := Reverse(Reverse(input))
        So(out, ShouldEqual, input)
    })
}
```

## Examples as Documentation

Keep examples executable with `// Output:` comments.

## Coverage

```bash
go test -coverprofile=coverage.out ./...
go tool cover -html=coverage.out
go tool cover -func=coverage.out
```

## Enforce with Linters

Use test-focused linters (for example `thelper`, `paralleltest`, `testifylint`) as automated guardrails where applicable.

## Cross-References

- -> Mock patterns and anti-patterns: [Mocking](./references/mocking.md)
- -> HTTP handler tests: [HTTP Testing](./references/http-testing.md)
- -> Integration fixtures and setup: [Integration Testing](./references/integration-testing.md)
- -> Timeout helper: [Helpers](./references/helpers.md)
- -> CI workflows: `samber/cc-skills-golang@golang-continuous-integration`

## Quick Reference

```bash
go test ./...                                  # all tests

# Required when tests use mockey
# disable inlining and optimization so patching works reliably
go test -gcflags="all=-l -N" ./...

# run specific tests
go test -run TestName ./...
go test -run TestName/case ./...

# quality checks
go test -race ./...
go test -cover ./...

# benchmark/fuzz/integration
go test -bench=. -benchmem ./...
go test -fuzz=FuzzName ./...
go test -tags=integration ./...
```

## Legacy Note

Existing projects may still contain `testify`-style tests. For new or revised tests under this skill, prefer the Mockey + GoConvey path above and avoid mixed assertion styles in a single test file.

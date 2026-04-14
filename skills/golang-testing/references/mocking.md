# Mocking and Test Fixtures

## Preferred Stack

For this skill, prefer:

- Mocking: `github.com/bytedance/mockey`
- Assertions and test organization: `github.com/smartystreets/goconvey/convey`

Avoid mixing multiple assertion styles in the same test file.

## Standard Test Skeleton

Use one `TestXxx` per tested method, and put all scenarios in nested case blocks:

```go
package service_test

import (
    "errors"
    "testing"

    . "github.com/bytedance/mockey"
    . "github.com/smartystreets/goconvey/convey"
)

func TestOrderService_Submit(t *testing.T) {
    PatchConvey("TestOrderService_Submit", t, func() {
        PatchConvey("case: valid request returns nil", func() {
            req := &OrderReq{UserID: 1, Amount: 100}

            Mock(validateReq).Return(nil).Build()
            Mock(createOrder).Return(int64(101), nil).Build()
            Mock(publishEvent).Return(nil).Build()

            err := Submit(req)

            So(err, ShouldBeNil)
        })

        PatchConvey("case: validate failed returns original error", func() {
            req := &OrderReq{UserID: 0, Amount: 100}
            wantErr := errors.New("invalid user")

            Mock(validateReq).Return(wantErr).Build()

            err := Submit(req)

            So(err, ShouldResemble, wantErr)
        })
    })
}
```

Recommended per-case structure:

1. Setup data
2. Mock dependencies
3. Execute logic
4. Assert result

## Unit Boundary Checklist

Before writing a case, decide if it is a true unit test:

- Is the behavior under assertion inside the current logic block?
- Are external dependencies mocked (DB/cache/MQ/network/filesystem/other modules)?
- Is time/randomness deterministic (mocked or fake clock)?
- Is the case free of unrelated upstream/downstream flow assertions?

If any answer is "no", split or refactor the test.

## Mockey Patterns

### Basic return

```go
Mock(targetFn).Return(result1, result2).Build()
```

### Hook replacement

```go
Mock(targetFn).To(func(arg1 string) error {
    return nil
}).Build()
```

### Conditional mocking

```go
Mock(targetFn).
    When(func(v int) bool { return v < 0 }).Return(errInvalid).
    When(func(v int) bool { return v == 0 }).Return(nil).
    Build()
```

### Lifecycle control

- Prefer `PatchConvey` when using GoConvey assertions.
- Use `PatchRun` for lightweight lifecycle scopes without GoConvey context.

## Required Runtime Flag for Mockey

Mockey patching depends on disabled inlining/optimization during tests:

```bash
go test -gcflags="all=-l -N" ./...
```

If mocks do not work, check in order:

1. `Build()` called
2. mock target signature/type match
3. command includes `-gcflags="all=-l -N"`

## Assertion Checklist

Prefer these `So` assertions:

- `ShouldEqual` for scalar equality
- `ShouldResemble` for deep object equality
- `ShouldBeNil` / `ShouldNotBeNil` for nil checks
- `ShouldBeTrue` / `ShouldBeFalse` for booleans
- `ShouldContainSubstring` / `ShouldHaveLength` for collections/strings

Keep one assertion family per file.

## Anti-pattern: Mixed Assertion Styles

Bad example (mixed libraries):

```go
func TestUserService_Get(t *testing.T) {
    PatchConvey("TestUserService_Get", t, func() {
        user, err := GetUser(1)

        So(err, ShouldBeNil)
        assert.NotNil(t, user) // mixed assertion style
    })
}
```

Good example (single style):

```go
func TestUserService_Get(t *testing.T) {
    PatchConvey("TestUserService_Get", t, func() {
        user, err := GetUser(1)

        So(err, ShouldBeNil)
        So(user, ShouldNotBeNil)
    })
}
```

## Parallel Execution Policy

- When using Mockey in a test/case: do not call `t.Parallel()`.
- For pure functions without patching and without shared state: `t.Parallel()` is acceptable.

## Time Control

Prefer fake clocks for time-dependent behavior (for example `clockwork`) instead of real `time.Sleep()`.
Keep unit tests deterministic and fast.

## Fixtures

Create reusable fixtures in dedicated files/packages when data setup is repeated. Keep fixture builders deterministic and explicit.

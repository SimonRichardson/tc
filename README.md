# tc

`github.com/canononical/tc` is a suite-based testing framework for Go, built on
top of `testing.T`. It is a derivative of [gocheck](https://gopkg.in/check.v1)
and provides a rich set of checkers, generic `Must*` helpers, and a lightweight
suite runner.

Tests within a suite are run in randomised order to prevent order-dependent
failures.

## Installation

```sh
go get github.com/canonical/tc
```

## Usage

### Suite

Define a suite struct, register lifecycle hooks, and call `tc.Run`:

```go
type MySuite struct{}

func TestMySuite(t *testing.T) {
    tc.Run(t, &MySuite{})
}

func (s *MySuite) SetUpSuite(c *tc.C)    {}
func (s *MySuite) TearDownSuite(c *tc.C) {}
func (s *MySuite) SetUpTest(c *tc.C)     {}
func (s *MySuite) TearDownTest(c *tc.C)  {}

func (s *MySuite) TestSomething(c *tc.C) {
    c.Assert(1+1, tc.Equals, 2)
}
```

Lifecycle methods accept either `*tc.C` or `*testing.T`.

### Assertions

Both `Check` (non-fatal) and `Assert` (fatal) are available as methods on
`*tc.C` or as package-level functions accepting `tc.LikeTB`.

```go
c.Assert(err, tc.ErrorIsNil)
c.Check(val, tc.Equals, "expected")
tc.Assert(t, err, tc.ErrorIsNil)
```

An optional `tc.Commentf` may be appended for extra context on failure.

### Checkers

| Checker | Description |
|---|---|
| `Equals` | `==` equality |
| `DeepEquals` | deep equality (nil slice == empty slice, `time.Time` zone-agnostic) |
| `IsNil` / `NotNil` | nil checks |
| `ErrorIsNil` | asserts error is nil; distinguishes typed nils |
| `ErrorIs` | wraps `errors.Is` |
| `ErrorMatches` | error message matches regex |
| `Matches` | string or `Stringer` matches regex |
| `HasPrefix` / `HasSuffix` / `Contains` | string predicates |
| `HasLen` | length check |
| `SameContents` | slice contains same elements regardless of order |
| `Panics` / `PanicMatches` | panic assertions |
| `FitsTypeOf` | assignability check |
| `Implements` | interface implementation check |
| `TimeBetween` | `time.Time` is within a range |
| `DurationLessThan` | `time.Duration` comparison |
| `LessThan` / `GreaterThan` | numeric less-than / greater-than comparison |
| `OrderedLeft` / `OrderedRight` / `OrderedMatch` | typed slice ordering checks |
| `UnorderedMatch` | typed slice unordered match |

### Composable Checkers

`Not` negates a checker. `And` requires all checkers to pass. `Or` requires
at least one checker to pass. Use `Bind` to pre-fill arguments when
composing checkers that take an expected value.

```go
c.Assert(x, tc.Not(tc.IsNil))

c.Assert(n, tc.And(
    tc.Bind(tc.GreaterThan, 0),
    tc.Bind(tc.LessThan, 100),
))

c.Assert(s, tc.Or(
    tc.Bind(tc.Equals, "foo"),
    tc.Bind(tc.Equals, "bar"),
))
```

### Bind

`Bind` pre-fills checker arguments so a checker can be used as a matcher:

```go
b := tc.Bind(tc.Equals, "expected")
c.Assert("expected", b)
b.Matches("expected") // true
```

`Binding` satisfies the `gomock.Matcher` interface (`Matches` and `String`),
so a bound checker can be passed directly as a gomock `EXPECT` argument:

```go
ctrl := gomock.NewController(t)
defer ctrl.Finish()
mock := NewMockFoo(ctrl)
mock.EXPECT().Bar(tc.Bind(tc.DeepEquals, wantArg))
mock.EXPECT().Count(tc.Bind(tc.GreaterThan, 0))
```

### MultiChecker

`MultiChecker` performs a deep-equality comparison by default, but lets you
override the checker for specific fields using Go path expressions. Use `_`
as a wildcard for any field name or slice index. All expressions begin with `_`
as the root object of the comparison.

```go
mc := tc.NewMultiChecker()
mc.AddExpr(`_[_].UUID`, tc.IsNonZeroUUID)
c.Assert(sliceOfEntities, mc, expectedSliceOfEntities)
```

Every field is compared with deep equality except each element's `UUID`,
which only needs to be non-zero, it does not have to match the value in
`expectedSliceOfEntities`.

Path expressions follow Go selector syntax:

| Expression | Matches |
|---|---|
| `_.Name` | `Name` field of the top-level value |
| `_[_].Name` | `Name` field of every element in a top-level slice |
| `_.Nested.Value` | a nested field path |
| `_[0].Value` | `Value` field of the first slice element only |

Multiple rules can be added with chained `AddExpr` calls:

```go
mc := tc.NewMultiChecker().
    AddExpr(`_[_].UUID`, tc.IsNonZeroUUID).
    AddExpr(`_[_].CreatedAt`, tc.Not(tc.IsNil))
```

#### Using the expected value

Pass `tc.ExpectedValue` as a checker argument to substitute the corresponding
expected value at that path:

```go
mc := tc.NewMultiChecker()
// Require Name to have a specific prefix, but still match the expected suffix.
mc.AddExpr(`_.Name`, tc.HasPrefix, "svc-")
mc.AddExpr(`_.Name`, tc.Equals, tc.ExpectedValue)
```

#### Overriding slice length

Wrap the path in `len(...)` to override how the length of a slice is checked:

```go
mc := tc.NewMultiChecker()
mc.AddExpr(`len(_.Items)`, tc.GreaterThan, 0)
```

#### Changing the default checker

`NewMultiCheckerWithDefault` replaces deep equality with another checker for
all fields that have no explicit rule. `tc.Ignore` is useful when you only
care about a small number of fields:

```go
mc := tc.NewMultiCheckerWithDefault(tc.Ignore)
mc.AddExpr(`_[_].ID`, tc.IsNonZeroUUID)
mc.AddExpr(`_[_].Name`, tc.Equals, tc.ExpectedValue)
c.Assert(results, mc, expected)
```

### Must helpers

Generic helpers that call a function and assert the error return is nil:

```go
val := tc.Must(c, os.Open, "file.txt")    // func(A) (T, error)
tc.Must0_0(c, os.Remove, "file.txt")      // func(A) error
r1, r2 := tc.Must0_2(c, twoValFunc)      // func() (T, T2, error)
```

The naming scheme is `Must{inputs}_{outputs}` where the error is not counted.

### TBC

`TBC` wraps a `testing.TB` and implements `LikeC`, allowing checkers to be
used outside a suite:

```go
func TestSomething(t *testing.T) {
    tbc := &tc.TBC{TB: t}
    tbc.Assert(1+1, tc.Equals, 2)
}
```

## Licences

The suite runner and checker infrastructure is derived from
[gocheck](https://gopkg.in/check.v1) (BSD 2-Clause). The `DeepEqual`
implementation is derived from the Go standard library (BSD-style).
Additional code is licensed under LGPLv3. See `LICENSE`, `LICENSE-gocheck`,
and `LICENSE-golang`.

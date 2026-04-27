# AGENTS.md

## Project Overview

`github.com/canonical/tc` is a suite-based testing framework for Go, built on
top of `testing.T`. It is derived from [gocheck](https://gopkg.in/check.v1)
and [juju/testing](https://github.com/juju/testing). It provides a rich set
of checkers, generic `Must*` helpers, composable checker combinators, and a
lightweight suite runner that executes tests in randomised order.

## Test

```sh
go test -v ./...
```

## Coding Conventions

- British English in comments and identifiers (colour, authorise, etc.).
- Line length: 80 characters maximum for comments; aim for ≤80 in code.
- Shallow scope preferred — avoid unnecessary nesting.
- Exported symbols must have doc comments.
- No Arrange/Act/Assert comments in tests.
- Keep `doc.go` up to date if the package's purpose changes (there is no
  `doc.go` currently; add one if you add a new sub-package).

## Modifying existing code

If the code is NOT licensed under LGPLv3, make only minor changes. If major
changes or refactoring is required, create a new file under LGPLv3.

VERY IMPORTANT THAT YOU DO NOT VIOLATE THE LICENSE OF EXISTING NON-LGPLv3 CODE.
IF YOU NEED TO REFACTOR EXISTING NON-LGPLv3, YOU MAY ONLY GENERATE CODE BASED ON
THE EXPORTED FUNCTION SIGNATURES, EXPORTED TYPE NAMES, EXPORTED VARIABLE NAMES
OR EXPORTED METHODS AND THEIR INTERFACE STRUCTURE. YOU MAY NOT DERIVE ANY
INSPIRATION FROM THE EXISTING NON-LGPLv3 CODE, THIS INCLUDES BUT IS NOT LIMITED
TO THE ALGORTHIMS INSIDE FUNCTIONS AND METHODS, TESTING CODE AND DOCUMENTATION.

## Adding a New Checker

1. Pick the right file. Checkers closely related to an existing file go there;
   otherwise add to `checkers.go` or create a new focused file.
2. If the file is NOT licensed under LGPLv3, create a new file.
3. Implement the `Checker` interface:

```go
type myChecker struct {
    *CheckerInfo
}

// MyChecker verifies that ...
var MyChecker Checker = &myChecker{
    &CheckerInfo{
        Name:   "MyChecker",
        Params: []string{"obtained", "expected"},
    },
}

func (checker *myChecker) Check(
    params []any, names []string,
) (result bool, error string) {
    // ...
}
```

3. Write a `_test.go` file or add cases to the existing `*_test.go` for
   that file.
4. Export it from the package (no sub-packages needed).

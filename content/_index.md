---
title: go-error
---

`go-error` is the gomatic ecosystem's **constant/sentinel error mechanism** for Go. It provides a single string-backed error type — [`errs.Const`](https://pkg.go.dev/github.com/gomatic/go-error#Const) — whose values are declared as constants and matched with [`errors.Is`](https://pkg.go.dev/errors#Is), never by string comparison. Its `.With(cause, args...)` method wraps a cause with `%w` so the result stays matchable against **both** the sentinel and the wrapped cause.

- **Source:** [`gomatic/go-error`](https://github.com/gomatic/go-error)
- **API reference:** [pkg.go.dev/github.com/gomatic/go-error](https://pkg.go.dev/github.com/gomatic/go-error)

## Install

```sh
go get github.com/gomatic/go-error
```

## Why constant errors

Comparing errors by their message string is brittle: a reworded message silently breaks every caller that matched on it. `errs.Const` makes each error a typed constant, so callers match the _identity_ of the error, not its text — and the compiler keeps the value in one place.

The library owns the **mechanism only** — it ships no error values. Every consumer declares its own sentinels in its own package:

```go
import errs "github.com/gomatic/go-error"

// Declare every error a package can emit as a const of errs.Const.
const (
	ErrOpenFile errs.Const = "failed to open file"
	ErrReadFile errs.Const = "failed to read file"
)
```

## Usage

### Declare and match

```go
package main

import (
	"errors"
	"fmt"

	errs "github.com/gomatic/go-error"
)

const ErrFoo errs.Const = "foo failed"

func do(cause error) error {
	// Wrap the cause; the result still matches ErrFoo (and cause) under errors.Is.
	return ErrFoo.With(cause)
}

func main() {
	err := do(errors.New("disk full"))
	fmt.Println(errors.Is(err, ErrFoo)) // true
}
```

### Wrapping a cause

`With` joins the sentinel and a non-nil cause with `%w`, so **both** remain recoverable:

```go
err := ErrOpenFile.With(cause)
errors.Is(err, ErrOpenFile) // true — the sentinel
errors.Is(err, cause)       // true — the wrapped cause
```

A `nil` cause produces the bare sentinel.

### Adding context

Trailing args render space-separated and are appended after the cause, so callers pass clean key/value pairs without baking separators into the message:

```go
err := ErrOpenFile.With(cause, "file", path)
// "failed to open file: <cause>: file <path>"
```

## Design

- **`Const` is a `string` newtype** implementing `error` via `Error() string` — a value type, safe to copy and compare; sentinels are untyped string constants.
- **No allocation for the bare sentinel** — returning `ErrFoo.With(nil)` is just the constant itself.
- **Dependency-light** — the package depends only on the standard library (`errors`/`fmt`).

## Who uses it

Every gomatic Go project declares its sentinels with `errs.Const`: [`renderizer`](https://github.com/gomatic/renderizer), [`template.cli`](https://github.com/gomatic/template.cli), and the other [`gomatic/go-*`](https://github.com/orgs/gomatic/repositories?q=go-) libraries.

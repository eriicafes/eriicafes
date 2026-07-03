---
title: "Fluent dependency injection in Go"
description: "Got is a typesafe, automatic, lazy way to wire dependencies in Go, with zero code generation."
pubDate: 2026-07-01
---

This is what using [Got](https://github.com/eriicafes/got) looks like:

```go
var GetPrinter = got.Using(func(c *got.Container) Printer {
	return &CapsPrinter{}
})

var GetOffice = got.Using(func(c *got.Container) *Office {
	return &Office{
		Printer: GetPrinter.From(c),
	}
})

func main() {
	c := got.New()
	office := GetOffice.From(c)

	office.Printer.Print("hello")
}
```

Get the printer from the container. Get the office from the container. That's the entire idea — dependency injection that reads like the sentence you'd use to describe it, with the compiler checking every word.

The types behind those constructors are regular Go, no different from what you'd write without any of this:

```go
type Printer interface {
	Print(string) string
}

type CapsPrinter struct{}

func (*CapsPrinter) Print(s string) string {
	return strings.ToUpper(s)
}

type Office struct {
	Printer Printer
}
```

## The idea

There are a few dependency injection libraries in the Go ecosystem already, and each solves part of the problem, at a cost. `wire` generates code at build time to stay typesafe. `dig` and `fx` skip code generation but lean on reflection, so a missing binding only surfaces at runtime. Other Go devs just wire everything by hand and give up automation entirely.

`got` aims for **typesafe**, **automatic**, **lazy** wiring with **zero code generation** and none of those tradeoffs: a constructor is defined for each type, colocating how it resolves its own dependencies with the type's own definition, instead of registering bindings somewhere else.

A single container instance is created once and passed into the entrypoint. Requesting any type from that container builds out its entire dependency graph on demand.

## Automatic and lazy

```go
c := got.New()
office := GetOffice.From(c)
```

Calling `GetOffice.From(c)` is what triggers `GetOffice`'s constructor — nothing runs until a dependency is actually requested. Inside it, `GetPrinter.From(c)` triggers `GetPrinter`'s constructor in turn, and the value it returns gets cached on the container. Call `GetPrinter.From(c)` again anywhere else in the graph and the container hands back that same cached instance instead of running the constructor a second time.

## Opting out of caching

Not everything should be a singleton. Swap `.From(c)` for `.New(c)` on any dependency to get a fresh instance every time, without touching anything else in the graph:

```go
var GetOffice = got.Using(func(c *got.Container) *Office {
	return &Office{
		// a new printer is created each time
		Printer: GetPrinter.New(c),
	}
})
```

## Constructors that can fail

Constructors can fail by returning an error alongside the value. `got.Using2` supports that two-return-value shape directly, so an error-returning constructor is still just a variable:

```go
var GetBadOffice = got.Using2(func(c *got.Container) (*Office, error) {
	return nil, fmt.Errorf("failed to create office")
})
```

## Swapping dependencies for tests

Tests can build their own container, override the constructors they want to replace on it with `got.Mock`, and pass that container into the entrypoint instead of the real one — everything downstream that depends on the original constructor picks up the substitute:

```go
type MockPrinter struct{}

func (*MockPrinter) Print(s string) string {
	return fmt.Sprintf("mocked %s", s)
}

func main() {
	mc := got.New()
	got.Mock(mc, GetPrinter, &MockPrinter{})
	office := GetOffice.From(mc)

	office.Printer.Print("hello") // uses MockPrinter
}
```

`got.Mock` takes the replacement instance directly. A constructor can be used when the mock itself needs something from the container:

```go
var GetMockPrinter = got.Using(func(c *got.Container) Printer {
	return &MockPrinter{Logger: GetLogger.From(c)}
})

got.Mock(mc, GetPrinter, GetMockPrinter.New(mc))
```

## Circular dependencies aren't a runtime problem

Other tools have to detect circular dependencies for you, at build time or at runtime. In `got`, constructors are global variables referencing each other, so a cycle is a cycle the Go compiler already refuses to build. The safety net comes free.

That's the whole pitch: dependencies that wire themselves, resolve lazily, and are guarded by the type system. `go get github.com/eriicafes/got` and see for yourself.

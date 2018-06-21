# Wire: Automated Initialization in Go

Wire is a code generation tool that automates connecting components using
[dependency injection][]. Dependencies between components are represented in
Wire as function parameters, encouraging explicit initialization instead of
global variables. Because Wire operates without runtime state or reflection,
code written to be used with Wire is useful even for hand-written
initialization.

[dependency injection]: https://en.wikipedia.org/wiki/Dependency_injection

## Basics

Wire has two core concepts: providers and injectors.

### Defining Providers

The primary mechanism in Wire is the **provider**: a function that can
produce a value. These functions are ordinary Go code.

```go
package foobarbaz

type Foo struct {
	X int
}

// ProvideFoo returns a Foo.
func ProvideFoo() Foo {
	return Foo{X: 42}
}
```

Provider functions must be exported in order to be used from other packages,
just like ordinary functions.

Providers can specify dependencies with parameters:

```go
package foobarbaz

// ...

type Bar struct {
	X int
}

// ProvideBar returns a Bar: a negative Foo.
func ProvideBar(foo Foo) Bar {
	return Bar{X: -foo.X}
}
```

Providers can also return errors:

```go
package foobarbaz

import (
	"context"
	"errors"
)

// ...

type Baz struct {
	X int
}

// ProvideBaz returns a value if Bar is not zero.
func ProvideBaz(ctx context.Context, bar Bar) (Baz, error) {
	if bar == 0 {
		return 0, errors.New("cannot provide baz when bar is zero")
	}
	return Baz{X: bar.X}, nil
}
```

Providers can be grouped into **provider sets**. This is useful if several
providers will frequently be used together. To add these providers to a new set
called `SuperSet`, use the `wire.NewSet` function:

```go
package foobarbaz

import (
	// ...
	"github.com/google/go-x-cloud/wire"
)

// ...

var SuperSet = wire.NewSet(ProvideFoo, ProvideBar, ProvideBaz)
```

You can also add other provider sets into a provider set.

```go
package foobarbaz

import (
	// ...
	"example.com/some/other/pkg"
)

// ...

var MegaSet = wire.NewSet(SuperSet, pkg.OtherSet)
```

### Injectors

An application wires up these providers with an **injector**: a function that
calls providers in dependency order. With Wire, you write the injector's
signature, then Wire generates the function's body.

An injector is declared by writing a function declaration whose body is a call
to `wire.Build`. The return values don't matter as long as they are of the
correct type. The values themselves will be ignored in the generated code. Let's
say that the above providers were defined in a package called
`example.com/foobarbaz`. The following would declare an injector to obtain a
`Baz`:

```go
// +build wireinject
// The build tag makes sure the stub is not built in the final build.

package main

import (
	"context"

	"github.com/google/go-x-cloud/wire"
	"example.com/foobarbaz"
)

func initializeApp(ctx context.Context) (foobarbaz.Baz, error) {
	wire.Build(foobarbaz.MegaSet)
	return foobarbaz.Baz{}, nil
}
```

Like providers, injectors can be parameterized on inputs (which then get sent to
providers) and can return errors. Arguments to `wire.Build` are the same as
`wire.NewSet`: they form a provider set. This is the provider set that gets
used during code generation for that injector.

Any non-injector declarations found in a file with injectors will be copied into
the generated file.

You can generate the injector by invoking `gowire` in the package directory:

```shell
gowire
```

Wire will produce an implementation of the injector in a file called
`wire_gen.go` that looks something like this:

```go
// Code generated by gowire. DO NOT EDIT.

//go:generate gowire
//+build !wireinject

package main

import (
	"example.com/foobarbaz"
)

func initializeApp(ctx context.Context) (foobarbaz.Baz, error) {
	foo := foobarbaz.ProvideFoo()
	bar := foobarbaz.ProvideBar(foo)
	baz, err := foobarbaz.ProvideBaz(ctx, bar)
	if err != nil {
		return 0, err
	}
	return baz, nil
}
```

As you can see, the output is very close to what a developer would write
themselves. Further, there is little dependency on Wire at runtime: all of the
written code is just normal Go code, and can be used without Wire.

Once `wire_gen.go` is created, you can regenerate it by running [`go generate`].

[`go generate`]: https://blog.golang.org/generate

## Advanced Features

The following features all build on top of the concepts of providers and
injectors.

### Binding Interfaces

Frequently, dependency injection is used to bind a concrete implementation for
an interface. Wire matches inputs to outputs via [type identity][], so the
inclination might be to create a provider that returns an interface type.
However, this would not be idiomatic, since the Go best practice is to [return
concrete types][]. Instead, you can declare an interface binding in a provider
set:

```go
type Fooer interface {
	Foo() string
}

type Bar string

func (b *Bar) Foo() string {
	return string(*b)
}

func ProvideBar() *Bar {
	b := new(Bar)
	*b = "Hello, World!"
	return b
}

var BarFooer = wire.NewSet(
	ProvideBar,
	wire.Bind(new(Fooer), new(Bar)))
```

The first argument to `wire.Bind` is a pointer to a value of the desired
interface type and the second argument is a zero value of the concrete type.
Any set that includes an interface binding must also have a provider in the
same set that provides the concrete type.

[type identity]: https://golang.org/ref/spec#Type_identity
[return concrete types]: https://github.com/golang/go/wiki/CodeReviewComments#interfaces

### Struct Providers

Structs can also be marked as providers. Instead of calling a function, an
injector will fill in each field using the corresponding provider. For a given
struct type `S`, this would provide both `S` and `*S`. For example, given the
following providers:

```go
type Foo int
type Bar int

func ProvideFoo() Foo {
	// ...
}

func ProvideBar() Bar {
	// ...
}

type FooBar struct {
	Foo Foo
	Bar Bar
}

var Set = wire.NewSet(
	ProvideFoo,
	ProvideBar,
	FooBar{})
```

A generated injector for `FooBar` would look like this:

```go
func injectFooBar() FooBar {
	foo := ProvideFoo()
	bar := ProvideBar()
	fooBar := FooBar{
		Foo: foo,
		Bar: bar,
	}
	return fooBar
}
```

And similarly if the injector needed a `*FooBar`.

### Binding Values

Occasionally, it is useful to bind a basic value (usually `nil`) to a type.
Instead of having injectors depend on a throwaway provider function, you can
add a value expression to a provider set.

```go
type Foo struct {
	X int
}

func injectFoo() Foo {
	wire.Build(wire.Value(Foo{X: 42}))
	return Foo{}
}
```

The generated injector would look like this:

```go
func injectFoo() Foo {
	foo := Foo{X: 42}
	return foo
}
```

It's important to note that the expression will be copied, so references to
variables will be evaluated during the call to the injector. `gowire` will emit
an error if the expression calls any functions.

### Cleanup functions

If a provider creates a value that needs to be cleaned up (e.g. closing a file),
then it can return a closure to clean up the resource. The injector will use
this to either return an aggregated cleanup function to the caller or to clean
up the resource if a provider called later in the injector's implementation
returns an error.

```go
func provideFile(log Logger, path Path) (*os.File, func(), error) {
	f, err := os.Open(string(path))
	if err != nil {
		return nil, nil, err
	}
	cleanup := func() {
		if err := f.Close(); err != nil {
			log.Log(err)
		}
	}
	return f, cleanup, nil
}
```

A cleanup function is guaranteed to be called before the cleanup function of any
of the provider's inputs and must have the signature `func()`.

### Alternate Injector Syntax

If you grow weary of writing `return foobarbaz.Foo{}, nil` at the end of your
injector function declaration, you can instead write it more concisely with a
`panic`:

```go
func injectFoo() Foo {
	panic(wire.Build(/* ... */))
}
```

## Best Practices

The following are practices we recommend for using Wire. This list will grow
over time.

### Distinguishing Types

If you need to inject a common type like `string`, create a new string type
to avoid conflicts with other providers. For example:

```go
type MySQLConnectionString string
```

### Options Structs

A provider function that includes many dependencies can pair the function with
an options struct.

```go
type Options struct {
	// Messages is the set of recommended greetings.
	Messages []Message
	// Writer is the location to send greetings. nil goes to stdout.
	Writer io.Writer
}

func NewGreeter(ctx context.Context, opts *Options) (*Greeter, error) {
	// ...
}

var GreeterSet = wire.NewSet(Options{}, NewGreeter)
```

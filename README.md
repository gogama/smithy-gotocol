# smithy-gen
Generates Go code from AWS Labs' Smithy API modelling language

---

Next steps.

1. Validate changing generic handler function signature.
2. Design repo and package layout for runtime components and note
   whether they are part of smithy-gen or a different repo.

---

Sketch idea follows.

1. The API Gateway/Lambda integration will be mostly library based, so
   the generator will have a "runtime library" as well as a generator
   component.
2. It may be possible to generify the entire runtime interface to have
   it be a "run anywhere" system, _i.e._ just as easily run as a plain
   HTTP server.
3. Models will depend on a fast low allocation JSON serialize/deserialize,
   hopefully a library but if necessary code generation.
4. Model ser/deser should
5. The runtime system should support a "filter chain" or "handler chain"
   type system where client software plugs in to do things like: massage
   input payload; override ser/deser; override validation; handle the
   actual request; translate errors to model; _etc._
6. The runtime system will have its own encapsulation of the HTTP error
   schema. Handlers should return the runtime's errors, and then they
   can use a plug-in filter to translate the runtime's errors to their
   specific model. Sometimes that can probably be automated, _e.g._ if
   it's just a matter of copying a message and setting a status code.
7. Code generator will generate constant-y stuff for `operation` shapes
   and possibly also `service` shapes. The `operation` shapes will know
   their input and output structure types. This should allow us to
   initialize the runtime (_e.g._ Lambda handler for APIG events) without
   having to _generate_ handler code.

---

A few tricky points are:

1. System needs to support `httpPayload` trait on input and output, in a
   way that is sensible, keeping in mind it might contain a `blob` or a
   `document` or a `string` or anything.


---

Sketch idea for `operation` shapes and runtime init.

Runtime library includes:

```go
package runtime

import (
	"net/http"
	"reflect"
)

type ValidationError interface {
	error
	// TODO: Fields for identifying the specific error(s).
}

type Structure interface {
    SerForHTTP([]byte, http.Header) error
	DeserForHTTP([]byte, http.Header) error
	Validate() ValidationError
}

type Operation[Input, Output Structure] struct {
    inputType, outputType reflect.Type
	Method string
	URI string
	Success int
}

func [Input, Output Structure] NewOperation(i Input, o Output, method, uri string, success int) Operation {
	return Operation{
		inputType: reflect.TypeOf(i),
		outputType: reflect.TypeOf(o),
		Method: method,
		URI: uri,
		Success: success,
    }
}

type Orchestrator struct { // Maybe a better name is Server?
	// TODO: (1) Design a mapping and invoking system that allows the orchestrator
	//           to instantiate the structure type AND, more importantly (because
	//           it is liable to be more difficult) call the handler function in
	//           a way that works with generics.
	//
	// TODO: (2) Design handler plugin system.
}

func (o *Orchestrator) [Input, Output Structure] AddOperation(o Operation[Input, Output], handler func(i Input) (Output, error)) {
	// Somehow this will install handler as a handler for operation o.
	//
	// The way this will work in practice is that users will init the
	// library by calling AddOperation with pointers to the generated
	// struct types. This works because the generated struct types
	// will implement the Structure interface. This will work well
	// as per this trivial playground: https://go.dev/play/p/4jQBmYXNqIl
	//
	// To make the installation work, the handler might need to
	// know the reflect.Type of the input and output types, which it
	// can take from the Operation. The handler also needs to know the
	// HTTP URI path and method for the purpose of routing (e.g. in
	// Lambda handler) and it can take that from the operation too.
	//
	// To store handler in a way that works, might be able to morph
	// handler from func(i Input) (Output, error) to func(i Structure) (Structure, error)
	// using an "unsafe" technique like: https://stackoverflow.com/a/22933640/1911388.
	// This would make it easier for Orchestrator to generically call any
	// handler using the same code, and to manipulate it for ser/deser
	// without having to know which type it is.
}
```

Smithy:

```smithy
operation Foo {
    input:  FooInput
    output: BadIdeaSharedOutput
}

@input
structure FooInput {
    
}

@output
structure BadIdeaSharedOutput {

}
```

To GoLang generated code:

```go
package generated

var (
	// Foo is a straight variable that is instantiated to know everything
	// about the Foo operation. 
	Foo = runtime.NewOperation(&FooInput{}, &BadIdeaSharedOutput{}, "POST", "/foo", 200)
)

type FooInput struct {
	// Will have methods to implement Structure interface.
}

type BadIdeaSharedOutput struct {
   // Will have methods to implement Structure interface.
}
```

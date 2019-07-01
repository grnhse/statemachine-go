<img alt="StateMachine" src="https://user-images.githubusercontent.com/39792/29720866-d95eda6a-89d8-11e7-907e-6f0edd7f4ee3.png" width="538" height="72"/>

StateMachine supports creating productive State Machines In Go

[![GoDoc](https://godoc.org/github.com/Gurpartap/statemachine-go?status.svg)](https://godoc.org/github.com/Gurpartap/statemachine-go)

<!-- TOC -->

- [TODO: check this https://www.w3.org/TR/scxml/](#todo-check-this-httpswwww3orgtrscxml)
	- [Introduction](#introduction)
		- [Further Reading](#further-reading)
		- [Project Goals](#project-goals)
		- [Project Status](#project-status)
	- [Installation](#installation)
	- [Usage](#usage)
		- [States and Initial State](#states-and-initial-state)
		- [Events](#events)
		- [Transitions](#transitions)
		- [Transition Guards (Conditions)](#transition-guards-conditions)
		- [State Callbacks](#state-callbacks)
			- [Before State](#before-state)
		- [Transition Callbacks](#transition-callbacks)
			- [Before Transition](#before-transition)
			- [Around Transition](#around-transition)
			- [After Transition](#after-transition)
			- [After Failure](#after-failure)
		- [Matchers](#matchers)
			- [Event Transition Matchers](#event-transition-matchers)
			- [Transition Callback Matchers](#transition-callback-matchers)
			- [Callback Functions](#callback-functions)
	- [About](#about)

<!-- /TOC -->

## Introduction

State machines provide an alternative way of thinking about how your workflows
may be implemented.

Using a state machine for an object, that reacts to events differently based on
its current state, reduces the amount of boilerplate and duct-taping you have
to introduce to your code. 

StateMachine package provides a feature complete implementation of
finite-state machines in Go.

__What Is A Finite-State Machine Even?__

> A finite-state machine (FSM) or finite-state automaton (FSA, plural:
> automata), finite automaton, or simply a state machine, is a mathematical
> model of computation. It is an abstract machine that can be in exactly one
> of a finite number of states at any given time. The FSM can change from one
> state to another in response to some external inputs; the change from one
> state to another is called a transition. An FSM is defined by a list of its
> states, its initial state, and the conditions for each transition.
>
> — [Wikipedia](https://en.wikipedia.org/wiki/Finite-state_machine)

In other words, state machines are utilized for automation of entities that
have a state, an example of which is implemented across [usage](#usage) below.
In the examples, I'm simulating a state machine for an executable [process](https://github.com/Gurpartap/cognizant/blob/master/lib/cognizant/process.rb#L28-L87)
running in an operating system.

State machines are also useful for, although not limited to, long lived
workflows. For example, in a visa application at an embassy, the application
workflow can be implemented with a state machine which may assist with the
automation of state transitions of:

- manual checking (e.g. manually verifying submitted documents)
- user input (e.g. the visa application requiring additional user input)
- time interval (e.g. 24 hours before the application moves into next state)
- notifications (e.g. automated notification of state changes to the applicant)
- and so on.

In some cases, however, even if state machine appears to be a useful approach,
it might actually be completely unnecessary to introduce an external package.
For example, a user sign up flow does not necessarily need to be a state
machine. Perhaps just a sequence of operations like, `validateInput()`,
`migrateFromV1()`, `saveUser()`, and `notifyEmail()`. might be enough. Or even
a quickly tailored switch state based state machine might suffice.

### Further Reading

- [Finite-state Machine](https://en.wikipedia.org/wiki/Finite-state_machine) on Wikipedia
- [Finite State Machines Course Notes](http://www4.ncsu.edu/~drwrigh3/docs/courses/csc216/fsm-notes.pdf) by David R. Wright
- [Coding State Machines in C or C++](https://barrgroup.com/Embedded-Systems/How-To/Coding-State-Machines) by Miro Samek
- [state_machines Ruby Gem](https://github.com/state-machines/state_machines)
- [Flying Spaghetti Monster](https://en.wikipedia.org/wiki/Flying_Spaghetti_Monster)

> A complex system that works is invariably found to have evolved from a simple
> system that worked. A complex system designed from scratch never works and
> cannot be patched up to make it work. You have to start over with a working
> simple system. 
>
> – [John Gall (1975)](https://en.wikipedia.org/wiki/John_Gall_(author)#Gall.27s_law)

### Project Goals

Performance is a fairly significant factor when considering the use of
a third party package. However, an API that I can actually code and design in
my mind, ahead of using it, is just as important to me.

`statemachine`'s API design and developer productivity take precedence over
its benchmark numbers (especially when compared to a bare metal switch
statement based state machine implementation, which doesn't take you far).

For this, the package provides DSL-ish API using builder objects. These
builders compute and verify the developer defined specs. They then inject the
result (states, events, transitions, callbacks, etc) into the state machine
object during its initialization. Subsequently, these builders get released
from the memory. The state _machinery_ is not dependent on builders, however.

### Project Status

This readme and the included examples define the specifications and feasible
feature-set of the project. These act as a lookahead reference for the
project's development. Currently, therefore, what you see here is ahead of what
you'll be able to use in code.

## Installation

```bash
$ dep ensure -add https://github.com/Gurpartap/statemachine-go
```

```bash
$ go get -u https://github.com/Gurpartap/statemachine-go
```

## Usage

A state machine definition comprises of the following components:

- [States and Initial State](#states-and-initial-state)
- [Events](#events)
- [Transitions](#transitions)
- [Transition Guards](#transition-guards-conditions)
- [Transition Callbacks](#transition-callbacks)

Each of these components are covered below, along with their example usage
code.

Adding a state machine is as simple as embedding statemachine.Machine in
your struct, defining states and events, along with their transitions.

```go
type Process struct {
    statemachine.Machine

    // or

    Machine statemachine.Machine
}

func NewProcess() *Process {
    process := &Process{}

    process.Machine = statemachine.NewMachine()
    process.Machine.Build(func(m statemachine.MachineBuilder) {
        // ...
    })

    // or

    process.Machine = statemachine.BuildNewMachine(func(m statemachine.MachineBuilder) {
        // ...
    })

    return process
}
```

States, events, and transitions are defined using, what I call "builders",
which including `statemachine.MachineBuilder` and
`statemachine.EventBuilder`. These builders provide a clean DSL for writing
the specification of how the state machine functions.  

The subsequent examples are a close port of
[my experience](https://github.com/Gurpartap/cognizant/blob/master/lib/cognizant/process.rb#L28-L87)
with using the [state_machines](https://github.com/state-machines/state_machines)
Ruby gem.

### States and Initial State

Possible states in the state machine may be manually defined, along with its
initial state. States are, however, also inferred and collected from event
transition definitions.

Initial state is set during the initialization of the state machine, and is
required to be defined in the builder.

```go
process.Machine.Build(func(m statemachine.MachineBuilder) {
    m.States("unmonitored", "running", "stopped")
    m.States("starting", "stopping", "restarting")

    // Initial state must be defined.
    m.InitialState("unmonitored")
})
```

### Events

Events act as a virtual function which when fired, trigger a state transition.

```go
process.Machine.Build(func(m statemachine.MachineBuilder) {
    m.Event("monitor", ... )

    m.Event("start", ... )

    m.Event("stop", ... )

    m.Event("restart", ... )

    m.Event("unmonitor", ... )

    m.Event("tick", ... )
})
```

### Transitions

Transitions represent the change in state when an event is fired.

Note that `.From(states ...string)` accepts variadic args.

```go
process.Machine.Build(func(m statemachine.MachineBuilder) {
    m.Event("monitor", func(e statemachine.EventBuilder) {
        e.Transition().From("unmonitored").To("stopped")
    })

    m.Event("start", func(e statemachine.EventBuilder) {
        // Set multiple From states.
        e.Transition().From("unmonitored", "stopped").To("starting")
    })

    m.Event("stop", func(e statemachine.EventBuilder) {
        e.Transition().From("running").To("stopping")
    })

    m.Event("restart", func(e statemachine.EventBuilder) {
        e.Transition().From("running", "stopped").To("restarting")
    })

    m.Event("unmonitor", func(e statemachine.EventBuilder) {
        e.Transition().FromAny().To("unmonitored")
    })

    m.Event("tick", func(e statemachine.EventBuilder) {
        // ...
    })
})
```

### Transition Guards (Conditions)

Transition Guards are conditional callbacks which expect a boolean return
value, implying whether or not the transition in context should occur.

```go
type TransitionGuardFnBuilder interface {
    If(guardFn TransitionGuardFunc)
    Unless(guardFn TransitionGuardFunc)
}
```

Valid TransitionGuardFunc signatures:

```go
*bool
func() bool
func(transition statemachine.Transition) bool
```

```go
// Assuming process.IsProcessRunning is a bool variable, and
// process.GetIsProcessRunning is a func returning a bool value.
m.Event("tick", func(e statemachine.EventBuilder) {
    // If guard
    e.Transition().From("starting").To("running").If(&process.IsProcessRunning)

    // Unless guard
    e.Transition().From("starting").To("stopped").Unless(process.GetIsProcessRunning)

    // ...

    e.Transition().From("stopped").To("starting").If(func(t statemachine.Transition) bool {
        return process.ShouldAutoStart && !process.GetIsProcessRunning()
    })
    
    // or

    e.Transition().From("stopped").To("starting").
        If(&process.ShouldAutoStart).
        AndUnless(&process.IsProcessRunning)

    // ...
})
```

### Transition Callbacks

Transition Callback methods are called before, around, or after a transition.
The following 3 transition callbacks are available:

- [Before Transition](#before-transition)
- [Around Transition](#around-transition)
- [After Transition](#after-transition)

#### Before Transition

Before transition callbacks do not act as a conditional, and a bool return
value will not impact the transition. 

Valid TransitionCallbackFunc signatures:

```go
func()
func(t statemachine.Transition)
```

```go
process.Machine.Build(func(m statemachine.MachineBuilder) {
    // ...

    m.BeforeTransition().FromAny().To("stopping").Do(func() { 
        process.ShouldAutoStart = false
    })

    // ...
}
```

#### Around Transition

Around transition's callback provides a method signature as input, which must
be called inside the callback. Missing to call the method will trigger a
runtime failure with an appropriately describing error. 

Valid TransitionCallbackFunc signatures:

```go
func(next func())
func(t statemachine.Transition, next func())
```

```go
process.Machine.Build(func(m statemachine.MachineBuilder) {
    // ...

    m.
        AroundTransition().
        From("starting", "restarting").
        To("running").
        Do(func(next func()) {
            start := time.Now()

            // It'll trigger a failure if next is not called.
            next()

            elapsed = time.Since(start)

            log.Printf("It took %s to [re]start the process.\n", elapsed)
        })

    // ...
})
```

#### After Transition

After transition callback is called when the state has successfully
transitioned.

Valid TransitionCallbackFunc signatures:

```go
func()
func(t statemachine.Transition)
```

```go
process.Machine.Build(func(m statemachine.MachineBuilder) {
    // ...

    // Notify system admin.
    m.AfterTransition().From("running").ToAny().Do(process.DialHome)

    // Log all transitions.
    m.
        AfterTransition().
        FromAny().
        ToAny().
        Do(func(t statemachine.Transition) {
            log.Printf("State changed from '%s' to '%s'.\n", t.From(), t.To())
        })

    // ...
})
```

### Event Callbacks

Event Callback methods are called after an fails to transition the state. The
following transition callback is available:

- [After Failure](#after-failure)

#### After Failure

After failure callback is called when there's an error with event firing.

Valid TransitionCallbackFunc signatures:

```go
func()
func(err error)
func(t statemachine.Transition, err error)
```

```go
process.Machine.Build(func(m statemachine.MachineBuilder) {
    // ...

    m.AfterFailure().OnAnyEvent().
        Do(func(t statemachine.Transition, err error) {
            log.Printf("Error occurred when transitioning from '%s' to '%s':\n", t.From(), t.To())
            log.Println(err)
        })

    // ...
})
```

### Matchers

#### Event Transition Matchers

These may map from one or more `from` states to exactly one `to` state.

```go
type TransitionBuilder interface {
    From(states ...string) TransitionFromBuilder
    FromAny() TransitionFromBuilder
    FromAnyExcept(states ...string) TransitionFromBuilder
}

type TransitionFromBuilder interface {
    ExceptFrom(states ...string) TransitionExceptFromBuilder
    To(state string) TransitionToBuilder
}

type TransitionExceptFromBuilder interface {
    To(state string) TransitionToBuilder
}
```

__Examples:__

```go
e.Transition().From("first_gear").To("second_gear")

e.Transition().From("first_gear", "second_gear", "third_gear").To("stalled")

allGears := vehicle.GetAllGearStates()
e.Transition().From(allGears...).ExceptFrom("neutral_gear").To("stalled")

e.Transition().FromAny().To("stalled")

e.Transition().FromAnyExcept("neutral_gear").To("stalled")
```

#### Transition Callback Matchers

These may map from one or more `from` states to one or more `to` states. 

```go
type TransitionCallbackBuilder interface {
    From(states ...string) TransitionCallbackFromBuilder
    FromAny() TransitionCallbackFromBuilder
    FromAnyExcept(states ...string) TransitionCallbackFromBuilder
}

type TransitionCallbackFromBuilder interface {
    ExceptFrom(states ...string) TransitionCallbackExceptFromBuilder
    To(states ...string) TransitionCallbackToBuilder
    ToSame() TransitionCallbackToBuilder
    ToAny() TransitionCallbackToBuilder
    ToAnyExcept(states ...string) TransitionCallbackToBuilder
}

type TransitionCallbackExceptFromBuilder interface {
    To(states ...string) TransitionCallbackToBuilder
    ToSame() TransitionCallbackToBuilder
    ToAny() TransitionCallbackToBuilder
    ToAnyExcept(states ...string) TransitionCallbackToBuilder
}
```

__Examples:__

```go
m.BeforeTransition().From("idle").ToAny().Do(someFunc)

m.AroundTransition().From("state_x").ToAnyExcept("state_y").Do(someFunc)

m.AfterTransition().FromAny().ToAny().Do(someFunc)
```

#### Event Callback Matchers

These may match on one or more `events`. 

```go
type EventCallbackBuilder interface {
	On(events ...string) EventCallbackOnBuilder
	OnAnyEvent() EventCallbackOnBuilder
	OnAnyEventExcept(events ...string) EventCallbackOnBuilder
}

type EventCallbackOnBuilder interface {
	Do(callbackFunc EventCallbackFunc) EventCallbackOnBuilder
}
```

__Examples:__

```go
m.AfterFailure().OnAnyEventExcept("event_z").Do(someFunc)
```

#### Callback Functions

Any callback function's arguments (and return types) are dynamically set based
on what types are defined (dependency injection), and therefore, unnecessary
variables may be skipped. In other words, all input parameters are optional.

For example, if your BeforeTransition() callback does not need access to the
`statemachine.Transition` variable, you may just define the callback with a
blank function signature: `func()`, instead of
`func(t statemachine.Transition)`. Similarly, for an AfterFailure()
callback you can use `func(err error)`, or
`func(e statemachine.Event, err error)`, or even just `func()` . 

## About

    Copyright 2017 Gurpartap Singh

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

        http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.

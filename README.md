# Pipeline
Go pipeline processing, connecting sources to sinks with operators (processing calculi).

[Process Calculi](https://en.wikipedia.org/wiki/Process_calculus) are, in this context, essentially functions that act on data from an input channel and return results to an output channel. This all fits nicely with Go's concurrency model, with is base on [CSP](https://en.wikipedia.org/wiki/Communicating_sequential_processes) (a common example of process calculi).

## Design:

This repository split up based on which part of the pipeline it drives, with a few helper packages:
- `pipeline`
  - plumbing
  - source
  - sink
- `forwarder`
- `contrib`
  - `sources`
  - `sinks`
  - `calculi`

Influences:
- (Automi)[https://github.com/vladimirvivien/automi)
  - Initial idea spark, API components. However, I wanted more generic operations. Map/Filter/etc should be operator implementations.
- Actor model, and then process calculi (which was a more apt model)

Sources provide arbitrary event streams expressed as strings, Sinks consume events and do something with them, while Plumbing takes a Source and connects it to a (or multiple) Sink. The main package will be a CLI that uses flags to setup a Source and any number of Sinks, however most work will be done via Plumbing functions to easily allow alternative UIs.

A generic helper Forwarder package allows an input channel to multiplex to registered output chans. Additionally, could look into code generation to automatically create Forwarder methods on arbitrary types.

Source Design:

Create Source interface with "Init func (control ->chan error, output ->chan string) error". Init implementations are expected to start a goroutine that populates output with source data, sends event error messages over control (and halt if closed), and return an error if initialization fails. Output is not buffered, not guaranteed to have a receiver, or otherwise may have a slow receiver. Source should be able to handle all cases by dropping messages, manual buffering, or other means. That said, depending on the Forwarder implementation, buffering may be automatically handled, allowing unrestricted channel writing.

Planned Implementations:
- Docker Events: listens to events from the docker events API
- STDIN

Sink Design:

Create Sink interface with "Init func (control ->chan error, input <-chan string) error". Sinks are expected to start a goroutine to consume events from input, send event processing errors over control (error if halt of closed), and return an error if initialization fails.

Planned Implementations:
- Desktop notifications. OS X notifier can use github.com/everdev/mack.Notify.
- STDOUT

Plumbing Design:

Functions acting on Sources and Sinks, setting up communication channels, initializing forwarders, etc.

Forwarder Design:

Forwarder package with New function returning a struct with InputChan method (func (f *Forwarder) InputChan() ->chan string), Register method (func (f *Forwarder) Register (output string) outputChan <-chan string, err error), and Unregister method (func (f *Forwarder) Unregister (output string) error).. InputChan returns the channel the Forwarder listens on, Register adds an output to forward to and returns the channel it will send on, while Unregister removes an output and closes its channel. Outputs can be un/registered at any time and will be reflected on the next message to InputChan. A Buffered Forwarder is expected to not lose messages sent on InputChan, even if any of its outputs are slow, not read from, etc. This may be done by having goroutines per output with a buffer for unsent messages, and selecting sends (from buffer) and writes (from input to buffer).

The planned design may introduce a lot of garbage because each output gets a copy of each input message, and slow outputs may incur largely duplicate buffers. Since they're currently strings, at least they'll have the same backing array, though that may not be ideal if outputs mutate it. To measure this, it would be good to have memory usage benchmarks and use profiling to minimize the overhead. Providing a configurable try timeout per output or globally that would unregister after no successful send would lessen the issue. Could also provide N queued items before dropping. That could be implemented a few ways, most likely dropping oldest when adding new.

Actually: Since strings are immutable, we don't have to copy them before sending on channels, can just send. But, when/if updating to support interfaces or structs or whatever, we have to do a copy, somehow...

Random Notes:

Must use 'main' packages to connect the pieces, ie set input, etc. Sinks, sources, etc could be made declaratively in json, but still have to import in a main. To aid that declarative end, could use `go generate` to take chain config and create a main package. Another interesting possibility would be to use something like DOT as the input, and parse a DOT file and create stubs for each node with `go generate`. Of course, should also be able to output a DOT file from a programmatically created Stream. Could also provide a lookup file that contains function bodies for nodes, so can create entire `main` from a DOT file and function sources file. Could have a DOT parser that works like https://github.com/golang/tools/blob/master/cmd/stringer/stringer.go

Would like to see a chain-able API such that `Stream.Source(Sourcer).Do(Operation).Do(Operation).Fork(Operation, Operation)`. A fork requires a split chain of course because it would return a list of chainables (in the order they were added). After all that, a `.Sink(Sinker)` (for each fork). Essentially a Sourcer is an interface that doesn't have an input chan (but may have context, etc, tbd later), but returns an output chan. An Operation is an interface that takes an input chan and returns an output chan. Finally, a Sinker is an interface that only takes in input chan. Note that Fork is not equivalent to splitting a job to multiple Operators, it is literally Forking the execution chain for different outputs, not to be reduced together. See the next point for more context.

I think it should be easy to enable multiple copies of an Operation, maybe via an ActN function that takes an Operation and how many to run. They would all accept the same input channel and output channels and just do work in parallel. This is similar to splitting up a task to multiple Maps and then Reducing in MapReduce, but avoids an explicit Reduce step (handled by writing to same channel).

Not sure how to structure the chain. I'm thinking .Source/Act/Sink will return a mutated Stream, however that gets a bit messy when Forking. A Fork could be implemented by copying and returning N Streams that can be Acted on independently, but then you lose an easy way to see the entire Stream from the top down, unless the original stores references to it's Forked Streams.

Since can't have arbitrarily nested maps or structs to represent it, could use linked list. Operations can be a recursive data type like a linked list. The difference will be that Operations will have references to previous ones (instead of next) to look up it's output 
channel, but also allow traversal up (which has limitations with forks).

Would like to allow inspection into the pipeline, so that you can see the chain, which the previous point enables. The Stream type could implement fmt.Stringer to pretty print it by default.

Allow the Forwarder to be buffered or not (not buffered chan, but list buffer). Default to buffered for simplicity, but allow setting output chan to be buffered. The semantics for where to buffer depends on the pipeline, but since it's a downstream workflow, it must be set by the outputting Operator, not the inputting Operation.

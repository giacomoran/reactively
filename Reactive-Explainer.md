Reactive programming allows you to define program segments called reactive elements that run automatically as needed. The main goal of a reactive API is to make it easy to define *reactive elements* and their *sources*. 

* Reactive elements rely on other reactive elements or external data as sources.
* Reactive elements run automatically when their sources change. 
* The reactive system orchestrates the execution of the reactive elements for efficiency and consistency.

Reactively uses a single annotation `@reactive` to identify reactive elements. 
And in Reactively, sources for reactive elements are detected automatically. 


**Reactively Priorities**
1) Simplicity in the developer experience (lightweight, easy to add, small api, clean semantics)
2) minimal re-execution of user code 
3) performance

Other frameworks have different priorities, in different orders.
That's just fine, we're exploring a different part of the design space.

For example, Reactively tries hard to not re-execute user code unnecessarily.
That's not always necessary. Any given segment of user code is probably cheap to run. 
But Reactively is willing to trade off some time thinking about what not to re-execute
so the user can put expensive computations/allocations inside reactive elements 
without extra protection.

```jsx
/* A lightweight reactive program can look almost like regular javascript programming. 
 *
 * This example uses a class, with a few @reactive annotations added.
 * (Reactively can be used without classes too)
 *
 * Here's the dataflow relationship between the reactive parts: 
 *    size -> bufferBlocks -> buffer
 */
class ResizeableBuffer {
	@reactive size = 0;

	@reactive blocks = () => Math.ceil(this.size / 2**16);

	@reactive buffer = () => {
		this._buf = new Buffer(this.blocks()).copyFrom(this._buf); 
    return this._buf;
  }
}

b = new ResizeableBuffer()
b.size = 10**5;

/* Here we call buffer() twice. In an imperative system we'd allocate twice
 * which is inefficient. A reactive system will allocate only once.  */
b.buffer().fill(-1); 
b.buffer().setAt(0,100); 

/* A reactive system can find other efficiencies. Here's one example: */

b.size += 1; // grow the number of elements, but blocks doesn't change
b.buffer();  // hopefully no new buffer allocated here. See 'equality propogation'
```

**Reactive system terminology** 

*reactive, computed, signals, effects, state*

In lighter weight reactive systems, reactive elements are sometimes called ‘computed’ values, ‘effects’, ‘memo’, ‘reactions’, or ‘reactive’ values. Some systems distinguish reactions from sources that are set from inputs external to the reactive system. Reactive sources are variously called ‘signals’ or ‘events’ or ‘state’. 

In Reactively, sources and reactions are simply labeled @reactive.

*streams* 

The flow of data through the reactive graph can also be encapsulated in asynchronous streams. Streams enable abstraction over patterns of data flow over time: debouncing, grouping, etc. Streams are rich but complicated abstractions. Most front end frameworks have chosen lighter weight approaches to reactivity, as we have here. 

*fine grain vs coarse grain*

A reactive library tracks the sources and a reactive elements and smartly re-executes only when needed.
But how big is a a reactive element? 
Reactively is fine grained - it applies to individual functions, and variables or properties. 
Some reactive libraries re-execute at a coarser level, by component. (e.g. lit or React)

**Execution model**

The set of reactive elements and their source dependencies forms a directed (and usually acyclic) graph. Conceptually, reactivity changes enter at the roots of the graph and propagate to the leaves. 

But that doesn’t mean the reactive system needs to respond to a change at the root by immediately deeply traversing the tree and re-executing all of the related reactive elements. What if the user has made changes to two roots? We might unnecessarily execute elements twice, which is inefficient. What if the user doesn’t consume a reactive leaf element before a new change comes around? Why bother executing that element if it’s not used? If two changes are in flight through the execution system at the same time, might user code see an unexpected mixed state (this is called a ‘glitch’)?

These are the questions the reactive execution system needs to address. 

Push systems emphasize pushing changes from the roots down to the leaves. Push systems use batches to combine the multiple changes into one execution pass to avoid glitches. Batching is user visible though, exposing issues of consistency between batches and batch control which adds complexity to the developer experience. 

Pull systems emphasize traversing the graph in reverse order, from user consumed reactive elements up towards roots. Pull systems have a simple developer experience and don’t require batching. But pull systems are are apt to traverse the tree too often. Each leaf element needs to traverse all the way up the tree to detect changes, potentially resulting in many extra traversals.

Reactively is a hybrid push pull system. It pushes dirty notifications down the graph, and then executes reactive elements lazily on demand as they are pulled from the lowest level. This costs the framework an extra traversal of its internal graph. But the developer wins by getting the simplicity of a pull system and the execution efficiency of a push system.
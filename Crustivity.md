# General Idea
Crustivity does not have the notion of a nested tree of widgets/components, where each widget contains the state it needs to be drawn and interacted with. Instead **ALL** state is defined by the user and state-management primitives provided by Crustivity are used to manage resources and actions like graphics drawing, audio, or networking. 

However Crustivity still uses a tree to describe its layout, but this tree is not polymorphic in the sense that all nodes in this tree are of the same (non generic) type. This makes layout easy to deal with in Rust, since neither state nor layout needs to deal with generics or trait objects. This tree is created/managed by the state-primitives and the positions and sizes of the nodes, calculated using this tree, are use during drawing the ui and user-interaction.

Since state does not need to be nested for layout purposes, most state is represented by a much more flatter structure compared to other ui-Toolkits.

# Components
In principal Components work similar as in Solid-Js, they use state and [[effect]]s to create a reactive graph that drives layout among other things.
![[Reactive_graph_layout_tree.excalidraw|400]]

The Component itself is made of all persistent state, and a function that that sets up [[effect]]s and [[constraints]] and returns the layout node. But a library that provide reusable components actually does not create its own state-`struct` but instead uses a `trait` that is implemented by the consumer. This implementation is mostly automated using [macros](Widget%20Macro). This allows projects to use one single struct for that Component (which may or may not be provided by the library) or multiple different structs based on the projects needs.

# State
All state is (can be) defined by the application author. This state lives in a structure complete defined by the user. Some values can be wrapped in a Variable, also often named Signal, that lets [[Task]]s know when the value changed. Behind the scenes the values of Variables are put into arch tables (think ECS without entities), to break up aliasing.

# Task
A Task is a function and an [[Extractor]] that extracts values from the application state and give that to the function as argument.

Tasks can be chained together to form lager tasks that are executed in order and can pass values from previous to next sub task.

The [[Extractor]] can fail, which cancels the entire task-chain. [[Reactive Design]] has more details.

## Executing a Task
First the [[Extractor]] is executed, then the extracted Variables with exclusive access are acquired. Then the values are passed as argument to the function of the Task.
If the task has no function, then the process stops before exclusive access should be acquired and the Tasks executed successfully.
If another Task acquired the same value after the extraction stared and before the function is started, the entire extraction is stared anew.

# Comparisons
## Solid-Js
This project is very much inspired by Solid-Js, but also makes a bunch of changes (hopefully improvements) to it, largely possible because the web platform is not first priority.
1. Cyclic reactive graph. Solid-Js' reactive graph is acyclic. This makes is much easier to reason about but also rejecting certain patterns. Crustivity can be cyclic but does place some restrictions on cycles to help manage that.
2. Implicit/Explicit effect dependencies. Crustivity used explicit dependency declarations. This is needed for the [[Constraint System]] as well as multi-threading. It may be possible to elide explicit dependencies for simple situations.
3. Multi-threading. All effect/constraints that can be run in parallel, will, making full use out of modern hardware.

## [[Xilem]]
[[Xilem]]s ui is composed out of two trees, a view tree and a element tree. The view tree is used as a description on what the ui should be and [[Xilem]] updates the element tree to reflect that. Crustivity does not have a view tree but uses a reactive graph to drive a underlying tree. Another difference is that the element tree contains all state needed for the ui where as Crustivities layout tree only contains state necessary for layout. 
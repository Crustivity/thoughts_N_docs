For declarative user interfaces the following equation is true:
$$ v = f(s) $$
- $v$: the user interface
- $s$: the state of the application
- $f$: the function (*code*) that creates the view from the function

A direct implementation of this relation reruns the function each time the state changes and displays the new view. Technicalities such as vdom-diffing is _just_ an optimization.

But let's zoom a little into this equation. 

Consider a user interface where pressing a button increments a counter that is displayed on the screen. On part of the state $s$ is the number of the current counter, but user interfaces often have to deal with the same data in multiple representations that needs to stay synchronized. In our counter example the in addition to the number we also need a string that contains the number in decimal encoding. We also need the individual font-glyphs which are then rasterized into a pix-map to be displayed. Not all of these representations need to be retained in memory on all times but doing so can lead to better performance.

Solid-Js and React approach this by splitting the state into main-state and derived-state. Changes can only occur on the main-state, then the derived values are calculated and all state together is the used as state for $f$. 
$$ v = f(s + s') $$
- $s'$: derived state from $s$. It may be memorized.

Crustivity does thing a bit different. It does not have a direct notion of derived state. Instead the programmer constrains multiple variables to achieve the desired synchronization. A [Constraint] uses one or more functions that take some of the variables as input and some as output to constrain all used variables together. Modifying one variable breaks the Constrain and it can execute any of the provided functions once restore the relation. Having a Constraint with only one function does essential the same thing as a memorized derived value. Non-memorized derived values are a little bit more challenging, also because in Rust every value needs to live somewhere and I cannot just put it in the aether. 

Now lets zoom in on $f$.

$f$ needs to be evaluated every time the state changes. But state changes are not random. They happen after events and usually not all state is affected. If we use that knowledge we can reuse existing parts of the view $v$ and only update parts that changed. 
$$ v' = f_v(s + s', u) $$
- $u$: the update that initiated the change
- $f_v$: the function that, using $v$ as previous view, calculated the new view using the total state and the update that initiated the change.

Solid-Js does updates only the view, that is indirectly related to the state, whose change initiated the update. Crustivity does the same in a similar fashion.

React does not do this. Instead it creates the new view and compares it the the previous view to calculate the changes needed to only update the parts of the user interface that actually changed. This does not mean that this is a less performant approach. Diffing can be fast (see [Blockdom]) and tracking change all the way down can lead lead to less ergonomic code. However it also makes it harder to create fundamental elements. In React you cannot really do this at all. You can use Web-Components as a new fundamental element but then there is a hard layer between React world and Web-Component world (maybe it is possible to bridge them but I do not know how). In other Frameworks that employ diffing (Xilem, Flutter) you can do it, but it requires a substantial amount of code and deep knowledge of how the framework works under the hood to make it fast. This trade-off makes sense if you can reuse fundamental elements very often, as the work has to be done only once.

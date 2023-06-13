# Application State
Consider this example
```rust
struct Address {
    code: Variable<u32>,
    street: Variable<String>,
}

struct User {
    name: Variable<String>,
    address: Variable<Address>
}
```
A `Variable` is just a generational index into its respective arch-table. 
```rust
struct World {
    /// synchronization needed
    tables: HashMap<TypeId, Any>
}

impl<T> Variable<T> {
  fn get(self, world: &World) -> &T;

  /// synchronization needed
  fn get_mut(self, world: &World) -> &mut T;
}
```

# Task

# Constraints
The idea was first developed in [[HotDrink]] allowing programmers to create complex relations between mutable variables that have to stay in sync over time.
Consider a the following equation:
$$ s = a + b $$
It involves three variables $s$, $a$, and $b$. And a constraint enforcing that $s$ has to be always the sum of $a$ and $b$. The goal is now to upkeep that relation even if **ANY** of the variable changes. For the variables $a$ and $b$ this is relatively easy, if one of them changes just re-evaluate the sum, but if $s$ changes it get way more complicated. Most frameworks do not want to deal with the complexity and just disallow it altogether. 

How does [[HotDrink]] do it?
What we need is to rewrite the equation so that it can solve for another variable making $s$ the input. And since that is in general not possible to do automatically the programmer has to provide these different rewrites manually. 
$$ a = s - b $$
$$ b = s - a $$
With these, once $s$ changes, we can select one equation and evaluate it making sure that the constraint is satisfied. 
A Constraint for $n$ variables is a set of equations ([[HotDrink]] calls them methods, I will call them functions since they have nothing to do with methods in an OOP sense), where each function takes a subset as input and the rest as output, and calculates the values for the outputs from the inputs. The relation the Constraint enforces is then implicitly defined by the functions and not some explicitly written thing.

# Extractors
Why do tasks need extractor in the first place? Two reasons:
1. Supporting nested variables
2. preventing deadlock for multi-threading

## Nested Variable
Now it gets complicated. This sections uses a Variable of a `Vec` with items that are Variables themselves as an example:
```rust
Varable<Vec<Variable<i32>>>
```
Now we look at different scenarios and try to solve them:

### Using the first Variable in a Constraint
A use-case could be to treat the first entry as a header of a list and to display is differently. 
Accessing the first element of the `Vec` and using it in a Constraint does work, but if we append a new Variable to the front of the `Vec` then we would not be using the first element of the `Vec` anymore. We could just be careful to not do that, but if we want to do that we have to do things differently.
In some sense the Variable `Vec` is part of the Constraint, but it is not part of the set of inputs and outputs. But if the `Vec` changes we need to be ready to change the Constraint itself. 
But what if the `Vec` is cleared? Then the Constraint cannot be changed to use a missing Variable. There are now two choices, either destroy the Constraint, or merely disable it, and reactivate it once the `Vec` as elements again.

### Using all Variables in the `Vec` as input
Computing the average of all numbers in a `Vec` where everything is changeable is pretty common. In this case you need both the `Vec` and the numbers itself as input. But if you have access to the `Vec` you also have access to everything contained in the `Vec`. For our example this is not much of an issue, but once mutability is in play we have to be careful, not to violate aliasing. Because of this the value of Variables are not stored inline, instead the Variable is _just_ an index into an array. That means access to the `Vec` allows you to access its field, such as `capacity` and `len`, you can even get access to Variables but you cannot use them to load the actual values that the index refers to.
If an item changes then the Constraint is violated and a function can be called with the `Vec` and all items as input. But if the `Vec` itself changes, for example by pushen a new Variable to it, then the Constrain needs to be updated taking the new Variable into account.

### Mutable access to an element and `Vec`
Since access to the `Vec` does not give access to the values of the element both can have mutable access at the same time. And because the items do not live technically in the `Vec` directly, it could be cleared while another thread is still using the elements. Dropping of the items would happen later, once they are not used anymore.


## Deadlock prevention
Tasks are scheduled and then acquire locks for the variables they want to access. This can easily lead to deadlocks and Crustivity is designed to prevent that from happening by restricting when exactly Variables are acquired and released. 
First things first it should be noted that with purely shared access deadlocks cannot happen, since there is never a place where a task needs to wait. But once exclusiv access is required we have to exercise some restrictions. Such a task cannot acquire access to additional Variables once it has started. 
In practice all tasks cannot do that. That right only belongs to extractors and those **MUST** use purely shared access to values. When an extractor needs to wait for a value, then that must mean the value is used inside a task and once a task started it cannot wait for additional values anymore. Don't put infinite loops in tasks, they will stall out the rest of the system, like it does in any other ui-system. A task that does not need exclusive access to values can do its work entirely in its extractor (which in reality are most [[Effect]]s), making it easier to write them.
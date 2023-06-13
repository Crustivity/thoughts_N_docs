# General Idea
Reusable components need state to function. This state is accessed though a trait instead of depending on a struct directly. These traits are pretty simple in themselves and only have object-save methods that take an `&self` and return a `Variable`
> [!question] Needs Thinking
> These traits should support variance.

However these traits are annotated by macros that provide more information to the facilitate customizable and automatic implementation of these traits.

## Variance
Variables that are written to need to be invariant. This is pretty straight-forward, but I like to support read only variables that are more permissive. A trait that specifies a read only Variable holding a `&str` should be implementable with a Variable that holds a `String` or any other type that de-references to `&str`. Constraints that use such read only Variables cannot write to them, which limits their use, but it should not me much of a restriction for effects.
Implementation wise a read only Variable refers **NOT** to the arch-table of the generic parameter but instead to some other arch-table. This requires the read only Variable to store the additional `TypeId` as a field. This runtime cost cannot be avoided, since two `ReadVar<str>` can have their value stored in different arch-tables (`Variable<String>`, `Variable<Box<str>>`, `Variable<Rc<str>>`) which are undecidable from the `ReadVar<str>` alone.

```rust
#[widget]
trait CheckBox {
    #[read]
    fn name(&self) -> str,
    fn state(&self) -> bool,
}

#[widget(CheckBoxWidget)]
struct MyButton {
    name: String,
    state: bool,
}

```

This roughly generates:
```rust
trait CheckBox {
   fn name(&self) -> ReadVar<str>,
   fn state(&self) -> Variable<bool>,
}

macro_rules! CheckBoxWidget {
    ($tt:tt) => {
        #[widget(/* custom dsl to provide implementation info */)]
        $tt
    }
}

struct MyButton {
    name: Variable<String>,
    state: Variable<bool>,
    /* layout info, not sure how that exctly that looks like */
    _widget: Widget,
}

impl CheckBox for MyButton {
   fn name(&self) -> ReadVar<str> {
      self.name.read_var()
   }

   fn state(&self) -> Variable<bool> {
      self.state
   }
}

impl Widget for MyButton {
   fn widget(&self) -> Widget {
      self._widget
   }
}

/* a builder is generated */
```

Usage of this stuff:
```rust
fn 
```

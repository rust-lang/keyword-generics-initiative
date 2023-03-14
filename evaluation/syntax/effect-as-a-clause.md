- Name: `effect-as-a-clause`
- Proposed by: [@mominul](https://github.com/mominul) [@satvikpendem](https://github.com/satvikpendem)
- Original proposal (optional): (https://github.com/rust-lang/keyword-generics-initiative/issues/14)

# Design
We want to propose the usage of the `effect` clause to achieve operation genericity, for example:
```rust
trait Read {
    fn read(&mut self, buf: &mut [u8]) -> Result<usize>
    effect
        async;

    fn read_to_string(&mut self, buf: &mut String) -> Result<usize> 
    effect
        async
    { .. }
}

/// Function to read from the file into a string which may exhibit async or const effect
fn read_to_string(path: &str) -> io::Result<String>
effect
       async, const 
{
    let mut string = String::new();

    // We can be conditional over the context the function has been called from, 
    // only when the function declaration has the `effect` clause
    if async || !async {
        let mut file = File::open("foo.txt")?; // File implements Read
        // Because `read_to_string` is also an `effect` function that may or may not exhibit 
        // async-ness par the declaration, we can use it on both contexts (async/sync) 
        // we are placing the condition on.
        file.read_to_string(&mut string)?;  // .await will be inferred.   
    } else { // must be const
        // As the `read_to_string` doesn't exhibit const-ness, we'll need to handle it ourselves.
        string = include_str!(path).to_string();
    }

    Ok(string)
}

/// A normal function
fn read() {
    let data = read_to_string("hello.txt").unwrap();
}

/// A async function
async fn read() {
    let data = read_to_string("hello.txt").await.unwrap();
}

/// A const function
const fn read() {
    let data = read_to_string("hello.txt").unwrap();
}
```
So in a nutshell, a function declaration with an `effect` clause is a special type of function that **may** or **may not** exhibit async or const behavior(effect) and it **depends on the context of the function being called from** and we can **execute a different piece of code according to the context** from the function was called from too (like the `const_eval_select`, resolves [#6](https://github.com/rust-lang/keyword-generics-initiative/issues/6)):

```rust
fn function() -> Result<()>
effect
    async, const
{
    // ...
    if async {
        // code for handling stuff asynchronously
    } else if const {
        // code for handling stuff `const`-way
    else {
        // code for handling stuff synchronously
    }
    // ...
}
```
## base (reference)

<!-- This is the snippet which is being translated to various scenarios we're
translating from. Please keep this as-is, so we can reference it later.-->

```rust
/// A trimmed-down version of the `std::Iterator` trait.
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    fn size_hint(&self) -> (usize, Option<usize>);
}

/// An adaptation of `Iterator::find` to a free-function
pub fn find<I, T, P>(iter: &mut I, predicate: P) -> Option<T>
where
    I: Iterator<Item = T> + Sized,
    P: FnMut(&T) -> bool;
```

## always async

<!-- A variant where all items are always `async` -->

```rust
pub trait Iterator {
    type Item;
    async fn next(&mut self) -> Option<Self::Item>;
    fn size_hint(&self) -> (usize, Option<usize>);
}

pub async fn find<I, T, P>(iter: &mut I, predicate: P) -> Option<T>
where
    I: Iterator<Item = T> + Sized,
    P: async FnMut(&T) -> bool;
```

## maybe async

<!-- A variant where all items are generic over `async` -->

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>
    effect async;
    fn size_hint(&self) -> (usize, Option<usize>);
}

pub fn find<I, T, P>(iter: &mut I, predicate: P) -> Option<T>
where
    I: Iterator<Item = T> + Sized,
    P: FnMut(&T) -> bool effect async;
effect
    async
```

## generic over all modifier keywords

<!-- A variant where all items are generic over all modifier keywords (e.g.
`async`, `const`, `gen`, etc.) -->

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>
    effect async, const;
    fn size_hint(&self) -> (usize, Option<usize>);
}

pub fn find<I, T, P>(iter: &mut I, predicate: P) -> Option<T>
where
    I: Iterator<Item = T> + Sized,
    P: FnMut(&T) -> bool effect async, const;
effect
    async, const
```

# Notes

<!-- Add additional notes, context, and thoughts you want to share about your design
here -->
We can introduce `maybe` keyword instead of `effect` if it seems more appropriate terminology for the semantics described in this proposal.
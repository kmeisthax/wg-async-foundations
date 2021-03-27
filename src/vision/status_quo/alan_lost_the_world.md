# 😱 Status quo stories: Alan lost the world!

## 🚧 Warning: Draft status 🚧

This is a draft "status quo" story submitted as part of the brainstorming period. It is derived from real-life experiences of actual Rust users and is meant to reflect some of the challenges that Async Rust programmers face today. 

If you would like to expand on this story, or adjust the answers to the FAQ, feel free to open a PR making edits (but keep in mind that, as they reflect peoples' experiences, status quo stories [cannot be wrong], only inaccurate). Alternatively, you may wish to [add your own status quo story][htvsq]!

## The story

Alan heard about a project to reimplement a deprecated browser plugin using Rust and WASM. This old technology had the ability to load resources over HTTP; so it makes sense to try and implement that functionality using the Fetch API. Alan looks up the documentation of `web_sys` and realizes they need to take a thing called a `Future` from the call to `fetch`, `await` it, and then do something else with the data.

```rust
use web_sys::{Request, window};

fn make_request(src: &url) -> Request {
    // Pretend this contains all of the complicated code necessary to
    // initialize a Fetch API request from Rust
}

async fn load_image(src: String) {
    let request = make_request(&url);
    window().unwrap().fetch_with_request(&request).await;
    log::error!("It worked");
}
```

Alan adds calls to `load_image` where appropriate, sees the message pop up in the console, and figures it's time to now actually do something to that loaded image.

At this point, Alan wants to put the downloaded image onto the screen, which in this project means putting it into a `Node` of the current `World`. A `World` is a bundle of global state that's passed around as things are loaded, rendered, and scripts are executed. It looks like this:

```rust

/// All of the player's global state.
pub struct World<'a> {
    /// A list of all display Nodes.
    nodes: &'a mut Vec<Node>,

    /// The last known mouse position.
    mouse_pos &'a mut (u16, u16),

    // ...
}
```

In synchronous code, this was perfectly fine. Alan figures it'll be fine in async code, too. So Alan adds the world as a function parameter and everything else needed to parse an image and add it to our list of nodes:

```rust
async fn load_image(src: String, inside_of: usize, world: &mut World<'_>) {
    let request = make_request(&url);
    let data = window().unwrap().fetch_with_request(&request).await.unwrap().etc.etc.etc;
    let image = parse_png(data, context);

    let new_node_index = world.nodes.len();
    if let Some(parent) = world.nodes.get(inside_of) {
        parent.set_child(new_node_index);
    }
    world.nodes.push(image.into());
}
```

Bang! Suddently, the project stops compiling, giving errors like...

```
error[E0597]: `world` does not live long enough
  --> src/motionscript/globals/loader.rs:21:43
```

Hmm, okay, that's kind of odd. We can pass a `World` to a regular function just fine - why do we have a problem here? Alan glances over at `loader.rs`...

```rust
fn attach_image_from_net(world: &mut World<'_>, args: &[Value]) -> Result<Value, Error> {
    let this = args.get(0).coerce_to_object()?;
    let url = args.get(1).coerce_to_string()?;

    spawn_local(load_image(url, this.as_node().ok_or("Not a node!")?, world))
}
```

Hmm, the error is in that last line. `spawn_local` is a thing Alan had to put into everything that called `load_image`, otherwise his async code never actually did anything. But why is this a problem? Alan can borrow a `World`, or anything else for that matter, inside of async code; and it should get it's own lifetime like everything else, right?

Alan has a hunch that this `spawn_local` thing might be causing a problem, so Alan reads the documentation. The function signature seems particuarly suspicious:

```rust
pub fn spawn_local<F>(future: F) 
where
    F: Future<Output = ()> + 'static
```

So, `spawn_local` only works with futures that return nothing - so far, so good - and are `'static`. Uh-oh. What does that last bit mean? Alan asks Barbara, who responds that it's the lifetime of the whole program. Yeah, but... the async function is part of the program, no? Why wouldn't it have the `'static` lifetime? Does that mean all functions that borrow values aren't `'static`, or just the async ones?

Barbara explains that when you borrow a value in a function, the function doesn't gain the lifetime of that borrow. Instead, the borrow comes with it's own lifetime, separate from the function's. The only time a function can have a non-`'static` lifetime is if the function *closes over* some other variable, like so:

```rust
fn benchmark_sort() -> usize {
    let mut num_times_called = 0;
    let test_values = vec![1,3,5,31,2,-13,10,16];

    test_values.sort_by(|a, b| {
        a.cmp(b)
        num_times_called += 1;
    });

    num_times_called
}
```

The function passed to `sort_by` has to borrow something not passed into it, namely the `num_times_called` variable. Hence, it has the lifetime of that borrow, not the whole program, because it can't be called anytime - only when `num_times_called` is a valid thing to read or write.

Async functions, it turns out, *close over all their parameters*! They *have to*, because the futures they return don't actually *have* function parameters that the user controls. So they have to copy everything they get passed into, and if that thing is borrowed, then the entire future is borrowed.

Barbara suggests changing all of the async function's parameters to be owned types. Alan asks Grace, who architected this project. Grace recommends holding a reference to the `Plugin` that owns the `World`, and then borrowing it whenver you need the `World`. That ultimately looks like the following:

At this point, Alan is looking at either rewriting a good chunk of the program to not use contexts, or rewriting his own code to sidestep the problem. Alan chooses the latter, by holding a reference to the thing that produces the context and then updating it after the async portion of the task has concluded:

```rust
async fn load_image(src: String, inside_of: usize, player: Arc<Mutex<Player>>) {
    let request = make_request(&url);
    let data = window().unwrap().fetch_with_request(&request).await.unwrap().etc.etc.etc;
    let image = parse_png(data, context);

    player.lock().unwrap().update(|world| {
        let new_node_index = world.nodes.len();
        if let Some(parent) = world.nodes.get(inside_of) {
            parent.set_child(new_node_index);
        }
        world.nodes.push(image.into());
    });
}
```

It works, well enough that Alan is able to finish his changes and PR them into the project. However, Alan wonders if this could be syntactically cleaner, somehow. Right now, async and update code have to be separated - if we need to do something with a `World`, then `await` something else, that requires jumping in and out of this `update` thing. It's a good thing that we only really *have* to be async in these loaders, but it's also a shame that we practically *can't* mix `async` code and `World`s.

## 🤔 Frequently Asked Questions

* **What are the morals of the story?**
    * Borrowing means that Rust has three function colors, not two:
      * Sync, which can both own and borrow it's parameters, or close over other variables at the expense of a lifetime.
      * Owned Async, which can only own parameters
      * Borrowed Async, which can both own and borrow parameters, but the latter results in closing over a borrow rather than gaining some new lifetime from the outside.
    * Non-`'static` Futures are of limited use as lifetimes are tied to the sync stack. I think you can use `block_on` with them but that's about it. Most practical executors need `'static` futures.
    * Borrowed Async functions would be useful if there was a way to have a future whose borrows were scoped to the moment they are `poll`ed.
* **What are the sources for this story?**
    * This is personal experience. Specifically, I had to do [almost exactly this dance][RuffleAsync] in order to get fetch to work in Ruffle.
    * I have omitted a detail from this story: in Ruffle, we use a GC library (`gc_arena`) that imposes a special lifetime on all GC references. This is how the GC library upholds it's memory safety invariants, but it's also what forces us to pass around contexts, and once you have that, it's natural to start putting even non-GC data into it. It also means we can't hold anything from the GC in the Future as we cannot derive it's `Collect` trait on an anonymous type.
* **Why did you choose Alan to tell this story?**
    * Lifetimes on closures is already non-obvious to new Rust programmers and using them in the context of Futures is particularly unintuitive.
* **How would this story have played out differently for the other characters?**
    * Niklaus probably had a similar struggle as Alan.
    * Grace would have felt constrained by the `async` syntax preventing some kind of workaround for this problem.
    * Barbara already knew about Futures and 'static and carefully organizes their programs accordingly.

[character]: ../characters.md
[status quo stories]: ./status_quo.md
[Alan]: ../characters/alan.md
[Grace]: ../characters/grace.md
[Niklaus]: ../characters/niklaus.md
[Barbara]: ../characters/barbara.md
[htvsq]: ../how_to_vision/status_quo.md
[cannot be wrong]: ../how_to_vision/comment.md#comment-to-understand-or-improve-not-to-negate-or-dissuade
[RuffleAsync]: https://github.com/ruffle-rs/ruffle/blob/master/core/src/loader.rs
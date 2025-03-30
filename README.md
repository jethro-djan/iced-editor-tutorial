This is a tutorial explaining the YouTube video [[Building a simple editor with iced](https://www.youtube.com/watch?v=gcBJ7cPSALo)] by @hecrj on how to create minimal editor using [iced.rs](https://www.iced.rs). 

[iced.rs](https://iced.rs/) is a GUI library that enables you to utilise [MVU or Elm Architecture](https://guide.elm-lang.org/architecture/) to make GUI apps in the Rust language. You would normally find this way of architecting GUIs common with functional languages like Elm and F#. But the same architecture can be leveraged to make GUIs in Rust because the language provides the same guarantees the architecture relies on in functional languages (immutability and referencial transparency). For more information on getting started with this library, head over to [Getting started - iced.rs](https://book.iced.rs/). 

Let us get into the it. Every `iced.rs` app has three main parts:
* State
* Messages
* `iced.rs` glue

For example, the state of this app is held in the struct `Editor`:
```rust
struct Editor {
    path: Option<PathBuf>,
    content: text_editor::Content,
    error: Option<Error>,
    theme: iced_highlighter::Theme,
    is_dirty: bool,
}
```
The struct `Editor` holds the state of the app. The changing of the state of the application gives the perception of interactivity. This is because one cannot change state directly but only through `update` messages. 

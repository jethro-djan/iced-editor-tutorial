This is a tutorial explaining the YouTube video [[Building a simple editor with iced](https://www.youtube.com/watch?v=gcBJ7cPSALo)] by @hecrj on how to create minimal text editor using [iced.rs](https://www.iced.rs). 

There are many ways to architect a GUI. By and large, most of the different ways make use of object-orientation. The UI widgets will most likely be classes which will have some properties and behaviours. It is said that GUI programming is one of the natural settings where object-oriented programming shines. [iced.rs](https://iced.rs/) is a GUI library that enables you to utilise [MVU or Elm Architecture](https://guide.elm-lang.org/architecture/) to make GUI apps in the Rust language. You would normally find this way of architecting GUIs common with functional languages like Elm and F#. But the same architecture can be leveraged to make GUIs in Rust because the language provides the same guarantees the architecture relies on in functional languages (immutability and referencial transparency). Further, Rust has first-class support for basic functional programming constructs. For more information on getting started with this library, head over to [Getting started - iced.rs](https://book.iced.rs/). 

Let us get into it. Every `iced.rs` app has three main parts:
* State (or Model)
* Message
* `iced.rs` glue

For example, the `State` of this app is held in the struct `Editor`:
```rust
struct Editor {
    path: Option<PathBuf>,
    content: text_editor::Content,
    error: Option<Error>,
    theme: iced_highlighter::Theme,
    is_dirty: bool,
}
```
In this way of architecting apps, one seeks to capture all possible states the app can be in. Our editor is basic; we only want to *edit the contents* of the file we have opened, *show the path* of the file, and *choose a theme* from the list of default themes supplied with the library. This accounts for three of `Editor`'s fields. The `error` field is for tasks that can fail through no fault of ours, like opening a file or saving one. The `is_dirty` field is to track unsaved changes so we can make the `save button` active or not. 

Changing the state of the application gives the perception of interactivity. `Messages` provide the ***only*** way to change the `State` of an app. In that way, the `State` itself becomes immutable, thus eliminating a whole class of errors which ensue from your app being in an illegal state. `Messages` are mostly used spell out user-initiated *activities* (like clicking button) and *events* that happen afterward (like what should happen when a button is clicked). Below is the `Message` enum for this app:
```rust
#[derive(Debug, Clone)]
enum Message {
    Edit(text_editor::Action),
    New,
    Open,
    FileOpened(Result<(PathBuf, Arc<String>), Error>),
    Save,
    FileSaved(Result<PathBuf, Error>),
    ThemeSelected(iced_highlighter::Theme),
}
```

The `iced.rs` library does the hard part for you: 
* it provides a runtime that continuously listens for interactions with the app, 
* and it provides common helpers one has come to expect from a GUI library.

The main API exposed by the library that creates an app for you is defined (in a simplified way) like this:
```rust
pub fn application<...>
    ( title: ..., update: ..., view: ...) 
-> Application<...>
```
This is a function that returns an instance of `Application` given the parameters listed. The `title` is a string displayed at the top of the window. Let's focus on the two parameters. The `view` parameter is actually a function that contains all your UI elements. For example, the `view` for the `iced-editor-app` looks like this:
```rust
fn view(&self) -> Element<'_, Message> {
    let controls = row![
        action(new_icon(), "New file", Some(Message::New)),
        action(open_icon(), "Open file", Some(Message::Open)),
        action(
            save_icon(),
            "Save file",
            self.is_dirty.then_some(Message::Save)
        ),
        horizontal_space(),
        pick_list(
            iced_highlighter::Theme::ALL,
            Some(self.theme),
            Message::ThemeSelected
        )
    ]
    .spacing(10)
    .align_y(Center);
    let input_area = text_editor(&self.content)
        .placeholder("")
        .on_action(Message::Edit)
        .highlight_with::<iced_highlighter::Highlighter>(
            iced_highlighter::Settings {
                theme: self.theme,
                token: self
                    .path
                    .as_ref()
                    .and_then(|path| path.extension()?.to_str())
                    .unwrap_or("rs")
                    .to_string(),
            },
            |highlight, _theme| highlight.to_format(),
        )
        .key_binding(|key_press| match key_press.key.as_ref() {
            keyboard::Key::Character("s") if key_press.modifiers.command() => {
                Some(text_editor::Binding::Custom(Message::Save))
            }
            _ => text_editor::Binding::from_key_press(key_press),
        });

    let status_bar = {
        let status = if let Some(Error::IOFailed(error)) = self.error.as_ref() {
            text(error.to_string())
        } else {
            match self.path.as_deref().and_then(Path::to_str) {
                Some(path) => text(path).size(14),
                None => text("New file"),
            }
        };

        let position = {
            let (line, column) = self.content.cursor_position();

            text(format!("{}:{}", line + 1, column + 1))
        };

        row![status, horizontal_space(), position].spacing(10)
    };

    column![controls, input_area, status_bar]
        .spacing(10)
        .padding(10)
        .into()
}
```
In `iced.rs`, the library nudges you towards creating UI elements as functions. This is different from the object-oriented approach, where UI widgets are classes with behaviours defined on it. As far as I know, all UI widgets are exposed as functions. Our simple app is visually a column with `controls`, `input_area` and `status_bar` stacked atop each other. This is similar to how other declarative frameworks like SwiftUI or Flutter do it. The `controls` and `status_area` UI elements are rows with other widgets in them.

The most important part, in my opinion, is the `update` function. `Messages` change the `State` of the app through this function. Here is a snippet of the `update` function in this app::
```rust
fn update(&mut self, message: Message) -> Task<Message> {
    match message {
        Message::Edit(action) => {
            self.is_dirty = self.is_dirty || action.is_edit();
            self.content.perform(action);
            self.error = None;

            Task::none()
        }
        Message::New => {
            // some code here
            Task::none()
        }
        Message::Open => Task::perform(pick_file(), Message::FileOpened),
        Message::FileOpened(Ok((path, content))) => {
            // some code here

            Task::none()
        }
        // More match statements...
    }
}
```
Every time a message is sent, the `update` function is called, and the `State` is changed, which in turns reloads the `view`. This may lead to another message being triggered; the whole process repeats. This is what I called the '`iced.rs` glue' earlier.

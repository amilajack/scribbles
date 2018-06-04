---
title: Virtual DOM for CLI and libui
categories:
  - rust
  - idea
discussions:
  "Twitter (@lqd)": "https://twitter.com/lqd/status/760223190182465538"
---

Among [other things]({% post_url 2016-05-19-things-to-rewrite-in-rust %}), I want to easily write UIs in Rust. (What I describe below could also be done in any other language though.)

## Contents
{:.no_toc}

* Table of contents
{:toc}

## Virtual DOM

Dealing with a virtual DOM (instead of directly manipulating the real DOM) was popularized by [React.js](https://facebook.github.io/react/). Advantages are:

- Developer declares what UI components should be shown, renderer determines good way to update your UI (clever diffing)
- Works nicely with reactive programming
- Works nicely with doing as little as possible in the render/UI thread (send diff result to UI thread which can just apply the changes)

The name "Virtual DOM" might suggest a direct relation to the HTML DOM implemented in browsers; I mean it in a more abstract way, though. (I just haven't come up with a good alternative name, yet!)

### Just a tree of UI components

While React.js started with rendering to HTML, there are several other targets available now, including React Native (iOS, Android) and HTML5 canvas.

Basic building blocks:

1. Define "UI component" trait that can be rendered.
2. UI components can be nested to build a tree (with exactly one root).
3. Implement clever diffing between two trees of UI components. (React uses the component type and `key` attributes unique among sibling components to short-circuit the otherwise `O(n^3)` diffing.)

### Abstract component definition

One nice thing about all things virtual DOM is that it also abstracts how component instances are created. While React introduced the [JSX syntax](https://facebook.github.io/jsx/) (think XML-like angular-bracket things in JS that compile to function calls), the [virtual-dom](https://www.npmjs.com/package/virtual-dom) module (also in JavaScript) includes a nice helper function that one can call like `h('.greeting', ['Hello ' + data.name])` (creates a div with class `greeting` and content `Hello ${data.name}`).

The abstract idea is this: UI components can have attributes and children (i.e., an array of more UI components). A plain string is also considered a UI component and is rendered as a simple text node.

For example: Your UI library gives you a bunch of functions with the signature `(attributes, children) -> VirtualUiNode`, and also exports the virtual DOM helper functions to update a tree of these `VirtualUiNodes`.

I can image it having an API like this:

```rust
fn render(name: &str) -> VirtualUiNode {
    div(map!["class" => "alert alert--info"], [
        p(map![], format!("Hi, {}", name)),
        p(map![], [
            a(map!["class" => "alert__close"], "x"),
        ]),
    ])
}

fn main() {
    let initial = render("Lorem");
    let renderer = vdom::Renderer::new(initial);
    
    let next = render("Ipsum");
    renderer.update(next);
}
```

## Render targets

### libui

[libui](https://github.com/andlabs/libui) is a "simple and portable (but not inflexible) GUI library in C that uses the native GUI technologies of each platform it supports". The Rust binding can be found in [libui-rs](https://github.com/pcwalton/libui-rs).

Looking at [one of the examples](https://github.com/pcwalton/libui-rs/blob/13299d28f69f8009be8e08e453a9b0024f153a60/ui/examples/controlgallery.rs), the code looks quite imperative.

Imagine a pseudo-Rust macro called `ui!` that matches like this: `($type:ident $label:expr? {$($key:ident => $val:expr),*}? [$($subcomponent:tt),*]?) => {...}` (disregarding text nodes for now). Assuming that most libui components can have attributes and sub-components, one could imagine defining the same UI like this:

```rust
fn run() {
    let menu = ui!(Menu {} [
        ("File" {} [
            ("Open" {onClick => open_clicked}),
            ("Save" {onClick => save_clicked}),
        ]),
        ("Edit" {} [
            ("Checkable Item" {checkable}),
            (---),
            ("Disabled Item" {disabled}),
            (preferences),
        ]), 
        ("Help" {}, [
            ("Help"),
            (about),
        ]),
    ]);

    let window = ui!(Window "ui Control Gallery" {
        w => 640, h => 480, menubar, margined,
        onClosing => |_| {
            ui::quit();
            false
        },
    } [
        (BoxControl {vertical, padded} [
            (Group "Basic Controls" {margined} [
                (Button "Button"),
                (Checkbox "Checkbox"),
                (Entry "Entry" {text => "Lorem ipsum"}),
                (label "label"),
                (---),
                (DatePicker),
                (TimePicker),
                (DateTimePicker),
                (FontButton),
                (ColorButton),
            ]),
        ]),
        (BoxControl {vertical, padded} [
            (Group "Numbers" {margined} [
                (BoxControl {padded} [
                    (Spinbox {min => 0, max => 100,
                        onChange => |spinbox| update(spinbox.value()),
                    }),
                ]),
                (BoxControl {padded} [
                    (Slider {min => 0, max => 100,
                        onChange => |slider| update(slider.value()),
                    }),
                ]),
                (ProgressBar),
            ]),
            (Group "Lists" {margined} [
                (BoxControl {padded} [
                    (Combobox {} [
                        "Combobox Item 1",
                        "Combobox Item 2",
                        "Combobox Item 3",
                    ]),
                    (Combobox {editable} [
                        "Editable Item 1",
                        "Editable Item 2",
                        "Editable Item 3",
                    ]),
                    (RadioButtons {} [
                        "Radio Button 1",
                        "Radio Button 2",
                        "Radio Button 3",
                    ]),
                ]),
            ]),
            (Tab {} [
                ("Page 1" {} [(BoxControl),]),
                ("Page 2" {} [(BoxControl),]),
                ("Page 3" {} [(BoxControl),]),
            ]),
        ]),
    ]);

    ui::main(menu, window);
}
```

Each sub-component or list of subcomponents can also be refactored into a variable using the same macro.

### CLI

(~~I think I've seen a virtual-dom-like CLI library some time ago, but can't find it right now.~~ Found it again: It's [react-blessed](https://github.com/Yomguithereal/react-blessed), a React-based renderer for the curses-like JS library [blessed](https://github.com/chjj/blessed).)

There are some ways to make fancy command like interfaces. I've recently used [termion](https://github.com/ticki/termion) and contributed to [inquirer-rs](https://github.com/Munksgaard/inquirer-rs).

Imagine defining a CLI select box like this:

```rust
fn render_select<C: Choice>(options: &[C], selected: usize) -> CliSection {
    let options = options.enumerate().map(|(option, index)| if index == selected {
        ui!(format!(" [x] {}", option.label))
    else {
        ui!(format!(" [ ] {}", option.label))
    });

    ui!(CliSection "Select" {header => "Please select an option:"} options)
}
```

## Updates

### Immediate-mode GUIs

[@lqd](https://twitter.com/lqd) mentioned [on Twitter](https://twitter.com/lqd/status/760223190182465538) that virtual DOM might be a poorer version of immediate-mode GUI APIs:

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr"><a href="https://twitter.com/killercup">@killercup</a> your vdom/libui post was cool, but to me vdom is also a poorer version of immediate-mode GUI APIs which would be even simpler :)</p>&mdash; Rémy Rakić (@lqd) <a href="https://twitter.com/lqd/status/760223190182465538">August 1, 2016</a></blockquote>

<blockquote class="twitter-tweet" data-conversation="none" data-lang="en"><p lang="en" dir="ltr"><a href="https://twitter.com/lqd">@lqd</a> interesting. do you have an example for nice immediate-mode GUI code?</p>&mdash; Pascal (@killercup) <a href="https://twitter.com/killercup/status/760224634855911424">August 1, 2016</a></blockquote>

<blockquote class="twitter-tweet" data-conversation="none" data-lang="en"><p lang="en" dir="ltr"><a href="https://twitter.com/killercup">@killercup</a> 1) <a href="https://t.co/zd9I0owlpf">https://t.co/zd9I0owlpf</a> 2) <a href="https://t.co/JgrbdaZGSw">https://t.co/JgrbdaZGSw</a> (page 34) 3) <a href="https://t.co/9lldQFUlWR">https://t.co/9lldQFUlWR</a> 4) surely on <a href="https://twitter.com/handmade_hero">@handmade_hero</a> as well</p>&mdash; Rémy Rakić (@lqd) <a href="https://twitter.com/lqd/status/760229545010163715">August 1, 2016</a></blockquote>

### React Fibre

[Andrew Clark](https://github.com/acdlite) published [an overview of the React Fiber Architecture](https://github.com/acdlite/react-fiber-architecture/blob/efbf8936293f7cc4e8a30f475ffd01087d4d974c/README.md) recently:

> React Fiber is an ongoing reimplementation of React's core algorithm. It is the culmination of over two years of research by the React team.
>
> The goal of React Fiber is to increase its suitability for areas like animation, layout, and gestures. Its headline feature is incremental rendering: the ability to split rendering work into chunks and spread it out over multiple frames.
>
> Other key features include the ability to pause, abort, or reuse work as new updates come in; the ability to assign priority to different types of updates; and new concurrency primitives.

It'll be interesting to see if React Fibre ends up being the same kind of abstraction I was describing here.

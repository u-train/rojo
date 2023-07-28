# Architecture

Rojo is a rather large project with a bunch of moving parts. While it's not too complicated in practice, it tends to be overwhelming because it's a fair bit of Rust and not very clear where to begin.

This document is a "what the heck is going on" level view of Rojo and the codebase that's written to make it more reasonable to jump into something. It won't go too into depth on *how* something is done, but it will go into depth on *what* is being done.

## Overarching

Rojo is divide into several components, each layering on top of each other provide Rojo's functionality.

At the core of Rojo, is `ServeSession`. As the name implies, it contains all of the components to keep a persistent DOM, react to events to update the DOM, and serve the DOM to consumers. It depends on:

- `RojoTree` to represent the DOM;
- `Project` to represent your root project file (e.g. `default.project.json`).
- [`Vfs`](#vfs) to provide a filesystem and emit events on changes;
- [`ChangeProcessor`](#changeprocessor) to process filesystem events from `Vfs` and consequently update the DOM through the [snapshotting system](#the-snapshotting-system);

It also provides an API for the higher level components so it can be used with the outside world.

- There is a `MessageQueue` of changes applied to the DOM.
- There is a channel to send changes to the `ServeSession` and update the DOM.
- And a `SessionId` to uniquely identify the `ServeSession`.

All of the public interfaces via CLI of Rojo are implemented using `ServeSession`.

### The Snapshotting System

To do what it does, Rojo has to do two main things: it must decide how the file system should map to Roblox and then send changes from the file system to the plugin. To accomplish this, Rojo uses what's referred to as snapshots.

Snapshots are essentially a capture of what a given Instance tree looks like at a given time. Once an initial snapshot is computed and sent to the plugin, any changes to the file system can be turned into a snapshot and compared directly against the previous snapshot, which Rojo can then use to make a set of patches that have to be applied by the plugin.

These patches represent changes, additions, and removals to the Roblox tree that Rojo creates and manages.

When generating snapshots, files are 'transformed' into Roblox objects through what's referred to as the `snapshot middleware`. As an example, this middleware takes files named `init.lua` and transforms them into a `ModuleScript` bearing the name of the parent folder. It's also responsible for things like JSON models and `.rbxm`/`.rbxmx` models being turned into snapshottable trees.

Inquiring minds should look at `snapshot/mod.rs` and `snapshot_middleware` for a more detailed explanation.

Because snapshots are designed to be translated into Instances anyway, this system is also used by the `build` command to turn a Rojo project into a complete file. The backend for serializing a snapshot into a file is provided by `rbx-dom`, which is a different project.

## Serve Command

There are two main pieces in doing this: the server and the Studio Plugin.

The server runs a local `LiveServer` on your computer (be it through a terminal, the visual studio extension, or a remote machine). It consumes a `ServeSession` and attaches a web server on top. The web server itself is very basic, consisting of around half a dozen endpoints. Generally, `LiveServer` acts as a middleman with the bulk of the work is performed by either the underlying `ServeSession` or the plugin. 

To serve a project to a connecting plugin, the server gathers data on all of the files in that project, puts it into a nice format, and then sends it to the plugin. After that, when something changes on the file system, the underlying `ServeSession` emits new patches. The web server has a `api/subscribe` endpoint where the plugin [long polls](https://en.wikipedia.org/wiki/Push_technology#Long_polling) on to receive the patches from the server and apply them to the datamodel in Studio.

When the plugin receives a patch from the server (whether it be the initial patch or any subsequent ones), the plugin reads through the patch and attempts to to apply the changes described by it. Any sugar (the patch visualizer, as an example) happens on top of the patches received from the server.

## The Plugin

This section of the document is left incomplete.

## Data Structures

Rojo has many data structures and their purpose might not be immediately clear at a glance. To alleviate this, they are documented below.

### Vfs

To learn more, read about [`memofs` architecture](crates/memofs/ARCHITECTURE.md).

### ServeSession

### ChangeProcessor

### RojoTree

### LifeServer

### InstanceSnapshot

# Rust Urbit HTTP API
This library wraps the Urbit ship http interface exposing it as an easy-to-use Rust crate.

All implementation details such as auth cookies, EventSource connections, tracking message ids, and other such matters are automatically handled for you, and as enables a greatly improved experience in writing Rust apps that interact with Urbit ships.

This crate currently enables devs to:
1. Authorize oneself and open a channel with the ship.
2. Subscribe to any app/path so that one can read the events currently taking place inside of the ship.
3. Issue `Poke`s to apps.

## Design
There are 3 main structs that this library exposes for interacting with an Urbit ship:
1. `ShipInterface`
2. `Channel`
3. `Subscription`

A `Subscription` is created by a `Channel` which is created by a `ShipInterface`. In other words, first you need to connect to an Urbit ship (using `ShipInterface`) before you can initiate a messaging `Channel`, before you can create a `Subscription` to an app/path.

### ShipInterface
The `ShipInterface` exposes two primary methods that will be useful when creating apps.

In short these allow you to create a new `ShipInterface` (thereby authorizing yourself with the ship), and create a new `Channel`.

```rust
/// Logs into the given ship and creates a new `ShipInterface`.
/// `ship_url` should be `http://ip:port` of the given ship. Example:
/// `http://0.0.0.0:8080`. `ship_code` is the code acquire from your ship
/// by typing `+code` in dojo.
pub fn new(ship_url: &str, ship_code: &str) -> Result<ShipInterface>;

/// Create a `Channel` using this `ShipInterface`
pub fn create_channel(&mut self) -> Result<Channel>;
```

### Channel
`Channel` is the most useful struct, because it holds methods related to interacting with both pokes and subscriptions.

It is instructive to look at the definition of the `Channel` struct to understand how it works:

```rust
// A Channel which is used to interact with a ship
pub struct Channel<'a> {
    /// `ShipInterface` this channel is created from
    pub ship_interface: &'a ShipInterface,
    /// The uid of the channel
    pub uid: String,
    /// The url of the channel
    pub url: String,
    // The list of `Subscription`s for this channel
    pub subscription_list: Vec<Subscription>,
    // / The `EventSource` for this channel which reads all of
    // / the SSE events.
    event_receiver: ReceiverSource,
    /// The current number of messages that have been sent out (which are
    /// also defined as message ids) via this `Channel`
    pub message_id_count: u64,
}
```

Once a `Channel` is created, an `EventSource` connection is created with the ship on a separate thread. This thread accepts all of the incoming events, and queues them on a (Rust) unbounded channel which is accessible internally via the `event_receiver`. This field itself isn't public, but processing events in this crate is handled with a much higher-level interface for the app developer.

Take note that a `Channel` has a `subscription_list`. As you will see below, each `Channel` exposes methods for creating subscriptions, which automatically get added to the `subscription_list`.
Once `Subscription`s are created/added to the list, the `Channel` will evidently start to receive event messages via SSE (which will be queued for reading in the `event_receiver`).

From the app developer's perspective, all one has to do is call the `parse_event_messages()` method on your `Channel`, and all of the queued events will be processed and passed on to the correct `Subscription`'s `message_list`. This is useful once multiple `Subscriptions` are created on a single channel, as the messages will be pre-sorted automatically for you.

Once the event messages are parsed, then one can simply call the `find_subscription` method in order to interact with the `Subscription` and read its messages.

The following are the useful methods exposed by a `Channel`:

```rust
/// Sends a poke over the channel
pub fn poke(&mut self, app: &str, mark: &str, json: &str) -> Result<Response>;

/// Create a new `Subscription` and thus subscribes to events on the ship with the provided app/path.
pub fn create_new_subscription(&mut self, app: &str, path: &str) -> Result<CreationID>;

/// Parses SSE messages for this channel and moves them into
/// the proper corresponding `Subscription`'s `message_list`.
pub fn parse_event_messages(&mut self);

/// Finds the first `Subscription` in the list which has a matching
/// `app` and `path`;
pub fn find_subscription(&self, app: &str, path: &str) -> Option<&Subscription>;

/// Finds the first `Subscription` in the list which has a matching
/// `app` and `path`, removes it from the list, and tells the ship
/// that you are unsubscribing.
pub fn unsubscribe(&mut self, app: &str, path: &str) -> Option<bool>;

/// Deletes the channel
pub fn delete_channel(self);
```


### Subscription
As mentioned in the previous section, a `Subscription` contains it's own `message_list` field where messages are stored after a `Channel` processes them.

From an app developer's perspective, this is the only useful feature of the `Subscription` struct. Once acquired, it is used simply to read the messages.

To improve the message reading experience, the `Subscription` struct exposes a useful method:

```rust
/// Pops a message from the front of `Subscription`'s `message_list`.
/// If no messages are left, returns `None`.
pub fn pop_message(&mut self) -> Option<String>;
```


## Code Examples


### Poke Example
This example displays how to connect to a ship using a `ShipInterface`, opening a `Channel`, issuing a `poke` over said channel, and then deleting the `Channel` to finish.

```rust
// Import the `ShipInterface` struct
use urbit_http_api::ShipInterface;

fn main() {
    // Create a new `ShipInterface` for a local ~zod ship
    let mut ship_interface =
        ShipInterface::new("http://0.0.0.0:8080", "lidlut-tabwed-pillex-ridrup").unwrap();
    // Create a `Channel`
    let mut channel = ship_interface.create_channel().unwrap();

    // Issue a poke over the channel
    let poke_res = channel.poke("hood", "helm-hi", "This is a poke");

    // Cleanup/delete the `Channel` once finished
    channel.delete_channel();
}
```


### Subscription Example
This example shows how to create, interact with, and delete a `Subscription`. In this scenario we desire to read messages from our `Subscription` for 10 seconds, and then perform cleanup.
```rust
use std::thread;
use std::time::Duration;
use urbit_http_api::ShipInterface;

fn main() {
    // Create a new `ShipInterface` for a local ~zod ship
    let mut ship_interface =
        ShipInterface::new("http://0.0.0.0:8080", "lidlut-tabwed-pillex-ridrup").unwrap();
    // Create a `Channel`
    let mut channel = ship_interface.create_channel().unwrap();
    // Create a `Subscription` for the `chat-view` app with the `/primary` path. This `Subscription`
    // is automatically added to the `Channel`'s `subscription_list`.
    channel
        .create_new_subscription("chat-view", "/primary")
        .unwrap();

    // Create a loop that iterates 10 times
    for _ in 0..10 {
        // Parse all of the event messages to move them into the correct
        // `Subscription`s in the `Channel`'s `subscription_list`.
        channel.parse_event_messages();

        // Find our chat `Subscription`
        let chat_sub = channel.find_subscription("chat-view", "/primary").unwrap();

        // Pop all of the messages from our `chat_sub` and print them
        loop {
            let pop_res = chat_sub.pop_message();
            if let Some(mess) = &pop_res {
                println!("Message: {:?}", mess);
            }
            // If no messages left, stop
            if let None = &pop_res {
                break;
            }
        }

        // Wait for 1 second before trying to parse the event messages again
        thread::sleep(Duration::new(1, 0));
    }

    // Once finished, unsubscribe/destroy our `Subscription`
    channel.unsubscribe("chat-view", "/primary");
    // Delete the channel
    channel.delete_channel();
}
```

-------

This library was created by ~mocrux-nomdep([Robert Kornacki](https://github.com/robkorn)).
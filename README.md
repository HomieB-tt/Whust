# Whust

A high-performance Rust client implementation for WhatsApp Web, enabling you to build applications that connect directly to WhatsApp's servers using the same protocol as the official web client.

## Overview

This project is a complete Rust port of the [whatsmeow](https://github.com/tulir/whatsmeow) Go library, implementing the WhatsApp Web binary protocol with full end-to-end encryption support via the Signal Protocol.

### Key Features

- **Full WhatsApp Web Protocol Support**: Complete implementation of the WhatsApp binary protocol
- **End-to-End Encryption**: Native Signal Protocol implementation for secure messaging
- **QR Code Pairing**: Connect to WhatsApp by scanning a QR code
- **Pair Code Authentication**: Alternative pairing method using a numeric code
- **One-on-One Messaging**: Send and receive text, media, and other message types
- **Group Messaging**: Create and manage groups, send and receive group messages
- **Media Handling**: Download and upload images, videos, documents, and stickers
- **Message Reactions**: Send and receive reactions to messages
- **Message Edits**: Edit previously sent messages
- **Presence & Chat State**: Track online status and typing indicators
- **Contacts Management**: Sync and manage contacts
- **Blocking**: Block and unblock contacts
- **Persistent Sessions**: Automatically reconnect with session restoration

## Architecture

The project is organized into a Cargo workspace with three main crates:

```
whatsapp-rust/
├── wacore/                 # Platform-agnostic core library
│   ├── binary/              # Binary protocol encoding/decoding
│   ├── appstate/           # App state sync protocol
│   ├── libsignal/          # Signal Protocol implementation
│   ├── derive/             # Custom derive macros
│   └── noise/              # Noise Protocol framework
├── waproto/                # Protocol Buffers definitions
├── whatsapp-rust/          # Main client library
│   ├── src/client.rs       # Central client implementation
│   ├── src/send.rs        # Outgoing message encryption
│   ├── src/message.rs     # Incoming message decryption
│   ├── src/download.rs    # Media download logic
│   ├── src/upload.rs      # Media upload logic
│   ├── src/features/      # High-level APIs (groups, contacts, etc.)
│   ├── src/handlers/      # Protocol stanza handlers
│   └── src/store/         # State management
├── transports/tokio-transport/   # Tokio-based WebSocket transport
├── http_clients/ureq-client/     # HTTP client implementation
└── storages/sqlite-storage/      # SQLite persistence layer
```

## Installation

Add `whatsapp-rust` to your `Cargo.toml`:

```toml
[dependencies]
whatsapp-rust = "0.2"
```

Or use the workspace for development:

```bash
git clone https://github.com/HomieB-tt/Whust
cd Whust
cargo build
```

## Quick Start

### Basic Example

```rust
use whatsapp_rust::bot::{Bot, MessageContext};
use whatsapp_rust::store::SqliteStore;
use whatsapp_rust_tokio_transport::TokioWebSocketTransportFactory;
use whatsapp_rust_ureq_http_client::UreqHttpClient;

#[tokio::main]
async fn main() -> Result<(), anyhow::Error> {
    let backend = SqliteStore::new("whatsapp.db").await?;
    let transport_factory = TokioWebSocketTransportFactory::new();
    let http_client = UreqHttpClient::new();

    let mut bot = Bot::builder()
        .with_backend(backend)
        .with_transport_factory(transport_factory)
        .with_http_client(http_client)
        .on_event(|event, client| {
            async move {
                match event {
                    Event::PairingQrCode { code, timeout } => {
                        println!("QR Code (valid {}s): {}", timeout.as_secs(), code);
                    }
                    Event::PairingCode { code, timeout } => {
                        println!("Pair Code (valid {}s): {}", timeout.as_secs(), code);
                    }
                    Event::Message(msg, info) => {
                        if let Some(text) = msg.text_content() {
                            println!("Received: {}", text);
                        }
                    }
                    Event::Connected(_) => {
                        println!("Connected!");
                    }
                    _ => {}
                }
            }
        })
        .build()
        .await?;

    bot.run().await?;
    Ok(())
}
```

### Sending a Message

```rust
use waproto::whatsapp as wa;

let message = wa::Message {
    conversation: Some("Hello, World!".to_string()),
    ..Default::default()
};

let message_id = client.send_message(&jid, message).await?;
println!("Message sent: {}", message_id);
```

### Sending Media

```rust
let media_data = std::fs::read("image.jpg")?;
let upload = client.upload(media_data, MediaType::Image).await?;

let message = wa::Message {
    image_message: Some(Box::new(wa::message::ImageMessage {
        mimetype: Some("image/jpeg".to_string()),
        url: Some(upload.url),
        direct_path: Some(upload.direct_path),
        media_key: Some(upload.media_key),
        file_enc_sha256: Some(upload.file_enc_sha256),
        file_sha256: Some(upload.file_sha256),
        file_length: Some(upload.file_length as u64),
        ..Default::default()
    })),
    ..Default::default()
};

client.send_message(&jid, message).await?;
```

### Creating a Group

```rust
use whatsapp_rust::{Groups, GroupCreateOptions};

let options = GroupCreateOptions::new("Group Name".to_string())
    .with_participants(vec![participant_jid1, participant_jid2])
    .with_description("A test group".to_string());

let result = client.create_group(options).await?;
println!("Group created: {}", result.group_jid);
```

### Working with Contacts

```rust
use whatsapp_rust::Contacts;

let contact = client.get_contact_info(&jid).await?;
println!("Name: {:?}", contact.name);

let profile_picture = client.get_profile_picture(&jid, false).await?;
println!("Profile picture URL: {}", profile_picture.url);
```

## Authentication

### QR Code Authentication

Run your application and scan the QR code with WhatsApp:

1. Open WhatsApp on your phone
2. Tap the three dots → Linked Devices
3. Tap "Link a Device"
4. Scan the QR code displayed by your application

### Pair Code Authentication

Alternatively, you can use a numeric pair code:

```rust
use whatsapp_rust::PairCodeOptions;

let options = PairCodeOptions {
    phone_number: "15551234567".to_string(),
    custom_code: None, // Auto-generated if None
    ..Default::default()
};

let bot = Bot::builder()
    .with_pair_code(options)
    // ... other options
    .build()
    .await?;
```

## Event Handling

The client uses an event-driven architecture:

```rust
enum Event {
    Connected(ConnectionInfo),
    PairingQrCode { code: String, timeout: Duration },
    PairingCode { code: String, timeout: Duration },
    Message(wa::Message, MessageInfo),
    Receipt(ReceiptEvent),
    LoggedOut(LogoutReason),
    Presence(PresenceEvent),
    ChatState(ChatStateEvent),
    // ... more events
}
```

## Message Types

Supported message types include:

- **Text Messages**: `conversation`, `extendedTextMessage`
- **Media Messages**: `imageMessage`, `videoMessage`, `audioMessage`, `documentMessage`, `stickerMessage`
- **Location**: `locationMessage`
- **Contact**: `contactMessage`, `contactArrayMessage`
- **Reactions**: `reactionMessage`
- **Polls**: `pollCreationMessage`, `pollUpdateMessage`
- **Status**: `statusV3Message`
- **Buttons**: `buttonsMessage`, `buttonsResponseMessage`
- **Lists**: `listMessage`, `listResponseMessage`

## Configuration Options

### Client Builder

```rust
Bot::builder()
    .with_backend(store)
    .with_transport_factory(transport)
    .with_http_client(http_client)
    .with_pair_code(pair_code_options)  // Optional
    .with_version((2, 3000, 1027868167)) // Optional, auto-fetched
    .build()
    .await
```

### Send Options

```rust
pub struct SendOptions {
    pub preview_url: bool,           // Enable link previews
    pub link_preview_format: LinkPreviewFormat,
    pub thumbnail: Option<Vec<u8>>,   // Custom thumbnail
    // ... more options
}
```

## State Management

The library provides a persistence layer for storing:

- **Device State**: Session keys, identity keys, registration info
- **Sessions**: Signal Protocol sessions for contacts
- **PreKeys**: Pre-key bundles for establishing sessions
- **App State**: Sync state patches
- **History Sync**: Message history for missed notifications

### SqliteStore

```rust
let store = SqliteStore::new("whatsapp.db").await?;
```

## Advanced Features

### Signal Protocol

The library includes a complete Signal Protocol implementation:

- **Session Management**: Establish and maintain encrypted sessions
- **Sender Keys**: For group encryption
- **Identity Keys**: Verify contact identities
- **PreKey Bundles**: Exchange keys for session establishment

### Media Encryption

Automatic encryption/decryption for media:

- **Download**: Encrypted media → decrypted bytes
- **Upload**: Plain bytes → encrypted upload

### WebSocket Transport

The Tokio-based WebSocket transport provides:

- Automatic reconnection
- Keep-alive pings
- Connection state management
- Noise Protocol handshake

## Development

### Building

```bash
cargo build --release
```

### Running Tests

```bash
cargo test --all
```

### Code Formatting

```bash
cargo fmt
```

### Linting

```bash
cargo clippy --all-targets
```

### Benchmarks

```bash
cargo bench --all
```

## Dependencies

### Runtime

- **Tokio**: Async runtime
- **SQLite/Diesel**: Persistence (optional, enabled by default)
- **WebSocket**: Connection management

### Protocol

- **Signal Protocol**: E2E encryption
- **Noise Protocol**: Transport encryption
- **Protocol Buffers**: Message serialization

## Error Handling

```rust
use whatsapp_rust::store::error::StoreError;
use whatsapp_rust::socket::error::SocketError;

match client.send_message(&jid, message).await {
    Ok(message_id) => println!("Sent: {}", message_id),
    Err(StoreError::SessionNotFound) => {
        // Need to establish session first
    }
    Err(SocketError::NotConnected) => {
        // Reconnect before sending
    }
    Err(e) => {
        eprintln!("Error: {}", e);
    }
}
```

## Limitations

- **Phone Number Registration**: Must link to an existing WhatsApp account via QR/pair code
- **Voice/Video Calls**: Not yet implemented
- **Status Updates**: Limited support
- **Broadcast Lists**: Not implemented

## Contributing

Contributions are welcome! Please read the [AGENTS.md](AGENTS.md) for detailed guidelines on:

- Architecture patterns
- Protocol implementation approach
- State management conventions
- Testing requirements

## References

- [WhatsApp Web JavaScript](docs/captured-js/) - Captured protocol traces
- [whatsmeow](https://github.com/tulir/whatsmeow) - Original Go implementation
- [Signal Protocol](https://signal.org/docs/) - Signal Protocol documentation
- [Noise Protocol](https://noiseprotocol.org/) - Noise Protocol framework

## License

MIT License - see [LICENSE](LICENSE) file for details.

## Acknowledgments

- [whatsmeow](https://github.com/tulir/whatsmeow) - Primary reference implementation
- [Signal Protocol Library](https://github.com/signalapp/libsignal-client) - Protocol reference
- [WhatsApp Web](https://web.whatsapp.com/) - Protocol reverse engineering source

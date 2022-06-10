# Practical Rust
This book is dedicated to helping Rust users quickly find solutions to
ordinary problems. Ordinary problems include:

- How do I connect to Postgres?
- What's the "right" way to handle errors?
- How do I download and scrape this website?
- etc. 

Rust leans heavily on its (excellent) ecosystem of 3rd-party
open source software. Common tasks like serialization (`serde`),
interfacing with a database (`sqlx`), making HTTP requests (`hyper`/
`reqwest`), *serving* HTTP requests (`actix-web`/`axum`/ `warp`/
`rocket`), etc. are all delegated to outside the official Rust
project.

This system has several distinct advantages, but one important
disadvantage is the fragmentation of learning material and general
confusion for beginners. We can't expect every new Rust user to show
up in [r/rust][rust-subreddit] to ask which HTTP client to use.

Practical Rust attempts to alleviate some of this pressure. Our goal
is to be a first stop for Rust users wondering if there's a well-trodden
path to accomplishing whatever task they have in front of them.

## Get in touch
Check out our Matrix space at [`#practicalrust:matrix.org`][space].

[rust-subreddit]: https://reddit.com/r/rust
[space]: https://matrix.to/#/#practicalrust:matrix.org

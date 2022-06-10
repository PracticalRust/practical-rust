# Designing Data Structures

> Rule 5.  Data dominates.  If you've chosen the right data structures and organized things well, the algorithms will almost always be self-evident.  Data structures, not algorithms, are central to programming. ([Rob Pike](http://doc.cat-v.org/bell_labs/pikestyle))

Data structures lie at the heart of all computing, and they're much too great a topic to cover here in any meaningful way. However, there are tips and tricks most experienced Rust programmers encounter at some point. Below is a loosely organized, ever-growing list of patterns at least tangentially related to data-structure design.


## Structuring complex enums

Occasionally we may need to have a somewhat complex enum type. There are some subtleties around structuring complex enums that can help with the ergonomics of actually *using* those types down the line.

As a motivating example, let's say that we're writing a backend API for a microblogging platform. Specifically, we're designing an endpoint called `/post` that will allow a user to submit a POST request carrying the content of their post, and our endpoint will write the content to our platform.

The catch is that there are two types of posts: text posts and picture posts. So it makes sense to come up with a type that can represent either one of them. This is exactly the purpose of Rust's enums.

### Pattern: bag-of-struct-variants

So we build the following type:

```rust
enum PostContent {
    Text {
        username: String,
        time: u64,
        location: (f32, f32),
        text: String,
    },
    Image {
        username: String,
        time: u64,
        location: (f32, f32),
        image_data: Vec<u8>,
        caption: String,
    }
}
```

This is the natural starting place for when we're thinking about just coming up with *something* that will work. Basically, we make a [struct variant][defining-enums] for each possible case and then fill that variant with every field we'll need in the corresponding case.

[defining-enums]: https://doc.rust-lang.org/rust-by-example/custom_types/enum.html

There are some downsides to this approach. First, there's often duplication of the fields that the different variants have in common. Every variant of `PostContent` has the fields `username`, `time`, and `location`, but this fact isn't expressed in the type system. This leads to the creation of methods like the following:

```rust
# enum PostContent {
#     Text {
#         username: String,
#         time: u64,
#         location: (f32, f32),
#         text: String,
#     },
#     Image {
#         username: String,
#         time: u64,
#         location: (f32, f32),
#         image_data: Vec<u8>,
#         caption: String,
#     }
# }
impl PostContent {
    fn username(&self) -> &str {
        match self {
            PostContent::Text { username, .. }
            | PostContent::Image { username, .. } => username
        }
    }
}
```

The `username` method expresses the fact that `PostContent` will always have a username field. This *works*. There's nothing terribly wrong with it. But… it's a little unsatisfying. And verbose. Surely there's a way we can express the guaranteed existence of `username`, `time`, and `location` without manually constructing a method for each of them.

Another issue is that the match ergonomics can be a little verbose. Let's say we want to write out a log for the request with all the relevant information. Here's how we might do that:

```rust
# enum PostContent {
#     Text {
#         username: String,
#         time: u64,
#         location: (f32, f32),
#         text: String,
#     },
#     Image {
#         username: String,
#         time: u64,
#         location: (f32, f32),
#         image_data: Vec<u8>,
#         caption: String,
#     }
# }
# fn get_post() -> PostContent {
#     PostContent::Text {
#         username: String::new(),
#         time: 0,
#         location: (0.0, 0.0),
#         text: String::new(),
#     }
# }
let post_content = get_post();

match post_content {
    PostContent::Text { username, time, location, .. } => println!(
        "username={username} \
        time={time} \
        location={location:?}"
    ),
    PostContent::Image { username, time, location, caption, .. } => println!(
        "username={username} \
        time={time} \
        location={location:?} \
        caption={caption}"
    )
}
```

As we add more logged information to these variants, the match patterns become longer and longer. I personally wouldn't want to see something like this in my codebase:

```rust,ignore
match post_content {
    PostContent::Image {
        username,
        full_name,
        time,
        location,
        caption,
        authentication_token,
        in_reply_to,
    } => {
        println!(todo!())
    }
}
```

I mean — it's *fine*. Nothing horrific. But it does get worse as the types get more complicated, and we can do better.

### Pattern: structs-in-tuple-variants

Instead of using struct variants, we can optionally create dedicated types for each of our cases, and then stick them in tuple variants. This will help solve the verbosity problem.

```rust
enum PostContent {
    Text(TextContent),
    Image(ImageContent),
}

struct TextContent {
    username: String,
    time: u64,
    location: (f32, f32),
    text: String,
}

struct ImageContent {
    username: String,
    time: u64,
    location: (f32, f32),
    image_data: Vec<u8>,
    caption: String,
}

# fn get_post() -> PostContent {
#     PostContent::Text(TextContent {
#         username: String::new(),
#         time: 0,
#         location: (0.0, 0.0),
#         text: String::new(),
#     })
# }
let post_content = get_post();

match post_content {
    PostContent::Text(text)=> println!(
        "username={} time={} location={:?}",
        text.username,
        text.time,
        text.location,
    ),
    PostContent::Image(image) => println!(
        "username={} time={} location={:?} caption={}",
        image.username,
        image.time,
        image.location,
        image.caption,
    )
}
```

We broke out the data from each variant into its own struct and then switched to tuple variants in the enum to hold our structs. This is primarily beneficial when you're frequently using lots of different fields from the variants. If you're only using one or two fields, the pattern matching won't get too crazy. 

One thing we lose is the ability to use the handy identifiers-in-format-strings approach to string interpolation. C'est la vie.

*Also*, we still haven't solved one of our two problems: the fields `username`, `time`, and `location` aren't guaranteed by the type system to be present. Boooooo.

### Pattern: struct-in-struct-variant-of-enum-in-struct

For our final trick, we create one last type and stick in it all the fields that are guaranteed to be present. Those fields are removed from the enum variants, and the enum itself has been renamed rather uncreatively to `InnerPostContent`. 

```rust
struct PostContent {
    username: String,
    time: u64,
    location: (f32, f32),
    inner: InnerPostContent,
}

enum InnerPostContent {
    Text(TextContent),
    Image(ImageContent),
}

struct TextContent {
    text: String,
}

struct ImageContent {
    image_data: Vec<u8>,
    caption: String,
}

# fn get_post() -> PostContent {
#     PostContent {
#         username: String::new(),
#         time: 0,
#         location: (0.0, 0.0),
#         inner: InnerPostContent::Text(TextContent {
#             text: String::new(),
#         })
#     }
# }

let post_content = get_post();

match post_content.inner {
    InnerPostContent::Text(_)=> println!(
        "username={} time={} location={:?}",
        post_content.username,
        post_content.time,
        post_content.location,
    ),
    InnerPostContent::Image(image) => println!(
        "username={} time={} location={:?} caption={}",
        post_content.username,
        post_content.time,
        post_content.location,
        image.caption,
    )
}
```

Now the type systems knows *for sure* that any instance of `PostContent` will have a `username`, `time`, and `location`. Accessor methods can be defined more compactly, or omitted entirely if the fields are public. This pattern does create a bit more boilerplate, but in the end **it manages to encode more of our business logic and invariants in the type system** than the other two patterns. That's a win.
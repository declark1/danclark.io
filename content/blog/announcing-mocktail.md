+++
title = "Announcing mocktail: HTTP & gRPC server mocking for Rust"
date = 2025-03-19
[taxonomies]
tags = ["rust"]
+++

This is an announcement post for [mocktail](https://github.com/IBM/mocktail), a **minimal** crate for mocking HTTP and gRPC servers in Rust, with native support for streaming.

## Motivation

**Simplicity to mock complexity.**

At work, my team is building AI platform services in Rust. These services call out to various HTTP and gRPC services, a mix of unary and streaming methods. Our requirements are simple: to properly test our code, we need to mock these various backend services, otherwise we have to deploy real services just for testing. That's no fun.

While there are some great mocking libraries in the Rust ecosystem such as [httpmock](https://github.com/alexliesenfeld/httpmock), [wiremock-rs](https://github.com/LukeMathWalker/wiremock-rs), and [stubr](https://github.com/beltram/stubr), none of them support gRPC or streaming. 

I reviewed these crates to see if it would be feasible to contribute streaming and gRPC support, but it really did not seem like a good fit for their designs, so I decided to experiment with creating a new crate from the ground up.

**Key requirements:**
- A simple, ergonomic API to define mocks in Rust
- First-class support for streaming and gRPC
- A minimal set of "matchers" to match requests to mock responses by method, path, and body

The result of this experiment is mocktail, which I am happy to share with the community, in case others have similar needs. I'll share additional technical details and learnings from this in future posts.

## Example

A basic usage example:

```rust
use anyhow::Error;
use mocktail::prelude::*;

#[tokio::test]
async fn test_example() -> Result<(), Error> {
    // Create a mock set
    let mut mocks = MockSet::new();

    // Build a mock that returns a "hello world!" response
    // to POST requests to /hello with the text "world"
    // in the body.
    mocks.mock(|when, then| {
        when.post()
            .path("/hello")
            .text("world");
        then.text("hello world!");
    });
    // NOTE: Shout out to httpmock for inspiring this 
    // closure-builder API design :)

    // Create and start a mock server
    let mut server = MockServer::new("example")
        .with_mocks(mocks);
    server.start().await?;

    // Create a client
    let client = reqwest::Client::builder().build()?;

    // Send a request that matches the mock created above
    let response = client
        .post(server.url("/hello"))
        .body("world")
        .send()
        .await?;
    assert_eq!(response.status(), http::StatusCode::OK);
    let body = response.text().await?;
    assert_eq!(body, "hello world!");

    // Send a request that doesn't match a mock
    let response = client
        .get(server.url("/nope"))
        .send()
        .await?;
    assert_eq!(response.status(), http::StatusCode::NOT_FOUND);

    // Mocks can also be registered to the server directly
    // Register a mock that will match the request above 
    // that returned 404
    server.mock(|when, then| {
        when.get()
            .path("/nope");
        then.text("yep!");
    });

    // Send the request again, it should now match
    let response = client
        .get(server.url("/nope"))
        .send()
        .await?;
    assert_eq!(response.status(), http::StatusCode::OK);
    let body = response.text().await?;
    assert_eq!(body, "yep!");

    // Mocks can be cleared from the server, enabling server reuse
    server.mocks.clear();

    Ok(())
}
```

See the [book](https://ibm.github.io/mocktail/) for additional details (WIP), [docs](https://docs.rs/mocktail/latest/mocktail/), and [examples](https://github.com/IBM/mocktail/tree/main/mocktail-tests/tests/examples) in the `mocktail-tests` crate for more.

#### This is an early stage alpha, subject to bugs and breaking changes.

## Next Steps
- Add TLS support
- Add additional matchers
- Keepin it minimal, no bloat or advanced features planned

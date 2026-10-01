---
layout: default
title: Aura's network tests without a server
description: How shimforge replaced network-dependent telemetry tests with small Rust unit tests.
date: 2026-10-02
---

# Aura's network tests without a server

Aura sends telemetry with `reqwest`. Before a request is reported, its sink classifies the error:

- an HTTP 4xx or 5xx status
- a timeout
- a connection failure
- another request error

Those labels matter to the retry and reporting logic. They also made the unit tests depend on a running mock server or a closed localhost port.

That is the problem [Aura PR #711](https://github.com/mezmo/aura/pull/711) solved with shimforge. The production telemetry code stayed the same. Only the tests changed.

## Start with a real error

`reqwest::Error` cannot be built by calling a public constructor. The test still needs a real value, so it creates one from a URL that fails while the request is being built:

```rust
fn url_error() -> reqwest::Error {
    reqwest::Client::new()
        .post("not a url")
        .build()
        .expect_err("the URL should fail before any request is sent")
}
```

This error has no HTTP status and is not a timeout or connection error. That makes it a clean input for each classification test.

## Mock the question the classifier asks

Aura's classifier calls methods such as `status()` and `is_timeout()` on the error. Shimforge can replace those methods for the current test session.

The status test supplies a 404 without starting a server:

```rust
use shimforge::{mock, Session};

fn classify_with_status(code: reqwest::StatusCode) -> &'static str {
    let error = url_error();
    let mut session = Session::new_global();
    let status = mock!(
        session,
        reqwest::Error::status,
        fn(&reqwest::Error) -> Option<reqwest::StatusCode>
    );

    status.expect().once().returns(Some(code));
    classify_post_error(&error)
}

#[test]
fn not_found_is_classified_as_an_http_client_error() {
    assert_eq!(
        classify_with_status(reqwest::StatusCode::NOT_FOUND),
        "http_4xx"
    );
}
```

The timeout test controls the two checks that can overlap:

```rust
#[test]
fn timeout_gets_the_timeout_label() {
    let error = url_error();
    let mut session = Session::new_global();

    let is_timeout = mock!(
        session,
        reqwest::Error::is_timeout,
        fn(&reqwest::Error) -> bool
    );
    is_timeout.expect().once().returns(true);

    let is_request = mock!(
        session,
        reqwest::Error::is_request,
        fn(&reqwest::Error) -> bool
    );
    is_request.expect().returns(true);

    assert_eq!(classify_post_error(&error), "timeout");
}
```

The error is still real. Shimforge only changes the answers used to select this branch. The test can therefore check the label and the order of the classifier without waiting for a delayed response.

The connection test uses the same pattern:

```rust
let is_connect = mock!(
    session,
    reqwest::Error::is_connect,
    fn(&reqwest::Error) -> bool
);
is_connect.expect().once().returns(true);
assert_eq!(classify_post_error(&error), "network");
```

## What changed in Aura

The classification tests no longer start WireMock or connect to a refused port. They keep their names and expected labels, but each test now sets only the `reqwest::Error` method it needs.

That also removes a platform-dependent failure. A refused localhost connection can take long enough on Windows to hit the test timeout. The new unit test does not open a socket, so the result does not depend on the host network stack.

Aura still keeps real request coverage for the wire format and delivery path. Shimforge handles the smaller question: given this error state, which label should the sink produce?

## Keep network tests focused

A server test is useful when the server interaction is what needs checking. It is extra setup when the code under test only classifies an error.

Shimforge lets Aura use a real error value and control one method at a time. That keeps the unit test close to the decision in the code, while end-to-end tests continue to cover real HTTP behavior.

## Learn more

- [Aura PR #711](https://github.com/mezmo/aura/pull/711)
- [Shimforge on GitHub](https://github.com/XTSoftwareLabs/shimforge)
- [shimforge home page](https://shimforge.com)

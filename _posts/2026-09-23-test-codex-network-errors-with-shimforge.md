---
layout: default
title: How Shimforge tests the network failures Codex could not reach
description: Use a real reqwest error and one shimforge mock to test Codex retry branches without a server.
date: 2026-09-23
---

# How Shimforge tests the network failures Codex could not reach

Some error branches are easy to write and hard to test. Codex has retry and error reporting code that checks flags on `reqwest::Error`, such as `is_connect()`, `is_body()`, and `is_decode()`.

`reqwest::Error` has no public constructor for these cases. A normal test must create a real failure: start a server that stalls, use a closed port, or wait for a timeout. Those tests are slow and can behave differently on each operating system.

[Codex issue #46596](https://github.com/openai/codex/issues/46596) shows a smaller path. Build a real error locally, then replace only the flag that the code checks.

## Build an error without a request

Add shimforge as a development dependency:

```toml
[dev-dependencies]
shimforge = "0.1.3"
```

The URL below is invalid, so `reqwest` returns an error while building the request. No request is sent:

```rust
fn registry_request_error() -> reqwest::Error {
    reqwest::Client::new()
        .get("not a url")
        .build()
        .expect_err("the URL should be rejected locally")
}
```

This gives the test a real `reqwest::Error`. The test does not need to construct the type itself or start a network service.

## Replace the flag the branch reads

The Codex retry path checks whether the error is a connection error. The small function below shows the part that matters:

```rust
fn is_retryable_registry_error(error: &reqwest::Error) -> bool {
    error.is_connect() || error.is_timeout()
}
```

Shimforge replaces `is_connect` for this test session:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use shimforge::{mock, Session};

    #[test]
    fn connection_failures_are_retryable() {
        let mut session = Session::new();
        let is_connect = mock!(
            session,
            reqwest::Error::is_connect,
            fn(&reqwest::Error) -> bool
        );

        is_connect.expect().once().returns(true);

        assert!(is_retryable_registry_error(&registry_request_error()));
        session.verify();
    }
}
```

The production function stays as it is. The test names the method, gives its signature, and controls the result for one call. The same shape covers `is_body()` and `is_decode()` by changing the method that is mocked and the value returned.

## Test the branch, not the machine

This pattern lets Codex test the decision it needs to make: retry this error or report it. The test does not depend on a listening port, a delayed server, or the network stack on the host running it.

That is where shimforge helps with code that was difficult to reach. It does not replace the error type or change the production API. It gives the test control at the call that decides which branch runs.

## Learn more

- [Codex issue #46596](https://github.com/openai/codex/issues/46596)
- [shimforge home page](https://shimforge.com)
- [shimforge on GitHub](https://github.com/XTSoftwareLabs/shimforge)

+++
title = "Writing a basic rate limiter in Rust"
description = "Writing a very basic rate limiter in Rust using Token Bucket algorithm"
draft = false
date = "2022-09-02"
template = "page.html"
menu = "sandesh"
+++
> I am looking for a job in Rust. Please (contact)[mailto://mail.sandeshbhusal@gmail.com]

## Introduction: 

Rate limiting algorithms are quintessential in limiting bad actors from DOSing your resources. In this post, I am implementing a very basic rate limiter based on Token bucket algorithm.

## The algorithm:

Token bucket works in the following way: 

1. Start the rate limiter
2. For every client request, check if the client has enough "tokens"
3. For every request sent from the client, decrement the client's token count. 
4. Increment the token content for the client in every 1/R seconds where R = request rate = number of request / window interval.

For #4, an example can be 2/5 where 2 requests are allowed every 5 seconds for a particular client. 

## Implementing the algorithm:

1. Let's begin by creating a project. 

```bash
cargo new --lib ratelimiter
```

Inside `src/lib.rs` we will import everything we need for now. 

```rust
use std::{collections::HashMap, hash::Hash, time::Instant};
```

2. Then we create a scaffold for the client data. For every client, we need to record two things: 
   1. How many tokens the client has left
   2. When was the last time the client made a successful request. 

For #2.2, we can use Rust's `std::time::Instant` since it will let us compute the elapsed time in different granularity as well.

Let's create the client data struct: 

```rust
struct client_info {
    last_sent: Instant,
    available_tokens: f64
}
```

The `available_tokens` field is floating point since we will need to do floating-point calculations to increment the tokens. 

Let's create a RateLimiter struct that will be exported by this library: 

```rust
pub struct RateLimiter<T> 
    where 
    T: Eq + Hash 
{
    client_data: HashMap<T, client_info>,
    max_allowed: f64,
    window_duration: f64,
    runnable: fn()    
}
```

To describe:
1. `client_data` is a hashmap that holds all clients' information for client_info. Each client is identified by a unique identifier, which is of the type `T` and needs to implement `Eq` and `Hash` traits. 
2. `max_allowed` is the number of maximum requests allowed in the `window_duration`.
3. `runnable` is a runnable function which is of a very simple signature `fn()` for now. 

Let's create a runnable function so that we can test our rate limiter later. 

```rust
fn runnable() {
    println!("I ran!");
}
```

Now let's go about implementing our `client_info`. Since `Instant` does not implement `Default` we will need to create a trivial `new` function for this struct. 

```rust
impl client_info {
    fn new() -> Self {
        return Self {
            last_request: Instant::now(),
            available_tokens: 0.0
        }
    }
}
```

Next is the implementation of our `RateLimiter` struct. 

```rust
impl<T> RateLimiter <T>
where 
    T: Eq + Hash 
{
    fn new(allowed: f64, time_interval: f64, function_to_run: fn()) -> Self {
        Self {
            client_data: HashMap::new(),
            max_allowd: allowed,
            window_duration: time_interval,
            runnable: function_to_run,
        }
    }
}
```

The `new` function returns the `RateLimiter` object defaults. The meat of the RateLimiter is in its `run` function which we will implement now. The `run` function takes in a client ID and runs the `runnable` function if the rate limit requirements are passed.

```rust
fn run(&mut self, client_id: T) -> Result<(), String> {
    // default client entry if the client has not made requests previously.
    let mut default_client_entry = client_info::new();
    default_client_entry.available_tokens = self.max_allowed;

    // extract the entry from client_data hashmap
    let entry = self
        .client_data
        .entry(client_id)
        .or_insert(default_client_entry);

    // Calculate the tokens available to this client.
    entry.available_tokens = entry.available_tokens
        + (entry.last_request.elapsed().as_secs()) as f64
            * (self.max_allowed / self.window_duration);

    // Cap the token. If client has gathered 10 tokens, e.g  and the max allowed requests
    // in 10 seconds is just 3 then we cap the available tokens to 3 as well.
    if entry.available_tokens >= self.max_allowed {
        entry.available_tokens = self.max_allowed;
    }

    // The client has tokens available. The request can be passed.
    if entry.available_tokens >= 1.0 {
        // can make a request.
        (self.runnable)();
        entry.available_tokens -= 1.0;
        entry.last_request = Instant::now();
    } else {
        // The request could not be completed.
        Err(String::from("Max rate limit reached!"));
    }

    Ok(())
}
```

In order to prevent us from having to change the hashmap every second, we will just calculate the number of tokens that could've been gathered since the last time the function was run successfully. If the token count exceeds the max available request count, then the tokens are capped. Finally, if the available tokens exceed 1 then the request is allowed to pass through. Otherwise we return a string error.

## Testing: 

Let's write a very simple test for this code. Each client is represented by a client_id which is a `usize`.

```rust

#[cfg(test)]
mod test_ratelimiter {
    use crate::{function_to_execute, RateLimiter, RateLimiterResult};

    #[test]
    fn test_limits() {
        let mut ratelimiter = RateLimiter::<usize>::new(3.0, 10.0, runnable);
        for i in 0..10usize {
            match ratelimiter.run(1) {
                RateLimiterResult::Ok => {
                    println!("Request {} passed", i);
                }
                RateLimiterResult::Err(_) => {
                    println!("Request {} fell through", i);
                }
            };

            // Let us assume that every 1 second, this function runs and makes a request.
            std::thread::sleep(std::time::Duration::from_secs(1));
        }
    }
}
```

## Conclusion: 

This is a very basic example of a rate limiter. I had a lot of fun writing this tutorial, especially, setting and executing a function as a structure member. The granularity of this rate limiter can be controlled using `as_millis()` instead of `as_secs()` in the code to replace seconds with milliseconds.
# axum-rust-nutshell

![alt text](assets/images/image-1.png)

<div style="position:relative;width:100%;padding-bottom:56.25%;height:0;">
  <iframe
    src="https://www.youtube.com/embed/FDWKlJmHv6k?si=KBQONqFJJottdmIE"
    title="YouTube video player"
    style="position:absolute;top:0;left:0;width:100%;height:100%;border:0;"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    referrerpolicy="strict-origin-when-cross-origin"
    allowfullscreen>
  </iframe>
</div>

# Creating an Axum Web Server in Rust

## Introduction to Axum

**Axum** is a web framework in Rust specifically designed for creating web servers with a focus on **ergonomics** and **modularity**. 

* **Prerequisites:** Axum installed and a new Rust project initialized.

---

## Setting Up the Router

The router is the core of the Axum web server, mapping URL paths and HTTP methods to specific handler functions.

### The `create_app` Function

The `create_app` function is responsible for setting up and returning the router.

* It returns a `Router` type, which is specifically the `Router` struct in Axum.
* You initialize it by calling `Router::new()`.

```rust
fn create_app() -> Router {
    Router::new()
        .route("/health", get(health_check))
}
```

### Chaining Route Calls

You chain `route` calls to define the API endpoints. Each route requires:

1. A **URL Path** (e.g., `/health`)
2. An **HTTP Method** (e.g., `get`, `post`, `patch`, `delete`)
3. A **Handler Function** (e.g., `health_check`) that processes the request. Note: The handler function is passed directly into the method function, not executed.

---

## Setting Up the Async Runtime and TCP Listener

To run the Axum web server, you need an asynchronous runtime and a TCP Listener.

### The Tokio Async Runtime

**Tokio** is the most popular async runtime in the Rust ecosystem. Because you need to await the creation of a TCP listener, the main function must be marked as `async`.
* **The `#[tokio::main]` macro:** Transforms the `async fn main` into a regular main function and sets up all the necessary Tokio runtime components behind the scenes so manual setup is not required. It allows the use of `async`/`await` everywhere.

### Binding the TCP Listener and Serving the App

1. **Bind the Listener:** Bind a `TcpListener` on port 3000 across all interfaces (`0.0.0.0`).
2. **Serve the Application:** Use `axum::serve`, which connects the TCP listener to the app (the router) and handles requests until the server is shut down.

```rust
#[tokio::main]
async fn main() {
    // Bind the TCP listener
    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000")
        .await
        .expect("failed to bind TCP listener");

    println!("server running on localhost:3000");

    // Initialize the router
    let app = create_app();

    // Start the Axum web server
    axum::serve(listener, app)
        .await
        .expect("failed to start server");
}
```

---

## Centralized Error Handling

Centralizing API errors into a single enum keeps handlers clean and makes error handling consistent across the application. Each variant of the enum represents a specific class of error exposed from handlers.

### Defining the `ApiError` Enum

An **enum** is a custom type that can be one of several possible values (called **variants**). 

* **`#[derive(Debug)]` trait:** Allows the errors to be printed for debugging purposes.

```rust
#[derive(Debug)]
enum ApiError {
    NotFound,
    InvalidInput { data: String },
    InternalError,
}
```

**Variants:**

* **`NotFound`:** Used when a resource does not exist (HTTP 404 error).
* **`InvalidInput`:** A special variant representing a Bad Request (HTTP 400). It carries data with it—specifically, a string containing an error message.
* **`InternalError`:** Used for an internal server error (HTTP 500).

### Implementing the `IntoResponse` Trait

Axum uses the `IntoResponse` trait to turn return values into actual HTTP responses. By implementing this trait for `ApiError`, any handler that returns `Result<T, ApiError>` automatically knows how to transform the error into a proper HTTP response.

* A **trait** is similar to an interface in other languages. In Rust, you can implement traits and methods for enums.

```rust
impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let (status, error_message) = match self {
            ApiError::NotFound => (StatusCode::NOT_FOUND, "data not found".to_string()),
            ApiError::InvalidInput { data: msg } => (StatusCode::BAD_REQUEST, msg),
            ApiError::InternalError => (StatusCode::INTERNAL_SERVER_ERROR, "internal server error".to_string()),
        };

        let body = axum::Json(serde_json::json!({
            "error": error_message
        }));

        (status, body).into_response()
    }
}
```

**Key Concepts in `IntoResponse` Implementation:**

* **`match self`:** Similar to a simple switch statement, but `self` refers to the specific instance of the enum being called. It differentiates return values for each individual variant.
* **Pattern Destructuring:** Seen in the `InvalidInput { data: msg }` arm. This syntax pulls out the string value stored inside the variant and binds it to the variable `msg` so it can be used as a return value.
* **Axum JSON Extractor:** Wrapping the body in `axum::Json` automatically sets the `Content-Type: application/json` header and serializes the web data into JSON format.

---

## Application Endpoints

### 1. Health Check Endpoint (`/health`)

A tiny endpoint to verify if the server is up and running.

* **Return Type:** `impl IntoResponse`. This means anything that can be turned into a response will be returned.
* **Response Body:** Returns JSON utilizing the `json!` macro containing a status (`"ok"`) and a message (`"server is running"`).

### 2. List Users Endpoint (`/users`)

Designed to demonstrate error handling by simulating a failure.

* **Return Type:** `Result<Json<Value>, ApiError>`. 
    * `Result` is a standard Rust enum for operations that can either succeed (`Ok` containing the success value) or fail (`Err` containing the error value).
* **Logic:** Immediately returns `Err(ApiError::InternalError)`. Axum catches this and seamlessly uses the `IntoResponse` implementation to generate the standard JSON response.

### 3. Get User Endpoint (`/users/:id`)

Demonstrates extracting dynamic path parameters from the URL.

* **Dynamic Path Parameter (`:id`):** Axum automatically captures the value in that part of the URL and makes it available to the handler via the **Path extractor**.

```rust
async fn get_user(Path(id): Path<u32>) -> Result<Json<Value>, ApiError> {
    if id > 100 {
        return Err(ApiError::NotFound);
    }
    
    Ok(Json(serde_json::json!({
        "id": id,
        "name": "user"
    })))
}
```

**Path Extraction Concepts:**

* **`Path<u32>`:** The path extractor pulls the ID from the URL and parses it into a `u32` integer.
* **Destructuring Syntax (`Path(id)`):** This specific syntax extracts the inner ID directly for use in the function.
* **Edge Case:** If the value cannot be parsed (e.g., passing letters instead of a number), Axum automatically rejects the request with a 400 Bad Request error.

---

## Testing the Web Server

Testing ensures the router and error handling work properly by spinning up the router in memory and sending synthetic requests through it.

### Test Setup

* **`mod tests` module:** Define a module for testing.
* **`#[cfg(test)]` macro:** Ensures the tests are only compiled when running the test suite.
* **`use super::*;`:** Brings all items from the parent module (the main code) into the scope of the test module.
* **`#[tokio::test]` macro:** Used instead of a standard test macro because the tests utilize the Tokio async runtime.

### Testing the Health Check Endpoint

1. **Construct a Request:** Use `Request::builder()` to construct a GET request to `/health`. Use `Body::empty()` since a GET request has no body. The HTTP method defaults to GET automatically.
2. **Send Request In-Memory:** Use the `oneshot` function on the app router: `app.oneshot(request).await.unwrap()`.
    * **Note on `oneshot`:** This function comes from the `tower` service extension trait. It allows sending a single request directly to the app's router and getting a response back entirely in memory. It is much more reliable and faster for testing than starting a real web server and sending requests over a network, as there is zero network overhead.
3. **Assertions:**
    * **Status Code:** Assert that the response status equals `StatusCode::OK` (200).
    * **Parse Body:** Read the full response body into memory using `response.into_body().collect().await.unwrap()`. Convert it to a bytes object using `.to_bytes()`.
    * **Validate JSON:** Parse the bytes into a structured JSON value using `serde_json::from_slice(&body).unwrap()`. A reference is used because `from_slice` does not want to own the value. Verify that the JSON contains `status: "ok"` and `message: "server is running"`.

### Testing the `ApiError` `IntoResponse` Implementation

To verify that custom errors map to the correct HTTP status codes, use a standard vector of test cases.

1. Define a vector where each individual element is a tuple containing:
    * The specific `ApiError` variant.
    * The expected HTTP `StatusCode`.
2. Iterate over the vector using a `for` loop.
3. For each case, convert the error into a response manually by calling `.into_response()`.
4. Assert that the resulting `response.status()` equals the expected status code.
5. Run tests using the `cargo test` command.

---

## Deployment and Hosting Note (Savala)

*Note: This information pertains to a sponsored platform discussed for deploying the web server.*

**Savala** is an all-in-one Platform as a Service (PaaS) for deploying apps, databases, and static sites without dealing with complex infrastructure.

* **Features & Limits:** No artificial limits, no restrictions on parallel builds, and no caps on team members. You only pay for what you use.
* **Infrastructure:** Runs on Google Kubernetes Engine across 25 regions, utilizing Cloudflare's global network for speed and reliability.
* **Support:** Provides support from real human developers who understand technical problems.
* **Developer Experience:** Seamless workflow allowing you to push to Git, receive automatic builds, enjoy instant previews, and benefit from one-click templates. Scales with the user from side projects to large applications.
* **Promotion:** Offers new users $50 in free credits to get started.
# Rust grpc helloworld

- [Init the project](#init-the-project)
- [Write protocol file](#write-protocol-file)
- [Write build.rs](#write-buildrs)
- [Write helloworld grpc server](#write-helloworld-grpc-server)
- [Write helloworld grpc client](#write-helloworld-grpc-client)
- [Run helloworld grpc server](#run-helloworld-grpc-server)
- [Run helloworld grpc client](#run-helloworld-grpc-client)
- [Generated helloworld.rs](#generated-helloworldrs)

In this tutorial, we'll walk you through how to setup a gRPC server in Rust using `tonic` crate.

# Init the project

First, we'll create a new project using `cargo init <project_name>`.

```bash
cargo init tonic_example
```

Let's see the structure of the directory.

```bash
❯ tree
.
├── Cargo.toml
└── src
    └── main.rs

2 directories, 2 files

~/tmp/tonic_example master*

❯ ct Cargo.toml
[package]
name = "tonic_example"
version = "0.1.0"
edition = "2021"

[dependencies]
```

As we'll build a grpc service, we can use `tonic` crate, which is a Rust implementation of gRPC.

# Write protocol file

We'll create a simple greeter service. We'll put proto file in `proto` directory.

```bash
mkdir proto
touch proto/helloworld.proto
```

Here's the content of the `helloworld.proto` file, in which we define `HelloRequest` type, `HelloReply` type and `Greeter` service.

```proto
syntax = "proto3";
package helloworld;

service Greeter {
    // Our SayHello rpc accepts HelloRequests and returns HelloReplies
    rpc SayHello (HelloRequest) returns (HelloReply);
}

message HelloRequest {
    // Request message contains the name to be greeted
    string name = 1;
}

message HelloReply {
    // Reply contains the greeting message
    string message = 1;
}
```

Here's an explanation of the code:

- `syntax = "proto3";`: This line indicates that you are using version 3 of the protobuf language.
- `package helloworld;`: This line defines the package name for the service. It helps to prevent name clashes between protobuf messages.

The service definition starts with this line: `service Greeter {`. Here are the key points:

- `Greeter`: This is the name of the service (essentially an API) you are defining.
- The service `Greeter` has a single method `SayHello` which is defined as:`rpc SayHello (HelloRequest) returns (HelloReply);`
  - `SayHello`: This is the name of the function that will be exposed to clients on the gRPC server.
  - `(HelloRequest)`: This denotes the input parameters of the method. It takes in a single parameter of the type `HelloRequest`.
  - `returns (HelloReply)`: This shows that the function returns a `HelloReply` message.

The protocol buffer message types are defined with this code:

- `.message HelloRequest`: The `HelloRequest` message has a single field name of type `string`. The `= 1;` bit is a unique number used to identify the field in the message binary format.
- `.message HelloReply`: The `HelloReply` message also has a single field message also of type `string`.

In a nutshell, you have defined a `Greeter` service that has a `SayHello` method expecting a `HelloRequest` that contains a name and returns a `HelloReply` containing the message. It's analogous to defining a REST API endpoint but in the gRPC and protocol buffers context.

# Write build.rs

In order to build protobuf file, we need to install the `protoc` protocol buffers compiler. On MacOS, we can install by using this command:

```bash
brew install protobuf
```

# Write helloworld grpc server

Now, let's write server side code.

Create a file `helloworld-server.rs` in `src/bin` directory: `touch src/bin/helloworld-server.rs`.

```rust
use tonic::{transport::Server, Request, Response, Status};

use hello_world::{
    greeter_server::{Greeter, GreeterServer},
    HelloReply, HelloRequest,
};

pub mod hello_world {
    tonic::include_proto!("helloworld");
}

#[derive(Debug, Default)]
pub struct MyGreeter {}

#[tonic::async_trait]
impl Greeter for MyGreeter {
    async fn say_hello(
        &self, request: Request<HelloRequest>,
    ) -> Result<Response<HelloReply>, Status> {
        // println!("Got a request: {:?}", request);

        let reply = hello_world::HelloReply {
            message: format!("Hello {}!", request.into_inner().name).into(),
        };

        Ok(Response::new(reply))
    }
}

#[tokio::main]
// #[tokio::main(core_threads = 16, max_threads = 32)]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // NOTE: This works!
    let addr = "[::1]:50051".parse()?;
    // NOTE❌: This does NOT works! ConnectionRefused error
    // let addr = "0.0.0.0:50051".parse()?;
    let greeter = MyGreeter::default();

    Server::builder()
        .add_service(GreeterServer::new(greeter))
        .serve(addr)
        .await?;

    Ok(())
}
```

# Write helloworld grpc client

Now, let's write client side code.

Create a file `helloworld-client.rs` in `src/bin` directory: `touch src/bin/helloworld-client.rs`.

```rust
use hello_world::greeter_client::GreeterClient;
use hello_world::HelloRequest;

pub mod hello_world {
    tonic::include_proto!("helloworld");
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut client = GreeterClient::connect("http://[::1]:50051").await?;

    let request = tonic::Request::new(HelloRequest { name: "Tonic".into() });

    let response = client.say_hello(request).await?;

    println!("RESPONSE={:?}", response);

    Ok(())
}
```

# Run helloworld grpc server

Now, let's run the grpc server.

```bash
cargo run --bin helloworld-server
```

# Run helloworld grpc client

While the server is up and running, we can run the grpc client to send request to the server.

```bash
cargo run --bin helloworld-client
```

Output:

```bash
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.11s
     Running `target/debug/helloworld-client`
RESPONSE=Response {
    metadata: MetadataMap {
        headers: {
            "content-type": "application/grpc",
            "date": "Wed, 03 Apr 2024 17:56:21 GMT",
            "grpc-status": "0",
        },
    },
    message: HelloReply {
        message: "Hello Tonic!",
    },
    extensions: Extensions,
}
```

# Generated helloworld.rs

If you're interested in what the generated file looks like, you can refer to `helloworld.rs` file which is located in `target/debug/build/tonic_example-4094918d1c86be5c/out` directory.

Below is the contents of the file.

```rust
#[allow(clippy::derive_partial_eq_without_eq)]
#[derive(Clone, PartialEq, ::prost::Message)]
pub struct HelloRequest {
    /// Request message contains the name to be greeted
    #[prost(string, tag = "1")]
    pub name: ::prost::alloc::string::String,
}
#[allow(clippy::derive_partial_eq_without_eq)]
#[derive(Clone, PartialEq, ::prost::Message)]
pub struct HelloReply {
    /// Reply contains the greeting message
    #[prost(string, tag = "1")]
    pub message: ::prost::alloc::string::String,
}
/// Generated client implementations.
pub mod greeter_client {
    #![allow(unused_variables, dead_code, missing_docs, clippy::let_unit_value)]
    use tonic::codegen::*;
    use tonic::codegen::http::Uri;
    #[derive(Debug, Clone)]
    pub struct GreeterClient<T> {
        inner: tonic::client::Grpc<T>,
    }
    impl GreeterClient<tonic::transport::Channel> {
        /// Attempt to create a new client by connecting to a given endpoint.
        pub async fn connect<D>(dst: D) -> Result<Self, tonic::transport::Error>
        where
            D: TryInto<tonic::transport::Endpoint>,
            D::Error: Into<StdError>,
        {
            let conn = tonic::transport::Endpoint::new(dst)?.connect().await?;
            Ok(Self::new(conn))
        }
    }
    impl<T> GreeterClient<T>
    where
        T: tonic::client::GrpcService<tonic::body::BoxBody>,
        T::Error: Into<StdError>,
        T::ResponseBody: Body<Data = Bytes> + Send + 'static,
        <T::ResponseBody as Body>::Error: Into<StdError> + Send,
    {
        pub fn new(inner: T) -> Self {
            let inner = tonic::client::Grpc::new(inner);
            Self { inner }
        }
        pub fn with_origin(inner: T, origin: Uri) -> Self {
            let inner = tonic::client::Grpc::with_origin(inner, origin);
            Self { inner }
        }
        pub fn with_interceptor<F>(
            inner: T,
            interceptor: F,
        ) -> GreeterClient<InterceptedService<T, F>>
        where
            F: tonic::service::Interceptor,
            T::ResponseBody: Default,
            T: tonic::codegen::Service<
                http::Request<tonic::body::BoxBody>,
                Response = http::Response<
                    <T as tonic::client::GrpcService<tonic::body::BoxBody>>::ResponseBody,
                >,
            >,
            <T as tonic::codegen::Service<
                http::Request<tonic::body::BoxBody>,
            >>::Error: Into<StdError> + Send + Sync,
        {
            GreeterClient::new(InterceptedService::new(inner, interceptor))
        }
        /// Compress requests with the given encoding.
        ///
        /// This requires the server to support it otherwise it might respond with an
        /// error.
        #[must_use]
        pub fn send_compressed(mut self, encoding: CompressionEncoding) -> Self {
            self.inner = self.inner.send_compressed(encoding);
            self
        }
        /// Enable decompressing responses.
        #[must_use]
        pub fn accept_compressed(mut self, encoding: CompressionEncoding) -> Self {
            self.inner = self.inner.accept_compressed(encoding);
            self
        }
        /// Limits the maximum size of a decoded message.
        ///
        /// Default: `4MB`
        #[must_use]
        pub fn max_decoding_message_size(mut self, limit: usize) -> Self {
            self.inner = self.inner.max_decoding_message_size(limit);
            self
        }
        /// Limits the maximum size of an encoded message.
        ///
        /// Default: `usize::MAX`
        #[must_use]
        pub fn max_encoding_message_size(mut self, limit: usize) -> Self {
            self.inner = self.inner.max_encoding_message_size(limit);
            self
        }
        /// Our SayHello rpc accepts HelloRequests and returns HelloReplies
        pub async fn say_hello(
            &mut self,
            request: impl tonic::IntoRequest<super::HelloRequest>,
        ) -> std::result::Result<tonic::Response<super::HelloReply>, tonic::Status> {
            self.inner
                .ready()
                .await
                .map_err(|e| {
                    tonic::Status::new(
                        tonic::Code::Unknown,
                        format!("Service was not ready: {}", e.into()),
                    )
                })?;
            let codec = tonic::codec::ProstCodec::default();
            let path = http::uri::PathAndQuery::from_static(
                "/helloworld.Greeter/SayHello",
            );
            let mut req = request.into_request();
            req.extensions_mut()
                .insert(GrpcMethod::new("helloworld.Greeter", "SayHello"));
            self.inner.unary(req, path, codec).await
        }
    }
}
/// Generated server implementations.
pub mod greeter_server {
    #![allow(unused_variables, dead_code, missing_docs, clippy::let_unit_value)]
    use tonic::codegen::*;
    /// Generated trait containing gRPC methods that should be implemented for use with GreeterServer.
    #[async_trait]
    pub trait Greeter: Send + Sync + 'static {
        /// Our SayHello rpc accepts HelloRequests and returns HelloReplies
        async fn say_hello(
            &self,
            request: tonic::Request<super::HelloRequest>,
        ) -> std::result::Result<tonic::Response<super::HelloReply>, tonic::Status>;
    }
    #[derive(Debug)]
    pub struct GreeterServer<T: Greeter> {
        inner: _Inner<T>,
        accept_compression_encodings: EnabledCompressionEncodings,
        send_compression_encodings: EnabledCompressionEncodings,
        max_decoding_message_size: Option<usize>,
        max_encoding_message_size: Option<usize>,
    }
    struct _Inner<T>(Arc<T>);
    impl<T: Greeter> GreeterServer<T> {
        pub fn new(inner: T) -> Self {
            Self::from_arc(Arc::new(inner))
        }
        pub fn from_arc(inner: Arc<T>) -> Self {
            let inner = _Inner(inner);
            Self {
                inner,
                accept_compression_encodings: Default::default(),
                send_compression_encodings: Default::default(),
                max_decoding_message_size: None,
                max_encoding_message_size: None,
            }
        }
        pub fn with_interceptor<F>(
            inner: T,
            interceptor: F,
        ) -> InterceptedService<Self, F>
        where
            F: tonic::service::Interceptor,
        {
            InterceptedService::new(Self::new(inner), interceptor)
        }
        /// Enable decompressing requests with the given encoding.
        #[must_use]
        pub fn accept_compressed(mut self, encoding: CompressionEncoding) -> Self {
            self.accept_compression_encodings.enable(encoding);
            self
        }
        /// Compress responses with the given encoding, if the client supports it.
        #[must_use]
        pub fn send_compressed(mut self, encoding: CompressionEncoding) -> Self {
            self.send_compression_encodings.enable(encoding);
            self
        }
        /// Limits the maximum size of a decoded message.
        ///
        /// Default: `4MB`
        #[must_use]
        pub fn max_decoding_message_size(mut self, limit: usize) -> Self {
            self.max_decoding_message_size = Some(limit);
            self
        }
        /// Limits the maximum size of an encoded message.
        ///
        /// Default: `usize::MAX`
        #[must_use]
        pub fn max_encoding_message_size(mut self, limit: usize) -> Self {
            self.max_encoding_message_size = Some(limit);
            self
        }
    }
    impl<T, B> tonic::codegen::Service<http::Request<B>> for GreeterServer<T>
    where
        T: Greeter,
        B: Body + Send + 'static,
        B::Error: Into<StdError> + Send + 'static,
    {
        type Response = http::Response<tonic::body::BoxBody>;
        type Error = std::convert::Infallible;
        type Future = BoxFuture<Self::Response, Self::Error>;
        fn poll_ready(
            &mut self,
            _cx: &mut Context<'_>,
        ) -> Poll<std::result::Result<(), Self::Error>> {
            Poll::Ready(Ok(()))
        }
        fn call(&mut self, req: http::Request<B>) -> Self::Future {
            let inner = self.inner.clone();
            match req.uri().path() {
                "/helloworld.Greeter/SayHello" => {
                    #[allow(non_camel_case_types)]
                    struct SayHelloSvc<T: Greeter>(pub Arc<T>);
                    impl<T: Greeter> tonic::server::UnaryService<super::HelloRequest>
                    for SayHelloSvc<T> {
                        type Response = super::HelloReply;
                        type Future = BoxFuture<
                            tonic::Response<Self::Response>,
                            tonic::Status,
                        >;
                        fn call(
                            &mut self,
                            request: tonic::Request<super::HelloRequest>,
                        ) -> Self::Future {
                            let inner = Arc::clone(&self.0);
                            let fut = async move {
                                <T as Greeter>::say_hello(&inner, request).await
                            };
                            Box::pin(fut)
                        }
                    }
                    let accept_compression_encodings = self.accept_compression_encodings;
                    let send_compression_encodings = self.send_compression_encodings;
                    let max_decoding_message_size = self.max_decoding_message_size;
                    let max_encoding_message_size = self.max_encoding_message_size;
                    let inner = self.inner.clone();
                    let fut = async move {
                        let inner = inner.0;
                        let method = SayHelloSvc(inner);
                        let codec = tonic::codec::ProstCodec::default();
                        let mut grpc = tonic::server::Grpc::new(codec)
                            .apply_compression_config(
                                accept_compression_encodings,
                                send_compression_encodings,
                            )
                            .apply_max_message_size_config(
                                max_decoding_message_size,
                                max_encoding_message_size,
                            );
                        let res = grpc.unary(method, req).await;
                        Ok(res)
                    };
                    Box::pin(fut)
                }
                _ => {
                    Box::pin(async move {
                        Ok(
                            http::Response::builder()
                                .status(200)
                                .header("grpc-status", "12")
                                .header("content-type", "application/grpc")
                                .body(empty_body())
                                .unwrap(),
                        )
                    })
                }
            }
        }
    }
    impl<T: Greeter> Clone for GreeterServer<T> {
        fn clone(&self) -> Self {
            let inner = self.inner.clone();
            Self {
                inner,
                accept_compression_encodings: self.accept_compression_encodings,
                send_compression_encodings: self.send_compression_encodings,
                max_decoding_message_size: self.max_decoding_message_size,
                max_encoding_message_size: self.max_encoding_message_size,
            }
        }
    }
    impl<T: Greeter> Clone for _Inner<T> {
        fn clone(&self) -> Self {
            Self(Arc::clone(&self.0))
        }
    }
    impl<T: std::fmt::Debug> std::fmt::Debug for _Inner<T> {
        fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
            write!(f, "{:?}", self.0)
        }
    }
    impl<T: Greeter> tonic::server::NamedService for GreeterServer<T> {
        const NAME: &'static str = "helloworld.Greeter";
    }
}
```

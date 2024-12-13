# Tokio Codec

- [Intro](#intro)
  - [EchoCodec](#echocodec)
  - [Echo using io::Copy](#echo-using-iocopy)
- [Stream Sink trait](#stream-sink-trait)

# Intro

今天来讲讲 `tokio` 的 `codec`。

顾名思义，`codec` 是一个编码解码器，用于将原始字节解码为 `rust` 的数据类型。

首先，我们来看看 `codec` 的基本用法。

## EchoCodec

我们首先来看一个简单的例子，我们将 `tokio` 中的 `TcpStream` 进行编码和解码。

实现 `Decoder` 和 `Encoder` 这两个 `trait` 即可拥有编解码的功能。

在实现这两个 `trait` 之前，我们首先定义错误类型，这里使用 `enum ConnectionError`，使用 `enum` 也是最常见的定义错误类型的方式。

为什么要先定义错误类型呢？因为 `Decoder` 和 `Encoder` 这两个 `trait` 都定义了一个叫做 `Error` 的关联类型（`associated type`），所以为了实现这两个 `trait`，我们也需要定义一个错误类型。

下面是 `tokio_util::codec` 这个包(`package`)里的 `Decoder` 和 `Encoder` 的定义。

`Decoder` `trait` 的定义:

```rust
/// Decoding of frames via buffers.
///
/// This trait is used when constructing an instance of [`Framed`] or
/// [`FramedRead`]. An implementation of `Decoder` takes a byte stream that has
/// already been buffered in `src` and decodes the data into a stream of
/// `Self::Item` frames.
///
/// Implementations are able to track state on `self`, which enables
/// implementing stateful streaming parsers. In many cases, though, this type
/// will simply be a unit struct (e.g. `struct HttpDecoder`).
///
/// For some underlying data-sources, namely files and FIFOs,
/// it's possible to temporarily read 0 bytes by reaching EOF.
///
/// In these cases `decode_eof` will be called until it signals
/// fullfillment of all closing frames by returning `Ok(None)`.
/// After that, repeated attempts to read from the [`Framed`] or [`FramedRead`]
/// will not invoke `decode` or `decode_eof` again, until data can be read
/// during a retry.
///
/// It is up to the Decoder to keep track of a restart after an EOF,
/// and to decide how to handle such an event by, for example,
/// allowing frames to cross EOF boundaries, re-emitting opening frames, or
/// resetting the entire internal state.
///
/// [`Framed`]: crate::codec::Framed
/// [`FramedRead`]: crate::codec::FramedRead
pub trait Decoder {
    /// The type of decoded frames.
    type Item;

    /// The type of unrecoverable frame decoding errors.
    ///
    /// If an individual message is ill-formed but can be ignored without
    /// interfering with the processing of future messages, it may be more
    /// useful to report the failure as an `Item`.
    ///
    /// `From<io::Error>` is required in the interest of making `Error` suitable
    /// for returning directly from a [`FramedRead`], and to enable the default
    /// implementation of `decode_eof` to yield an `io::Error` when the decoder
    /// fails to consume all available data.
    ///
    /// Note that implementors of this trait can simply indicate `type Error =
    /// io::Error` to use I/O errors as this type.
    ///
    /// [`FramedRead`]: crate::codec::FramedRead
    type Error: From<io::Error>;

    /// Attempts to decode a frame from the provided buffer of bytes.
    ///
    /// This method is called by [`FramedRead`] whenever bytes are ready to be
    /// parsed. The provided buffer of bytes is what's been read so far, and
    /// this instance of `Decode` can determine whether an entire frame is in
    /// the buffer and is ready to be returned.
    ///
    /// If an entire frame is available, then this instance will remove those
    /// bytes from the buffer provided and return them as a decoded
    /// frame. Note that removing bytes from the provided buffer doesn't always
    /// necessarily copy the bytes, so this should be an efficient operation in
    /// most circumstances.
    ///
    /// If the bytes look valid, but a frame isn't fully available yet, then
    /// `Ok(None)` is returned. This indicates to the [`Framed`] instance that
    /// it needs to read some more bytes before calling this method again.
    ///
    /// Note that the bytes provided may be empty. If a previous call to
    /// `decode` consumed all the bytes in the buffer then `decode` will be
    /// called again until it returns `Ok(None)`, indicating that more bytes need to
    /// be read.
    ///
    /// Finally, if the bytes in the buffer are malformed then an error is
    /// returned indicating why. This informs [`Framed`] that the stream is now
    /// corrupt and should be terminated.
    ///
    /// [`Framed`]: crate::codec::Framed
    /// [`FramedRead`]: crate::codec::FramedRead
    ///
    /// # Buffer management
    ///
    /// Before returning from the function, implementations should ensure that
    /// the buffer has appropriate capacity in anticipation of future calls to
    /// `decode`. Failing to do so leads to inefficiency.
    ///
    /// For example, if frames have a fixed length, or if the length of the
    /// current frame is known from a header, a possible buffer management
    /// strategy is:
    ///
    /// # use std::io;
    /// #
    /// # use bytes::BytesMut;
    /// # use tokio_util::codec::Decoder;
    /// #
    /// # struct MyCodec;
    /// #
    /// impl Decoder for MyCodec {
    ///     // ...
    ///     # type Item = BytesMut;
    ///     # type Error = io::Error;
    ///
    ///     fn decode(&mut self, src: &mut BytesMut) -> Result<Option<Self::Item>, Self::Error> {
    ///         // ...
    ///
    ///         // Reserve enough to complete decoding of the current frame.
    ///         let current_frame_len: usize = 1000; // Example.
    ///         // And to start decoding the next frame.
    ///         let next_frame_header_len: usize = 10; // Example.
    ///         src.reserve(current_frame_len + next_frame_header_len);
    ///
    ///         return Ok(None);
    ///     }
    /// }
    ///
    /// An optimal buffer management strategy minimizes reallocations and
    /// over-allocations.
    fn decode(&mut self, src: &mut BytesMut) -> Result<Option<Self::Item>, Self::Error>;

    /// A default method available to be called when there are no more bytes
    /// available to be read from the underlying I/O.
    ///
    /// This method defaults to calling `decode` and returns an error if
    /// `Ok(None)` is returned while there is unconsumed data in `buf`.
    /// Typically this doesn't need to be implemented unless the framing
    /// protocol differs near the end of the stream, or if you need to construct
    /// frames _across_ eof boundaries on sources that can be resumed.
    ///
    /// Note that the `buf` argument may be empty. If a previous call to
    /// `decode_eof` consumed all the bytes in the buffer, `decode_eof` will be
    /// called again until it returns `None`, indicating that there are no more
    /// frames to yield. This behavior enables returning finalization frames
    /// that may not be based on inbound data.
    ///
    /// Once `None` has been returned, `decode_eof` won't be called again until
    /// an attempt to resume the stream has been made, where the underlying stream
    /// actually returned more data.
    fn decode_eof(&mut self, buf: &mut BytesMut) -> Result<Option<Self::Item>, Self::Error> {
        match self.decode(buf)? {
            Some(frame) => Ok(Some(frame)),
            None => {
                if buf.is_empty() {
                    Ok(None)
                } else {
                    Err(io::Error::new(io::ErrorKind::Other, "bytes remaining on stream").into())
                }
            }
        }
    }

    /// Provides a [`Stream`] and [`Sink`] interface for reading and writing to this
    /// `Io` object, using `Decode` and `Encode` to read and write the raw data.
    ///
    /// Raw I/O objects work with byte sequences, but higher-level code usually
    /// wants to batch these into meaningful chunks, called "frames". This
    /// method layers framing on top of an I/O object, by using the `Codec`
    /// traits to handle encoding and decoding of messages frames. Note that
    /// the incoming and outgoing frame types may be distinct.
    ///
    /// This function returns a *single* object that is both `Stream` and
    /// `Sink`; grouping this into a single object is often useful for layering
    /// things like gzip or TLS, which require both read and write access to the
    /// underlying object.
    ///
    /// If you want to work more directly with the streams and sink, consider
    /// calling `split` on the [`Framed`] returned by this method, which will
    /// break them into separate objects, allowing them to interact more easily.
    ///
    /// [`Stream`]: futures_core::Stream
    /// [`Sink`]: futures_sink::Sink
    /// [`Framed`]: crate::codec::Framed
    fn framed<T: AsyncRead + AsyncWrite + Sized>(self, io: T) -> Framed<T, Self>
    where
        Self: Sized,
    {
        Framed::new(io, self)
    }
}
```

`Encoder` `trait` 的定义:

```rust
/// Trait of helper objects to write out messages as bytes, for use with
/// [`FramedWrite`].
///
/// [`FramedWrite`]: crate::codec::FramedWrite
pub trait Encoder<Item> {
    /// The type of encoding errors.
    ///
    /// [`FramedWrite`] requires `Encoder`s errors to implement `From<io::Error>`
    /// in the interest letting it return `Error`s directly.
    ///
    /// [`FramedWrite`]: crate::codec::FramedWrite
    type Error: From<io::Error>;

    /// Encodes a frame into the buffer provided.
    ///
    /// This method will encode `item` into the byte buffer provided by `dst`.
    /// The `dst` provided is an internal buffer of the [`FramedWrite`] instance and
    /// will be written out when possible.
    ///
    /// [`FramedWrite`]: crate::codec::FramedWrite
    fn encode(&mut self, item: Item, dst: &mut BytesMut) -> Result<(), Self::Error>;
}
```

由于还不知道未来会有几种类型的错误，我们先随意定义两个: `Disconnected` 和 `Io(io::IoError)`，分别代表 `网络连接出错（断开）`以及`读取 socket 时发生的 io 错误`，当然实际场景的错误更加复杂和多样。

```rust
// 因为 codecs 的 Encoder trait 有个 associate type ，所以需要 Error 定义
#[derive(Debug)]
pub enum ConnectionError {
    Io(io::Error),
    Disconnected,
}

impl From<io::Error> for ConnectionError {
    fn from(err: io::Error) -> Self {
        ConnectionError::Io(err)
    }
}
```

其次定义 `EchoCodec`，它实现了 `Decoder` 和 `Encoder` `trait`，其中 `Decoder` 和 `Encoder` 的 `associated type` 都是 `ConnectionError`。

实现 `From<io::Error>` 的原因是 `Encoder` 的关联类型的类型约束: `type Error: From<io::Error>`，即我们必须能够将 `io::Error` 转换为 `ConnectionError`。

接下来我们定义需要被编码的消息类型 `Message`，它是一个 `String` 类型，并为此实现 `Encoder` 和 `Decoder` `trait`。

被编码（`Encode`）的意思是，将 `Message` 类型转换为 `BytesMut`，然后写入到 `TcpStream` 中。
被解码（`Decode`）的意思是，从 `FramedRead` 中读取 `BytesMut`，然后解码为 `Message` 供应用程序使用。

```rust
use tokio::codec::{Decoder, Encoder};

type Message = String;

struct EchoCodec;

// 给 EchoCodec 实现 Encoder trait
impl Encoder<Message> for EchoCodec {
    type Error = ConnectionError;

    fn encode(
        &mut self, item: Message, dst: &mut BytesMut,
    ) -> Result<(), Self::Error> {
        // 将 Message 写入 dst
        dst.extend(item.as_bytes());
        Ok(())
    }
}

// 给 EchoCodec 实现 Decoder trait
impl Decoder for EchoCodec {
    type Item = Message;

    type Error = ConnectionError;

    fn decode(
        &mut self, src: &mut BytesMut,
    ) -> Result<Option<Self::Item>, Self::Error> {
        // 将 src 中的数据转换为 String
        if src.is_empty() {
            return Ok(None);
        }
        // 将 src 中的数据移除
        let data = src.split();
        let data = String::from_utf8_lossy(&data[..]).to_string();

        // 将 line 转换为 Message
        Ok(Some(data))
    }
}
```

上面可以看出，`encode` 方法就是将 `Message` 转换为 `bytes` 并写入 `BytesMut`（通过 `BytesMut` 的 `extend` 方法），而 `decode` 方法就是将 `BytesMut` 转换为 `Message`。

最后，在 `main` 函数里是这么使用的:

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // start listening on 50007
    let listener = TcpListener::bind("127.0.0.1:50007").await?;
    println!("echo server started!");

    loop {
        let (socket, addr) = listener.accept().await?;

        println!("accepted connection from: {}", addr);

        tokio::spawn(async move {
            let codec = EchoCodec {};
            let mut conn = codec.framed(socket);
            while let Some(message) = conn.next().await {
                if let Ok(message) = message {
                    println!("received: {:?}", message);
                    conn.send(message).await.unwrap();
                }
            }
        });
    }
}
```

值得注意的是，`codec` 的 `framed` 方法（`codec.framed(socket)`）将 `TcpStream` 转换为 `Framed<TcpStream, EchoCodec>`，这个 `Framed` 就是实现了 `tokio` 中的 `Stream` 和 `Sink` 这两个 `trait`，因而具有了接收（通过 `Stream`）和发送（通过 `Sink`）数据的功能，关于这两个 `trait`，后面会提到。

上述 `framed` 方法是 `Decoder` `trait` 的方法，它第一个参数是 `TcpStream`，第二个参数是 `EchoCodec`，这个 `EchoCodec` 就是我们定义的 `EchoCodec`。

`Decoder::framed` 方法的定义如下:

```rust
fn framed<T, U>(self, codec: U) -> Framed<T, U>
where
    T: AsyncRead + AsyncWrite,
    U: Decoder + Encoder<Item = Self::Item>,
{
    Framed::new(self, codec)
}
```

`Framed::new` 方法创建一个 `Framed` 实例，将 `TcpStream` 和 `EchoCodec` 保存在 `FramedImpl` 中。

`Framed` struct 的定义以及 `new` 方法的定义:

```rust
pin_project! {
    /// A unified [`Stream`] and [`Sink`] interface to an underlying I/O object, using
    /// the `Encoder` and `Decoder` traits to encode and decode frames.
    ///
    /// You can create a `Framed` instance by using the [`Decoder::framed`] adapter, or
    /// by using the `new` function seen below.
    /// [`Stream`]: futures_core::Stream
    /// [`Sink`]: futures_sink::Sink
    /// [`AsyncRead`]: tokio::io::AsyncRead
    /// [`Decoder::framed`]: crate::codec::Decoder::framed()
    pub struct Framed<T, U> {
        #[pin]
        inner: FramedImpl<T, U, RWFrames>
    }
}

impl<T, U> Framed<T, U>
where
    T: AsyncRead + AsyncWrite,
{
    /// Provides a [`Stream`] and [`Sink`] interface for reading and writing to this
    /// I/O object, using [`Decoder`] and [`Encoder`] to read and write the raw data.
    ///
    /// Raw I/O objects work with byte sequences, but higher-level code usually
    /// wants to batch these into meaningful chunks, called "frames". This
    /// method layers framing on top of an I/O object, by using the codec
    /// traits to handle encoding and decoding of messages frames. Note that
    /// the incoming and outgoing frame types may be distinct.
    ///
    /// This function returns a *single* object that is both [`Stream`] and
    /// [`Sink`]; grouping this into a single object is often useful for layering
    /// things like gzip or TLS, which require both read and write access to the
    /// underlying object.
    ///
    /// If you want to work more directly with the streams and sink, consider
    /// calling [`split`] on the `Framed` returned by this method, which will
    /// break them into separate objects, allowing them to interact more easily.
    ///
    /// Note that, for some byte sources, the stream can be resumed after an EOF
    /// by reading from it, even after it has returned `None`. Repeated attempts
    /// to do so, without new data available, continue to return `None` without
    /// creating more (closing) frames.
    ///
    /// [`Stream`]: futures_core::Stream
    /// [`Sink`]: futures_sink::Sink
    /// [`Decode`]: crate::codec::Decoder
    /// [`Encoder`]: crate::codec::Encoder
    /// [`split`]: https://docs.rs/futures/0.3/futures/stream/trait.StreamExt.html#method.split
    pub fn new(inner: T, codec: U) -> Framed<T, U> {
        Framed {
            inner: FramedImpl {
                inner,
                codec,
                state: Default::default(),
            },
        }
    }
}
```

`FramedImpl` 除了保存了 `TcpStream` 和 `EchoCodec`，它还保存了一个 `State`，这个 `State` 是一个 `RWFrames` 实例，相当于一个缓冲区。

`FramedImpl` struct 的定义:

```rust
pin_project! {
    #[derive(Debug)]
    pub(crate) struct FramedImpl<T, U, State> {
        #[pin]
        pub(crate) inner: T,
        pub(crate) state: State,
        pub(crate) codec: U,
    }
}
```

`RWFrames` 、`ReadFrame` 和 `WriteFrame` 的定义。

```rust
#[derive(Debug)]
pub(crate) struct ReadFrame {
    pub(crate) eof: bool,
    pub(crate) is_readable: bool,
    pub(crate) buffer: BytesMut,
    pub(crate) has_errored: bool,
}

pub(crate) struct WriteFrame {
    pub(crate) buffer: BytesMut,
}

#[derive(Default)]
pub(crate) struct RWFrames {
    pub(crate) read: ReadFrame,
    pub(crate) write: WriteFrame,
}
```

`RWFrames` 实现了 `Borrow` 和 `BorrowMut` 这两个 `trait`，能分别返回 `ReadFrame` 和 `WriteFrame` 用来作为读写数据的缓冲。

```rust
impl Borrow<ReadFrame> for RWFrames {
    fn borrow(&self) -> &ReadFrame {
        &self.read
    }
}
impl BorrowMut<ReadFrame> for RWFrames {
    fn borrow_mut(&mut self) -> &mut ReadFrame {
        &mut self.read
    }
}
impl Borrow<WriteFrame> for RWFrames {
    fn borrow(&self) -> &WriteFrame {
        &self.write
    }
}
impl BorrowMut<WriteFrame> for RWFrames {
    fn borrow_mut(&mut self) -> &mut WriteFrame {
        &mut self.write
    }
}
```

`RWFrames` 实现 `Borrow` `BorrowMut` 也比较有意思，当需要从 `Stream` `读` 数据（这里指异步读取 `AsyncRead`）的时候，会调用 `BorrowMut<ReadFrame>` 方法，返回内部的 `ReadFrame` 的引用，作为读数据的缓冲。当需要向 `Sink` `写` 数据（这里指异步写入 `AsyncWrite`）的时候，会调用 `BorrowMut<WriteFrame>` 方法，返回内部的 `WriteFrame` 的引用，作为写数据的缓冲。

而 `FramedImpl` 实现了 `Stream` 和 `Sink` 这两个 `trait`。`Stream` 代表读数据，`Sink` 代表写数据。实现 `Stream` 时，`FramedImpl` 的泛型参数的约束是 `T: AsyncRead` 和 `R: BorrowMut<ReadFrame>`，表示 `FramedImpl::inner` 只需满足 `AsyncRead`，而且读取操作时会用到 `ReadFrame`。

实现 `Sink` 时，`FramedImpl` 的泛型参数的约束是 `T: AsyncWrite` 和 `R: BorrowMut<WriteFrame>`。实现 `Sink` 时，表示 `FramedImpl::inner` 只需满足 `AsyncWrite`，而且写入操作时会用到 `WriteFrame`。

另外，比较有趣的是，`FramedImpl` 实现 `Stream` 时，`poll_next` 方法有个状态机，体现了读取数据流时复杂的流程。

```rust
impl<T, U, R> Stream for FramedImpl<T, U, R>
where
    T: AsyncRead,
    U: Decoder,
    R: BorrowMut<ReadFrame>,
{
    type Item = Result<U::Item, U::Error>;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>> {
        use crate::util::poll_read_buf;

        let mut pinned = self.project();
        let state: &mut ReadFrame = pinned.state.borrow_mut();
        // The following loops implements a state machine with each state corresponding
        // to a combination of the `is_readable` and `eof` flags. States persist across
        // loop entries and most state transitions occur with a return.
        //
        // The initial state is `reading`.
        //
        // | state   | eof   | is_readable | has_errored |
        // |---------|-------|-------------|-------------|
        // | reading | false | false       | false       |
        // | framing | false | true        | false       |
        // | pausing | true  | true        | false       |
        // | paused  | true  | false       | false       |
        // | errored | <any> | <any>       | true        |
        //                                                       `decode_eof` returns Err
        //                                          ┌────────────────────────────────────────────────────────┐
        //                   `decode_eof` returns   │                                                        │
        //                             `Ok(Some)`   │                                                        │
        //                                 ┌─────┐  │     `decode_eof` returns               After returning │
        //                Read 0 bytes     ├─────▼──┴┐    `Ok(None)`          ┌────────┐ ◄───┐ `None`    ┌───▼─────┐
        //               ┌────────────────►│ Pausing ├───────────────────────►│ Paused ├─┐   └───────────┤ Errored │
        //               │                 └─────────┘                        └─┬──▲───┘ │               └───▲───▲─┘
        // Pending read  │                                                      │  │     │                   │   │
        //     ┌──────┐  │            `decode` returns `Some`                   │  └─────┘                   │   │
        //     │      │  │                   ┌──────┐                           │  Pending                   │   │
        //     │ ┌────▼──┴─┐ Read n>0 bytes ┌┴──────▼─┐     read n>0 bytes      │  read                      │   │
        //     └─┤ Reading ├───────────────►│ Framing │◄────────────────────────┘                            │   │
        //       └──┬─▲────┘                └─────┬──┬┘                                                      │   │
        //          │ │                           │  │                 `decode` returns Err                  │   │
        //          │ └───decode` returns `None`──┘  └───────────────────────────────────────────────────────┘   │
        //          │                             read returns Err                                               │
        //          └────────────────────────────────────────────────────────────────────────────────────────────┘
        loop {
            // too long, omit
        }
    }
```

`FramedImpl` 实现了 `Stream`，我们就能够从它那里读取数据了。

读取数据的过程是通过 `StreamExt::next` 方法实现的，它是对 `Stream` `trait` 的扩展，提供了很多实用方法，其中 `next` 就是其中一个。

`StreamExt::next` 方法的定义:

```rust
/// An extension trait for `Stream`s that provides a variety of convenient
/// combinator functions.
pub trait StreamExt: Stream {
    /// Creates a future that resolves to the next item in the stream.
    ///
    /// Note that because `next` doesn't take ownership over the stream,
    /// the [`Stream`] type must be [`Unpin`]. If you want to use `next` with a
    /// [`!Unpin`](Unpin) stream, you'll first have to pin the stream. This can
    /// be done by boxing the stream using [`Box::pin`] or
    /// pinning it to the stack using the `pin_mut!` macro from the `pin_utils`
    /// crate.
    ///
    /// # Examples
    ///
    /// # futures::executor::block_on(async {
    /// use futures::stream::{self, StreamExt};
    ///
    /// let mut stream = stream::iter(1..=3);
    ///
    /// assert_eq!(stream.next().await, Some(1));
    /// assert_eq!(stream.next().await, Some(2));
    /// assert_eq!(stream.next().await, Some(3));
    /// assert_eq!(stream.next().await, None);
    /// # });
    fn next(&mut self) -> Next<'_, Self>
    where
        Self: Unpin,
    {
        assert_future::<Option<Self::Item>, _>(Next::new(self))
    }
    // other methods...
}
```

`StreamExt::next` 方法创建一个对自身的引用，并且返回一个 `Next` 对象，这个对象实现了 `Future` `trait`，所以我们可以通过 `await` 来读取数据。

`Next` struct 的定义:

```rust
/// Future for the [`next`](super::StreamExt::next) method.
#[derive(Debug)]
#[must_use = "futures do nothing unless you `.await` or poll them"]
pub struct Next<'a, St: ?Sized> {
    stream: &'a mut St,
}

impl<St: ?Sized + Unpin> Unpin for Next<'_, St> {}

impl<'a, St: ?Sized + Stream + Unpin> Next<'a, St> {
    pub(super) fn new(stream: &'a mut St) -> Self {
        Self { stream }
    }
}

impl<St: ?Sized + FusedStream + Unpin> FusedFuture for Next<'_, St> {
    fn is_terminated(&self) -> bool {
        self.stream.is_terminated()
    }
}

impl<St: ?Sized + Stream + Unpin> Future for Next<'_, St> {
    type Output = Option<St::Item>;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        self.stream.poll_next_unpin(cx)
    }
}
```

在得知 `FramedImpl` 如何读取数据之后，那么 `FramedImpl` 是如何实现向 `Sink` 写入数据的呢？

`FramedImpl` 实现了 `Sink` `trait`，可以看到主要是调用了 `FramedImpl::poll_flush` 方法将 `Encoder` 编码的数据通过字节流发送出去。

```rust
impl<T, I, U, W> Sink<I> for FramedImpl<T, U, W>
where
    T: AsyncWrite,
    U: Encoder<I>,
    U::Error: From<io::Error>,
    W: BorrowMut<WriteFrame>,
{
    type Error = U::Error;

    fn poll_ready(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        if self.state.borrow().buffer.len() >= BACKPRESSURE_BOUNDARY {
            self.as_mut().poll_flush(cx)
        } else {
            Poll::Ready(Ok(()))
        }
    }

    fn start_send(self: Pin<&mut Self>, item: I) -> Result<(), Self::Error> {
        let pinned = self.project();
        pinned
            .codec
            .encode(item, &mut pinned.state.borrow_mut().buffer)?;
        Ok(())
    }

    fn poll_flush(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        use crate::util::poll_write_buf;
        trace!("flushing framed transport");
        let mut pinned = self.project();

        while !pinned.state.borrow_mut().buffer.is_empty() {
            let WriteFrame { buffer } = pinned.state.borrow_mut();
            trace!("writing; remaining={}", buffer.len());

            let n = ready!(poll_write_buf(pinned.inner.as_mut(), cx, buffer))?;

            if n == 0 {
                return Poll::Ready(Err(io::Error::new(
                    io::ErrorKind::WriteZero,
                    "failed to \
                     write frame to transport",
                )
                .into()));
            }
        }

        // Try flushing the underlying IO
        ready!(pinned.inner.poll_flush(cx))?;

        trace!("framed transport flushed");
        Poll::Ready(Ok(()))
    }

    fn poll_close(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
        ready!(self.as_mut().poll_flush(cx))?;
        ready!(self.project().inner.poll_shutdown(cx))?;

        Poll::Ready(Ok(()))
    }
}
```

而我们能通过 `send` 方法（参看 `main` 函数中的 `conn.send(message).await.unwrap();`）将编码的 `Message` 发送出去，是因为 `SinkExt` 是对 `Sink` `trait` 的扩展，它提供了 `send` 方法。

```rust
impl<T: ?Sized, Item> SinkExt<Item> for T where T: Sink<Item> {}

/// An extension trait for `Sink`s that provides a variety of convenient
/// combinator functions.
pub trait SinkExt<Item>: Sink<Item> {
    /// A future that completes after the given item has been fully processed
    /// into the sink, including flushing.
    ///
    /// Note that, **because of the flushing requirement, it is usually better
    /// to batch together items to send via `feed` or `send_all`,
    /// rather than flushing between each item.**
    fn send(&mut self, item: Item) -> Send<'_, Self, Item>
    where
        Self: Unpin,
    {
        assert_future::<Result<(), Self::Error>, _>(Send::new(self, item))
    }

    // other methods...
}
```

这个方法返回一个 `Send` struct，它是对 `Feed` 的一个简单 wrapper，它的作用是将 `item` 发送出去，发送功能交给 `Feed::sink_pin_mut::poll_flush` 来实现。

```rust
/// Future for the [`send`](super::SinkExt::send) method.
#[derive(Debug)]
#[must_use = "futures do nothing unless you `.await` or poll them"]
pub struct Send<'a, Si: ?Sized, Item> {
    feed: Feed<'a, Si, Item>,
}

// Pinning is never projected to children
impl<Si: Unpin + ?Sized, Item> Unpin for Send<'_, Si, Item> {}

impl<'a, Si: Sink<Item> + Unpin + ?Sized, Item> Send<'a, Si, Item> {
    pub(super) fn new(sink: &'a mut Si, item: Item) -> Self {
        Self { feed: Feed::new(sink, item) }
    }
}

impl<Si: Sink<Item> + Unpin + ?Sized, Item> Future for Send<'_, Si, Item> {
    type Output = Result<(), Si::Error>;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = &mut *self;

        if this.feed.is_item_pending() {
            ready!(Pin::new(&mut this.feed).poll(cx))?;
            debug_assert!(!this.feed.is_item_pending());
        }

        // we're done sending the item, but want to block on flushing the
        // sink
        ready!(this.feed.sink_pin_mut().poll_flush(cx))?;

        Poll::Ready(Ok(()))
    }
}
```

这里是 `Feed` struct 的定义:

```rust
/// Future for the [`feed`](super::SinkExt::feed) method.
#[derive(Debug)]
#[must_use = "futures do nothing unless you `.await` or poll them"]
pub struct Feed<'a, Si: ?Sized, Item> {
    sink: &'a mut Si,
    item: Option<Item>,
}

// Pinning is never projected to children
impl<Si: Unpin + ?Sized, Item> Unpin for Feed<'_, Si, Item> {}

impl<'a, Si: Sink<Item> + Unpin + ?Sized, Item> Feed<'a, Si, Item> {
    pub(super) fn new(sink: &'a mut Si, item: Item) -> Self {
        Feed { sink, item: Some(item) }
    }

    pub(super) fn sink_pin_mut(&mut self) -> Pin<&mut Si> {
        Pin::new(self.sink)
    }

    pub(super) fn is_item_pending(&self) -> bool {
        self.item.is_some()
    }
}

impl<Si: Sink<Item> + Unpin + ?Sized, Item> Future for Feed<'_, Si, Item> {
    type Output = Result<(), Si::Error>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.get_mut();
        let mut sink = Pin::new(&mut this.sink);
        ready!(sink.as_mut().poll_ready(cx))?;
        let item = this.item.take().expect("polled Feed after completion");
        sink.as_mut().start_send(item)?;
        Poll::Ready(Ok(()))
    }
}
```

`Feed` 实现了 `Future` `trait`，其 `poll` 方法首先调用 `poll_ready` 方法，如果 `poll_ready` 返回 `Ready`，则调用 `start_send` 方法，将 `item` 发送出去，如果 `start_send` 返回 `Ready`，则返回 `Ready`，否则继续调用 `poll_ready` 方法。

`poll_ready` 存在的意义是对是否能够发送 `item` 做出判断，如果不能发送，则需要等待（`poll_ready` 返回 `Poll::Pending` 等待被唤醒，具体实现是通过调用 `cx.waker().wake_by_ref()` 将异步任务注册，等待下一次被调度，`poll_ready` 的文档说明了这个过程，见下面 👇），直到能够发送。举个例子，在 `FramedImpl` 实现 `Sink` `trait` 时，采用了底层缓冲区（`WriteFrame`）的方式来存储待发送的数据，如果缓冲区满了，则调用 `poll_flush` 方法，否则表示可以开始发送数据（调用 `start_send` 方法）。

`FramedImpl::poll_ready` 方法的实现如下：

```rust
/// Attempts to prepare the `Sink` to receive a value.
///
/// This method must be called and return `Poll::Ready(Ok(()))` prior to
/// each call to `start_send`.
///
/// This method returns `Poll::Ready` once the underlying sink is ready to
/// receive data. If this method returns `Poll::Pending`, the current task
/// is registered to be notified (via `cx.waker().wake_by_ref()`) when `poll_ready`
/// should be called again.
///
/// In most cases, if the sink encounters an error, the sink will
/// permanently be unable to receive items.
fn poll_ready(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>> {
    if self.state.borrow().buffer.len() >= BACKPRESSURE_BOUNDARY {
        self.as_mut().poll_flush(cx)
    } else {
        Poll::Ready(Ok(()))
    }
}
```

通过分析 `Feed` 的 `poll` 方法，我们得知数据最终是如何发送出去的了。

至于待发送的数据何时被编码，我们可以看到是在 `FramedImpl::start_send` 方法来做的。

`FramedImpl::start_end` 方法的实现如下：

```rust
fn start_send(self: Pin<&mut Self>, item: I) -> Result<(), Self::Error> {
    let pinned = self.project();
    pinned
        .codec
        .encode(item, &mut pinned.state.borrow_mut().buffer)?;
    Ok(())
}
```

所以，我们能通过 `next` 来从数据流中接收并解析成 `Message` 结构，然后又通过 `send` 方法来将接收到的数据发送出去了。

`next` 接收数据 `send` 发送数据:

```rust
while let Some(message) = conn.next().await {
    if let Ok(message) = message {
        println!("received: {:?}", message);
        conn.send(message).await.unwrap();
    }
}
```

总结一下就是:

- `SinkExt` 提供了 `send` 方法，用于将接收到的数据发送出去
- `SinkExt::send` 方法通过 `Send::new` 返回一个实现了 `Future` 的 `Send` struct
- `Send` 内部采用 `Feed` 实现，目的是防止重复发送（将带发送的 `Item` 放入 `Option`，在 `poll` 被调用前检查是否已经被编码发送出去，如果已经被编码并发送，则 `Option` 为 `None`），`Feed` 也实现了 `Future` `trait`
- `Send` 方法首先检查 `Feed` 的 `is_item_pending` 方法，如果 `Feed` 的 `item` 为 `None`，则表示 `Feed` 已经被编码并发送出去，如果 `Feed` 的 `item` 为 `Some`，则表示 `Feed` 还未被编码并发送出去，需要调用 `Feed` 的 `poll` 方法。
- `Feed::poll` 方法完成发送逻辑
  - 调用 `poll_ready` 判断是否可以发送，缓冲区 `BACKPRESSURE_BOUNDARY` 大小为 8k，满了则无法发送
  - 调用 `self.item.take` 将待发送的 `Item` 取出
  - 调用 `start_send` 对 `Item` 进行编码
- `Send` 最后调用 `poll_flush` （此时是 `FramedImpl::poll_flush`）刷新写缓冲区

客户端测试:

首先运行 server 端:

```rust
echo server started!
accepted connection from: 127.0.0.1:60105
received: "1\r\n"
received: "2\r\n"
received: "3\r\n"
received: "44\r\n"
received: "55\r\n"
received: "66\r\n"
received: "777\r\n"
received: "888\r\n"
received: "999\r\n"
```

其次使用 `telnet` 来连接 `server` 端，并输入数字，然后按回车键，这些数字会被转换成字符串，然后会被发送到 `server` 端。

客户端的连接：

```
telnet localhost 50007
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
1
1
2
2
3
3
44
44
55
55
66
66
777
777
888
888
999
999
```

可以看到，`server` 端接收到的数据是 `1\r\n`, `2\r\n`, `3\r\n`, `44\r\n`, `55\r\n`, `66\r\n`, `777\r\n`, `888\r\n`, `999\r\n`，成功接受到来自 `client` 端的数据。

## Echo using io::Copy

手动实现 `EchoCodec` 比较繁琐，为了方便，我们可以使用 `io::copy` 来实现 `EchoCodec` 的功能，它的实现如下：

首先，`socket.split()` 将 `socket` 分成两个部分，一个是接收数据（这个在 `tokio` 里叫做 `ReadHalf`），一个是发送数据（这个在 `tokio` 里叫做 `WriteHalf`）。`io::copy` 将接收数据（`ReadHalf`）的部分拷贝到发送数据（`WritHalf`）的部分，这样就实现了数据的双向传输。

```rust
// 使用 io::copy 自动拷贝数据，需要调用 tokio::io::split 分割成 reader 和 writer
let (mut rd, mut wr) = socket.split();
if io::copy(&mut rd, &mut wr).await.is_err() {
    eprintln!("failed to copy");
}
```

完整的实现如下：

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // start listening on 50007
    let listener = TcpListener::bind("127.0.0.1:50007").await?;
    println!("echo server started!");

    loop {
        let (mut socket, addr) = listener.accept().await?;

        println!("accepted connection from: {}", addr);

        tokio::spawn(async move {
            // 方法1:
            // 使用 io::copy 自动拷贝数据，需要调用 tokio::io::split 分割成 reader 和 writer
            let (mut rd, mut wr) = socket.split();
            if io::copy(&mut rd, &mut wr).await.is_err() {
                eprintln!("failed to copy");
            }
        });
    }
    Ok(())
}
```

同样的，我们用客户端来进行测试：

首先运行 server 端:

```rust
echo server started!
accepted connection from: 127.0.0.1:60205
received: "1\r\n"
received: "2\r\n"
received: "3\r\n"
received: "44\r\n"
received: "55\r\n"
received: "66\r\n"
received: "777\r\n"
received: "888\r\n"
received: "999\r\n"
```

其次使用 `telnet` 来连接 `server` 端，并输入数字，然后按回车键，这些数字会被转换成字符串，然后会被发送到 `server` 端。

客户端的连接：

```
telnet localhost 50007
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
1
1
2
2
3
3
44
44
55
55
66
66
777
777
888
888
999
999
```

可以看到，利用 `io::copy` 和手动实现 `EchoCodec` 的输出一致。

# Stream Sink trait

最后，附赠一下 `Stream` 和 `Sink` `trait` 的定义。

实现了 `tokio` 中的 `Stream` 和 `Sink` 就能从数据流（如 `TcpStream` 或 `File`）中获取数据，并且能够将数据写回到数据流中。

`Stream` `trait` 的定义:

```rust
/// A stream of values produced asynchronously.
///
/// If `Future<Output = T>` is an asynchronous version of `T`, then `Stream<Item
/// = T>` is an asynchronous version of `Iterator<Item = T>`. A stream
/// represents a sequence of value-producing events that occur asynchronously to
/// the caller.
///
/// The trait is modeled after `Future`, but allows `poll_next` to be called
/// even after a value has been produced, yielding `None` once the stream has
/// been fully exhausted.
#[must_use = "streams do nothing unless polled"]
pub trait Stream {
    /// Values yielded by the stream.
    type Item;

    /// Attempt to pull out the next value of this stream, registering the
    /// current task for wakeup if the value is not yet available, and returning
    /// `None` if the stream is exhausted.
    ///
    /// # Return value
    ///
    /// There are several possible return values, each indicating a distinct
    /// stream state:
    ///
    /// - `Poll::Pending` means that this stream's next value is not ready
    /// yet. Implementations will ensure that the current task will be notified
    /// when the next value may be ready.
    ///
    /// - `Poll::Ready(Some(val))` means that the stream has successfully
    /// produced a value, `val`, and may produce further values on subsequent
    /// `poll_next` calls.
    ///
    /// - `Poll::Ready(None)` means that the stream has terminated, and
    /// `poll_next` should not be invoked again.
    ///
    /// # Panics
    ///
    /// Once a stream has finished (returned `Ready(None)` from `poll_next`), calling its
    /// `poll_next` method again may panic, block forever, or cause other kinds of
    /// problems; the `Stream` trait places no requirements on the effects of
    /// such a call. However, as the `poll_next` method is not marked `unsafe`,
    /// Rust's usual rules apply: calls must never cause undefined behavior
    /// (memory corruption, incorrect use of `unsafe` functions, or the like),
    /// regardless of the stream's state.
    ///
    /// If this is difficult to guard against then the [`fuse`] adapter can be used
    /// to ensure that `poll_next` always returns `Ready(None)` in subsequent
    /// calls.
    ///
    /// [`fuse`]: https://docs.rs/futures/0.3/futures/stream/trait.StreamExt.html#method.fuse
    fn poll_next(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Option<Self::Item>>;

    /// Returns the bounds on the remaining length of the stream.
    ///
    /// Specifically, `size_hint()` returns a tuple where the first element
    /// is the lower bound, and the second element is the upper bound.
    ///
    /// The second half of the tuple that is returned is an [`Option`]`<`[`usize`]`>`.
    /// A [`None`] here means that either there is no known upper bound, or the
    /// upper bound is larger than [`usize`].
    ///
    /// # Implementation notes
    ///
    /// It is not enforced that a stream implementation yields the declared
    /// number of elements. A buggy stream may yield less than the lower bound
    /// or more than the upper bound of elements.
    ///
    /// `size_hint()` is primarily intended to be used for optimizations such as
    /// reserving space for the elements of the stream, but must not be
    /// trusted to e.g., omit bounds checks in unsafe code. An incorrect
    /// implementation of `size_hint()` should not lead to memory safety
    /// violations.
    ///
    /// That said, the implementation should provide a correct estimation,
    /// because otherwise it would be a violation of the trait's protocol.
    ///
    /// The default implementation returns `(0, `[`None`]`)` which is correct for any
    /// stream.
    #[inline]
    fn size_hint(&self) -> (usize, Option<usize>) {
        (0, None)
    }
}
```

`Sink` `trait` 的定义:

```rust
/// A `Sink` is a value into which other values can be sent, asynchronously.
///
/// Basic examples of sinks include the sending side of:
///
/// - Channels
/// - Sockets
/// - Pipes
///
/// In addition to such "primitive" sinks, it's typical to layer additional
/// functionality, such as buffering, on top of an existing sink.
///
/// Sending to a sink is "asynchronous" in the sense that the value may not be
/// sent in its entirety immediately. Instead, values are sent in a two-phase
/// way: first by initiating a send, and then by polling for completion. This
/// two-phase setup is analogous to buffered writing in synchronous code, where
/// writes often succeed immediately, but internally are buffered and are
/// *actually* written only upon flushing.
///
/// In addition, the `Sink` may be *full*, in which case it is not even possible
/// to start the sending process.
///
/// As with `Future` and `Stream`, the `Sink` trait is built from a few core
/// required methods, and a host of default methods for working in a
/// higher-level way. The `Sink::send_all` combinator is of particular
/// importance: you can use it to send an entire stream to a sink, which is
/// the simplest way to ultimately consume a stream.
#[must_use = "sinks do nothing unless polled"]
pub trait Sink<Item> {
    /// The type of value produced by the sink when an error occurs.
    type Error;

    /// Attempts to prepare the `Sink` to receive a value.
    ///
    /// This method must be called and return `Poll::Ready(Ok(()))` prior to
    /// each call to `start_send`.
    ///
    /// This method returns `Poll::Ready` once the underlying sink is ready to
    /// receive data. If this method returns `Poll::Pending`, the current task
    /// is registered to be notified (via `cx.waker().wake_by_ref()`) when `poll_ready`
    /// should be called again.
    ///
    /// In most cases, if the sink encounters an error, the sink will
    /// permanently be unable to receive items.
    fn poll_ready(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;

    /// Begin the process of sending a value to the sink.
    /// Each call to this function must be preceded by a successful call to
    /// `poll_ready` which returned `Poll::Ready(Ok(()))`.
    ///
    /// As the name suggests, this method only *begins* the process of sending
    /// the item. If the sink employs buffering, the item isn't fully processed
    /// until the buffer is fully flushed. Since sinks are designed to work with
    /// asynchronous I/O, the process of actually writing out the data to an
    /// underlying object takes place asynchronously. **You *must* use
    /// `poll_flush` or `poll_close` in order to guarantee completion of a
    /// send**.
    ///
    /// Implementations of `poll_ready` and `start_send` will usually involve
    /// flushing behind the scenes in order to make room for new messages.
    /// It is only necessary to call `poll_flush` if you need to guarantee that
    /// *all* of the items placed into the `Sink` have been sent.
    ///
    /// In most cases, if the sink encounters an error, the sink will
    /// permanently be unable to receive items.
    fn start_send(self: Pin<&mut Self>, item: Item) -> Result<(), Self::Error>;

    /// Flush any remaining output from this sink.
    ///
    /// Returns `Poll::Ready(Ok(()))` when no buffered items remain. If this
    /// value is returned then it is guaranteed that all previous values sent
    /// via `start_send` have been flushed.
    ///
    /// Returns `Poll::Pending` if there is more work left to do, in which
    /// case the current task is scheduled (via `cx.waker().wake_by_ref()`) to wake up when
    /// `poll_flush` should be called again.
    ///
    /// In most cases, if the sink encounters an error, the sink will
    /// permanently be unable to receive items.
    fn poll_flush(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;

    /// Flush any remaining output and close this sink, if necessary.
    ///
    /// Returns `Poll::Ready(Ok(()))` when no buffered items remain and the sink
    /// has been successfully closed.
    ///
    /// Returns `Poll::Pending` if there is more work left to do, in which
    /// case the current task is scheduled (via `cx.waker().wake_by_ref()`) to wake up when
    /// `poll_close` should be called again.
    ///
    /// If this function encounters an error, the sink should be considered to
    /// have failed permanently, and no more `Sink` methods should be called.
    fn poll_close(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Result<(), Self::Error>>;
}
```

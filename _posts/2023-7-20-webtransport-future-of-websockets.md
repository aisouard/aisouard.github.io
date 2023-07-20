---
layout: single
title: "Meet WebTransport: The Future of WebSockets"
excerpt: "Deep dive into how WebTransport can overcome the limitations of WebSockets and WebRTC, discussing its unique features like the use of datagrams, unidirectional and bidirectional streams, and the revolutionary QUIC protocol"
header:
  image: /assets/images/posts/webtransport-future-of-websockets/header.jpg
  teaser: /assets/images/posts/webtransport-future-of-websockets/header.jpg
  overlay_image: /assets/images/posts/webtransport-future-of-websockets/header.jpg
  overlay_filter: 0.5
  og_image: /assets/images/posts/webtransport-future-of-websockets/og.png
date: 2023-07-20
comments: true
toc: true
---

During many years, [WebSockets][websockets] have been the only way to transmit
arbitrary data in a bidirectional way between a client and a server. However,
we've encountered some limitations making them not suitable for some use cases.

For example, WebSockets are not multiplexed, so if you want to send multiple
messages at the same time, you need to implement your own multiplexing layer on
top of WebSockets.

Furthermore, WebSockets can fall short in scenarios demanding low latency, [such
as video gaming,][agar-is-hard] due to their reliance on TCP. As a protocol
designed for reliability rather than speed, TCP may not always be the optimal
choice for time-sensitive applications where a delay can drastically impact user
experience.

This has left [WebRTC][webrtc] as the primary alternative for transmitting fast
and unreliable data thanks to the [datachannels][datachannels] feature provided
by UDP. However, while it can facilitate high-speed data transfer, WebRTC
presents its own set of complexities and may not be the most feasible solution
in all contexts.

This is where [WebTransport][webtransport] steps in, offering a promising way to
address these challenges.

# What is WebTransport?

[WebTransport][webtransport] represents a significant step forward in the realm
of web communication. Developed as part of the [IETF (Internet Engineering Task
Force)][ietf] standard, WebTransport is designed to provide flexible, efficient,
and secure communication between browsers and servers.

This technology, already available in the latest version of [Chrome][chrome] and
[Firefox][firefox], addresses the limitations of WebSockets and WebRTC by
offering multiplexed and unordered delivery of messages, as well as low latency,
thanks to its use of [UDP][udp].

We know that the use of [UDP][udp] can be a double-edged sword. On the one hand,
it allows us to send data faster, but on the other hand, it can be unreliable,
due to a higher probability of packet loss than [TCP][tcp].

The main advantage of using UDP is that we don't need to establish a connection
before sending data. If you can picture TCP being a strong and solid network
cable, UDP would be a radio antenna.

Thanks to [QUIC][QUIC], a transport protocol built on top of UDP, we can have
the best of both worlds. [QUIC][QUIC] provides a reliable and secure connection,
taking care of retransmitting lost packets, guaranteeing the order of the
messages, and automatically encrypting the data for us.

That's right, we won't need to worry about UDP's issues. But as the protocol is
different, we'll need to use it in a very different way than we're used to with
WebSockets.

# Getting started with WebTransport

Enough theory, let's see how we can use that marvel in practice. As the
technology is still in its early stages at the time of writing this article, it
could be very difficult to create a WebTransport server from scratch.

The only implementation I've found is the [wtransport crate][wtransport] written
in [Rust][rust]. It could be a great opportunity to finally learn the Rust
language. But you have to keep in mind that the web browser will require the
WebTransport server to have a valid TLS certificate.

Unfortunately, self-signed certificates won't work that time, as you'll require
the browser to trust your certificate via command-line. Our easiest way to get
started would be using the demo WebTransport server available at
[https://webtransport.day](https://webtransport.day/).

## Establishing a connection

The first step is to establish a connection with the server. This process will
be very similar to the one we're used to with WebSockets:

```js
const url = 'https://echo.webtransport.day'
const transport = new WebTransport(url)
console.log('Initiating connection')

try {
  await transport.ready
} catch (error) {
  console.error('Connection failed', error)
  return
}

console.log('Connection ready')
```

As you can see, we have to wait for the `ready` promise to be resolved before
going any further. This is due to our WebTransport client sending a connection
request to the server through the UDP protocol. This is the same as setting up
a radio antenna and having to wait for a signal to be received.

<div id="1-webtransport-console" style="width: 100%; height: 200px; border: solid lightgray 1px; margin-bottom: 8px; overflow: scroll">
</div>
<button id="1-webtransport-connect">Connect</button>
<button id="1-webtransport-disconnect" disabled>Disconnect</button>

<script type="text/javascript">
(function() {
  var connectButton;
  var disconnectButton;
  var webtransportConsole;
  var transport;

  function webtransportLog(message) {
    var line = document.createElement('p');
    line.style.margin = '0';
    line.textContent = message;
    webtransportConsole.appendChild(line);
  }

  function onConnectClick() {
    connectButton.disabled = true;
    var url = 'https://echo.webtransport.day';
    transport = new WebTransport(url);
    webtransportLog('Connecting to https://echo.webtransport.day ...');

    transport.ready
      .then(() => {
        webtransportLog('Connected!');
        disconnectButton.disabled = false;
      })
      .catch((error) => {
        webtransportLog('Connection failed: ' + error);
      });

    transport.closed
      .then(() => {
        webtransportLog('Connection closed normally');
        disconnectButton.disabled = true;
        connectButton.disabled = false;
      })
      .catch((error) => {
        webtransportLog('Connection closed abruptly: ' + error);
        disconnectButton.disabled = true;
        connectButton.disabled = false;
      });
  }

  function onDisconnectClick() {
    webtransportLog('Disconnecting');
    transport.close();
  }

  window.addEventListener('load', function () {
    connectButton = document.getElementById('1-webtransport-connect');
    connectButton.addEventListener('click', onConnectClick);

    disconnectButton = document.getElementById('1-webtransport-disconnect');
    disconnectButton.addEventListener('click', onDisconnectClick);

    webtransportConsole = document.getElementById('1-webtransport-console');
  });
})();
</script>

Handling disconnection is also considered best practice and can be done by
listening waiting for the `closed` promise to be fullfilled:

```js
transport.closed
  .then(() => {
    console.log('Connection closed normally')
  })
  .catch((error) => {
    console.error('Connection closed abruptly', error)
  })
```

# Sending and receiving data

Now that we have a connection, we can start sending and receiving data. However,
this process is very different from what we're used to with WebSockets. You
would think that sending data would be as simple as calling a `send` method on
our `transport` object, but it's not that simple.

There are currently three possible ways to send data with WebTransport. Each
way have something in common: they use _streams_:

**Datagram streams**: they are similar to UDP packets, they are unordered and
can be lost. They are the fastest way to send data, but they are not
guaranteed to be received. Perfect use cases for datagrams would be sending
the position of a player in a video game or sending the current position of a 
user's mouse cursor.

**Unidirectional streams**: they are similar to WebSockets, they are ordered
and guaranteed to be received. Take note that they are unidirectional, they
should be used when you only need to send or receive data in one direction.
A good example would be a file transfer, where you only need to receive data
from the server to the browser.

**Bidirectional streams**: exactly like unidirectional streams, but they can
be used to send and receive data in both directions. They should be used when
you need to wait for a response from the server after sending a message, like
when you would like to authenticate a user.

## Using datagrams

Datagrams are the fastest way to send data, making the process way easier than
using unidirectional or bidirectional streams. We can send data by retrieving
the `datagrams` stream from our `transport` object:

```js
const stream = transport.datagrams
const writer = stream.writable.getWriter()
await writer.write(new TextEncoder().encode('Hello there!'))
```

That's right, as simple as that.

On every stream object, such as `datagrams`, you will find a `readable` and a
`writable` property, being each one an instance of a `ReadableStream` and a
`WritableStream` respectively.

You can then use the `getReader()` and `getWriter()` methods to receive and send
data respectively. In our example, we're sending a string, but you can send any
kind of data, such as a `Uint8Array` or a `Blob`.

However, we still need to receive incoming data, and that won't be as simple as
listening for a `message` event like we're used to with WebSockets. Instead, we
need to use the `read()` method from the `readable` property.

This method will return an object with two properties: one named `value`, which
will contain the data received, and a boolean named `done`, which will be `true`
when the stream is closed.

```js
async function receiveDatagrams(transport) {
  const stream = transport.datagrams
  const reader = stream.readable.getReader()
    
  while (true) {
    const { value, done } = await reader.read()
    
    if (done) {
      console.log('Datagram stream closed by the server')
      return
    }
    
    console.log('Received', value.length, 'bytes:', new TextDecoder().decode(value))
  }
}
```

Now this can start to be confusing.

This infinite loop will keep reading incoming data from the server until the
stream is closed. But wouldn't it be more convenient just checking for a
`message` event? Or call the blocking `read()` method only when some data is
available?

Well, there's a simple way to execute the function above:

```js
receiveDatagrams(transport)
```

That's right, we just need to call the function, ignoring the fact that it's
a `Promise`, that it could be executed only if we defined a `.then()` callback.

But in that case, it's really enough just calling the function without blocking
it with an `await` keyword. It will just be executed in parallel with the rest
of our code.

Feel free to see that by yourself, using the demo below:

<div id="2-webtransport-console" style="width: 100%; height: 200px; border: solid lightgray 1px; margin-bottom: 8px; overflow: scroll">
</div>
<div style="margin-bottom: 8px">
<button id="2-webtransport-connect">Connect</button>
<button id="2-webtransport-disconnect" disabled>Disconnect</button>
</div>
<div style="display: flex; margin-bottom: 8px">
<input id="2-webtransport-input" type="text" placeholder="Datagram..." style="margin: 0; width: 100%; border: solid lightgray 1px" disabled/>
<button id="2-webtransport-send" disabled>Send</button>
</div>

<script type="text/javascript">
(function() {
  var connectButton;
  var disconnectButton;
  var sendButton;
  var inputText;
  var webtransportConsole;
  var transport;

  function webtransportLog(message) {
    var line = document.createElement('p');
    line.style.margin = '0';
    line.textContent = message;
    webtransportConsole.appendChild(line);
  }

  function onConnectClick() {
    connectButton.disabled = true;
    var url = 'https://echo.webtransport.day';
    transport = new WebTransport(url);
    webtransportLog('Connecting to https://echo.webtransport.day ...');

    transport.ready
      .then(() => {
        webtransportLog('Connected!');
        disconnectButton.disabled = false;
        inputText.disabled = false;
        sendButton.disabled = false;
        receiveDatagrams();
      })
      .catch((error) => {
        webtransportLog('Connection failed: ' + error);
      });

    transport.closed
      .then(() => {
        webtransportLog('Connection closed normally');
        disconnectButton.disabled = true;
        connectButton.disabled = false;
        inputText.disabled = true;
        sendButton.disabled = true;
      })
      .catch((error) => {
        webtransportLog('Connection closed abruptly: ' + error);
        disconnectButton.disabled = true;
        connectButton.disabled = false;
        inputText.disabled = true;
        sendButton.disabled = true;
      });
  }

  function onDisconnectClick() {
    webtransportLog('Disconnecting');
    transport.close();
  }

  async function onSendClick() {
    const value = inputText.value;

    if (value.length < 1) {
      return;
    }

    inputText.value = '';
    const stream = transport.datagrams;
    const writer = stream.writable.getWriter();

    await writer.write(new TextEncoder().encode(value));
    webtransportLog('You: ' + value);
  }

  async function receiveDatagrams() {
    const stream = transport.datagrams;
    const reader = stream.readable.getReader();
    
    while (true) {
      const { value, done } = await reader.read();
    
      if (done) {
        webtransportLog('Datagram stream closed by the server');
        return;
      }
    
      webtransportLog('Server: ' + new TextDecoder().decode(value));
    }
  }

  window.addEventListener('load', function () {
    connectButton = document.getElementById('2-webtransport-connect');
    connectButton.addEventListener('click', onConnectClick);

    disconnectButton = document.getElementById('2-webtransport-disconnect');
    disconnectButton.addEventListener('click', onDisconnectClick);

    sendButton = document.getElementById('2-webtransport-send');
    sendButton.addEventListener('click', onSendClick);

    inputText = document.getElementById('2-webtransport-input');
    webtransportConsole = document.getElementById('2-webtransport-console');
  });
})();
</script>

## Using streams

Now that we know how to send and receive datagrams, let's see how to use
unidirectional and bidirectional streams.

First of all, it is important to understand that streams in WebTransport are
designed to not be left open forever. This stands in contrast to the persistent
connections of WebSockets or the constant opening and closing of HTTP polling.

Stream are more similar to a phone call, where you open a connection,
communicate, and then hang up when you're done. The great thing about streams is
that it can go both ways, meaning that you can also receive a phone call from
the server side.

### Creating an unidirectional stream

Let's start by creating a unidirectional stream. This is the simplest type of
stream, where only one side can send data.

```js
const stream = await transport.createUnidirectionalStream()
const writer = stream.writable.getWriter()
await writer.write(new TextEncoder().encode("Hello!"))
await writer.close()
```

Note the importance of waiting for the stream to be created before sending a
message. This is because we need to wait for the server to accept our request
to open it.

Also remember that you can only use the `writable` property of the stream to
send data, the `readable` property is only available on the server's side and
only if you've received that stream from the server instead of creating it by
yourself.

### Creating a bidirectional stream

This step is very similar to the previous one, just with a small difference:

```js
const stream = await transport.createBidirectionalStream()
const writer = stream.writable.getWriter()
await writer.write(new TextEncoder().encode("Hello!"))
```

We can also receive data from the server, with the same way we can receive a
datagram:

```js
async function receiveBidirectionalData(stream) {
  const reader = stream.readable.getReader()
  const writer = stream.writable.getWriter()
    
  while (true) {
    const { value, done } = await reader.read()
    
    if (done) {
      console.log('Datagram stream closed by the server')
      return
    }
    
    console.log('Received', value.length, 'bytes:', new TextDecoder().decode(value))
    await writer.write(new TextEncoder().encode("Message received!"))
  }
}
```

### Receiving incoming streams

Up until now we've only seen how to create streams, but what would happen if the
server wanted to send us a reliable message?

The answer is that we need to wait for the server to send us a stream, and then
we can use it to communicate with it. There's actually two special streams that
we can read from, in order to receive incoming streams:
`incomingBidirectionalStreams` and `incomingUnidirectionalStreams` attributes.

```js
async listenIncomingStreams() {
  const reader = transport.incomingBidirectionalStreams.getReader()

  while (true) {
    // value will be a bidirectional stream
    // like if we have used `createBidirectionalStream`
    const { value, done } = await reader?.read()

    if (done) {
      console.log('Incoming bidirectional streams closed by the server')
      return
    }

    const stream = value
    receiveBidirectionalData(stream)
  }
}
```

# Quick summary

In any case you might be confused about the proper use of WebTransport, here's a
small summary:

- Use [datagrams](#using-datagrams) for small messages that don't need to be
  reliable. These are similar to UDP packets, and are useful for things like
  real-time coordinates.
- Use [createUnidirectionalStream](#creating-an-unidirectional-stream) for 
  sending reliable messages to the server. Switch to
  [createBidirectionalStream](#creating-a-bidirectional-stream) whenever you
  need to receive a reply from it.
- Use [incomingUnidirectionalStreams and incomingBidirectionalStreams](#receiving-incoming-streams)
  to receive incoming streams from the server.
  This would prevent the requirement of polling the server for new messages, or
  leaving a stream open forever.

# Conclusion

WebTransport is a revolutionary technology that provides a powerful and flexible
framework for bidirectional communication between a client and a server.

It resolves the limitations of WebSockets and WebRTC, providing developers
a solution that is multiplexed, efficient, and suitable for low-latency
applications.

Getting started with that feature can be challenging due to its infancy and lack
of widespread implementation.

However, as it continues to mature and become more mainstream, it has the
potential to redefine the way we build real-time, high-performance web
applications.

[websockets]: https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API/Writing_WebSocket_client_applications
[webrtc]: https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API
[datachannels]: https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Using_data_channels
[webtransport]: https://webtransport.day/
[ietf]: https://datatracker.ietf.org/doc/draft-ietf-webtrans-overview/
[agar-is-hard]: https://news.ycombinator.com/item?id=13266692
[udp]: https://en.wikipedia.org/wiki/User_Datagram_Protocol
[tcp]: https://en.wikipedia.org/wiki/Transmission_Control_Protocol
[chrome]: https://blog.chromium.org/2021/11/chrome-97-webtransport-new-array-static.html
[firefox]: https://www.mozilla.org/en-US/firefox/114.0/releasenotes/
[rust]: https://www.rust-lang.org/
[QUIC]: https://www.chromium.org/quic
[HTTP/3]: https://tools.ietf.org/html/draft-vvv-webtransport-over-http3-02
[wtransport]: https://github.com/BiagioFesta/wtransport
---
layout: post
title: "Rust echo server"
date: 2014-03-24
categories: rust
---

The process of creation of an echo server in Rust is quite simple.
Rust comes with libraries such as `std::io::net` that provides TCP,
UDP sockets and IP Addresses. All we need to do is to include these
libraries.

{% highlight rust linenos %}


use std::io::{Acceptor,Listener};
use std::io::net::ip::{SocketAddr, Ipv4Addr};
use std::io::net::tcp::TcpListener;

fn main () {
    let address = SocketAddr {
        ip: Ipv4Addr(127, 0, 0, 1),
        port: 8080
    };

    let server = TcpListener::bind(address);
    let mut acceptor = server.listen();

    for mut stream in acceptor.incoming() {
        let msg = stream.read_to_end().unwrap();
        stream.write(msg);
    }
}
{% endhighlight %}

**loc 1.** Create an address using the `SocketAddr` struct and passing an
`Ipv4Addr` with 127.0.0.1 and port 8080.

**loc: 11.** Create a "server" with a `TcpListener` that will be binded to the
localhost address.

**loc: 12.** Start listening or in other words... "Start the server".

**loc: 14.** For each successful connection, iterate with a mutable stream.

**loc: 15.** Read from the stream and allocate it in `msg`.

**loc: 16.**  Write msg to the mutable stream.


## Play with it

Compile your code with:

`rustc echo_server.rs`

And run it:

`./echo_server`


In another terminal:


{% highlight bash %}
cat echo.rs | nc localhost 8080
==> (lots of code)

echo "hello" | nc localhost 8080
==> hello
{% endhighlight %}

I couln't figure out how to echo the message every time you press enter.
Can you ? Let me know.

[Source cod3](https://github.com/alan-andrade/Doodles/blob/master/rust/echo.rs)

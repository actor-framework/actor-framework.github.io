---
layout: post
title:  "Testing Brokers"
author: dominik
tags: Testing Brokers
---

A major benefit of actors is that they are very easy to test. No complex
mocking, no simulation, simply sending and receiving messages. With brokers
(actors doing IO) this is a bit different. Sending messages still works, of
course, but usually the important part to test is the IO itself. Does your
actor parse its input correctly? Does it close connections appropriately?  This
cannot be easily tested with message passing alone.

The most important aspect in unit testing is reproducibility. When writing a
test, you want to make sure to be in charge of each step individually. This is
generally not the case when doing IO, since this involves components of the
operating system, e.g., sockets and the TCP stack. Fortunately, brokers in CAF
do never talk to any component of the OS directly. This means we can trick
brokers into a fake environment where we are in charge of everything.

# Brokers

Before we talk about how to take control of IO, we first need to understand how
the machinery behind brokers works.

![brokers]({{ site.url }}/static/img/broker.png)

The UML diagram above shows the relations a brokers has with IO-related classes
in CAF. The `middleman` is a singleton in CAF that provides access to various
IO-related functionality. The three classes that are responsible for doing IO
are `multiplexer`, `scribe`, and `doorman`.

## Scribes

A scribe decouples a broker from the underlying networking APIs. To the broker,
it provides access to an output buffer and the scribe will create
`new_data_msg` messages for the broker whenever the scribe has received enough
data according to the read policy set by the broker (via `configure_read`).
Note that the user-facing API of the broker does not expose the scribe to the
programmer. All the broker needs to worry about is a `connection_handle`. This
handle simply identifies the corresponding scribe and forwards the operation.

For example, if you call `self->configure_read(hdl,
receive_policy::at_most(1024));` on a broker, it looks which scribe manages
`hdl` and forwards the second argument to this scribe.

It is also worth mentioning that the `new_data_msg` received by the broker will
always contain the same buffer. The scribe merely re-writes its content over
and over again.

## Doormen

A doorman, just like a scribe, decouples the broker from underlying networking
APIs. Whenever a new network connection has been established, the doorman
generates a `new_connection_msg` for the broker.

## Multiplexer

The multiplexer is an IO loop and a factory for scribes and doormen. If you
want to change which networking API CAF is using, this is the (abstract) class
you need to implement. It has no member functions in the UML diagram for
brevity, but here are the important ones we need to know:

```cpp
class multiplexer {
public:
  /// Assigns an unbound scribe identified by `hdl` to `ptr`.
  /// @warning Do not call from outside the multiplexer's event loop.
  virtual void assign_tcp_scribe(abstract_broker* ptr,
                                 connection_handle hdl) = 0;

  /// Assigns an unbound doorman identified by `hdl` to `ptr`.
  /// @warning Do not call from outside the multiplexer's event loop.
  virtual void assign_tcp_doorman(abstract_broker* ptr, accept_handle hdl) = 0;

  /// Simple wrapper for runnables
  struct runnable;

  using runnable_ptr = intrusive_ptr<runnable>;

  /// Runs the multiplexers event loop.
  virtual void run() = 0;

  /// Invokes @p fun in the multiplexer's event loop, calling `fun()`
  /// immediately when called from inside the event loop.
  /// @threadsafe
  template <class F>
  void dispatch(F fun);

  /// Invokes @p fun in the multiplexer's event loop, forcing
  /// execution to be delayed when called from inside the event loop.
  /// @threadsafe
  template <class F>
  void post(F fun);

  // ...
};
```

Calling `multiplexer::assign_tcp_scribe` will create a new scribe (for TCP-like
communication) and assign this scribe to the given broker. The function
`assign_tcp_doorman` does the same thing for doormen. If you have ever used
ASIO, `run`, `dispatch`, and `post` will remind you of ASIO's `io_service`. And
you are right. In fact, CAF's `asio_multiplexer` is simply implemented using an
`io_service`. The function `run` is called in a thread crated by the `middleman`
on startup. Whenever a broker receives a message from other actors, it calls
`post` to schedule handling the message for later by creating a `runnable` for
this task.

# Faking IO

For testing brokers without actually doing IO, CAF has a `multiplexer`
implementation that is solely meant for testing:

```cpp
class test_multiplexer : public multiplexer {
public:
  /// A buffer storing bytes.
  using buffer_type = std::vector<char>;

  /// Models pending data on the network, i.e., the network 
  /// input buffer usually managed by the operating system.
  buffer_type& virtual_network_buffer(connection_handle hdl);

  /// Returns the output buffer of the scribe identified by `hdl`.
  buffer_type& output_buffer(connection_handle hdl);

  /// Returns the input buffer of the scribe identified by `hdl`.
  buffer_type& input_buffer(connection_handle hdl);

  /// Returns `true` if this handle has been closed
  /// for reading, `false` otherwise.
  bool& stopped_reading(connection_handle hdl);

  /// Returns `true` if this handle has been closed
  /// for reading, `false` otherwise.
  bool& stopped_reading(accept_handle hdl);

  /// Stores `hdl` as a pending connection for `src`.
  void add_pending_connect(accept_handle src, connection_handle hdl);

  /// Accepts a pending connect on `hdl`.
  void accept_connection(accept_handle hdl);

  /// Reads data from the external input buffer until
  /// the configured read policy no longer allows receiving.
  void read_data(connection_handle hdl);

  /// Appends `buf` to the virtual network buffer of `hdl`
  /// and calls `read_data(hdl)` afterwards.
  void virtual_send(connection_handle hdl, const buffer_type& buf);

  /// Waits until a `runnable` is available and executes it.
  void exec_runnable();

  /// Returns `true` if a `runnable` was available, `false` otherwise.
  bool try_exec_runnable();

  /// Executes all pending `runnable` objects.
  void flush_runnables();

  // ...
};
```

The `test_multiplexer` has several member function that "fake" network events
and allow you to manipulate the buffers of scribes directly. In particular,
`virtual_send` allows you to fake incoming data on a `connection_handle`. This
will cause the scribe to generate one or more `new_data_msg` messages
(depending on the configured receive policy). Those messages are handled by the
broker immediately.

To simulate a remote connection, one needs to create a pending connection using
`add_pending_connect` and cause the corresponding doorman to accept it via
`accept_connection`.

Using the test multiplexer requires calling `set_middleman(new
network::test_multiplexer)` in `main`, *before* calling any IO-related function
in CAF. Before showing the test multiplexer in action, we first implement
implement a broker we want to test.

# Example Application

The application we are going to test is a simplistic HTTP server. We do not
want to bother with actually parsing HTTP, so we always only consider the first
line of a HTTP header and check if it is equal to `"GET / HTTP/1.1"`. If we
receive anything else, we will send a 404 as response.

However, we are going to deal with chunked input and we (pedantically) require
`"\n\r"` for line breaks. We are not going to generate HTTP dynamically.
Instead, we use the following constants:

```cpp
constexpr char http_valid_get[] = "GET / HTTP/1.1";

constexpr char http_get[] = "GET / HTTP/1.1\r\n"
                            "Host: localhost\r\n"
                            "Connection: close\r\n"
                            "Accept: text/plain\r\n"
                            "User-Agent: CAF/0.14\r\n"
                            "Accept-Language: en-US\r\n"
                            "\r\n";

constexpr char http_ok[] = "HTTP/1.1 200 OK\r\n"
                           "Content-Type: text/plain\r\n"
                           "Connection: close\r\n"
                           "Transfer-Encoding: chunked\r\n"
                           "\r\n"
                           "d\r\n"
                           "Hi there! :)\r\n"
                           "\r\n"
                           "0\r\n"
                           "\r\n"
                           "\r\n";

constexpr char http_error[] = "HTTP/1.1 404 Not Found\r\n"
                              "Connection: close\r\n"
                              "\r\n";

constexpr char newline[2] = {'\r', '\n'};
```

When receiving input chunks, we need to keep track of the state of our parser.
We will also store each received line in a vector of strings and make use of
CAF's stateful actors, as shown below.

```cpp
enum parser_state {
  receive_new_line,
  receive_continued_line,
  receive_second_newline_half
};

struct http_state {
  http_state(abstract_broker* self) : self_(self) {
    // nop
  }

  ~http_state() {
    aout(self_) << "http worker finished with exit reason: "
                << self_->planned_exit_reason()
                << endl;
  }

  std::vector<std::string> lines;
  parser_state ps = receive_new_line;
  abstract_broker* self_;
};

using http_broker = caf::experimental::stateful_actor<http_state, broker>;
```

To make bookkeeping easier, we use one broker (our server) that accepts
incoming connections and spawns a new HTTP worker per connection. The server
does not need any additional state and is quite simple.

```cpp
behavior server(broker* self) {
  aout(self) << "server up and running" << endl;
  return {
    [=](const new_connection_msg& ncm) {
      aout(self) << "fork on new connection" << endl;
      auto worker = self->fork(http_worker, ncm.handle);
    },
    others >> [=] {
      aout(self) << "unexpected: "
                 << to_string(self->current_message()) << endl;
    }
  };
}
```

And here is the implementation of our `http_worker` using the state from above.

```cpp
behavior http_worker(http_broker* self, connection_handle hdl) {
  // tell network backend to receive any number of bytes between 1 and 128
  self->configure_read(hdl, receive_policy::at_most(128));
  return {
    [=](const new_data_msg& msg) {
      assert(! msg.buf.empty());
      assert(msg.handle == hdl);
      // extract lines from received buffer
      auto& lines = self->state.lines;
      auto i = msg.buf.begin();
      auto e = msg.buf.end();
      // search position of first newline in data chunk
      auto nl = std::search(i, e, std::begin(newline), std::end(newline));
      // store whether we are continuing a previously started line
      auto append_to_last_line = self->state.ps == receive_continued_line;
      // check whether our last chunk ended between \r and \n
      if (self->state.ps == receive_second_newline_half) {
        if (msg.buf.front() == '\n') {
          // simply skip this character
          ++i;
        }
      }
      // read line by line from our data chunk
      do {
        if (append_to_last_line) {
          append_to_last_line = false;
          auto& back = lines.back();
          back.insert(back.end(), i, nl);
        } else {
          lines.emplace_back(i, nl);
        }
        // if our last search didn't found a newline, we're done
        if (nl != e) {
          // skip newline and seek the next one
          i = nl + sizeof(newline);
          nl = std::search(i, e, std::begin(newline), std::end(newline));
        }
      } while (nl != e);
      // store current state of our parser
      if (msg.buf.back() == '\r') {
        self->state.ps = receive_second_newline_half;
        self->state.lines.pop_back(); // drop '\r' from our last read line
      } else if (msg.buf.back() == '\n') {
        self->state.ps = receive_new_line; // we've got a clean cut
      } else {
        self->state.ps = receive_continued_line; // interrupted in the middle
      }
      // we don't need to check for completion in any intermediate state
      if (self->state.ps != receive_new_line)
        return;
      // we have received the HTTP header if we have an empty line at the end
      if (lines.size() > 1 && lines.back().empty()) {
        auto& out = self->wr_buf(hdl);
        // we only look at the first line in our example and reply with our
        // OK message if we receive exactly "GET / HTTP/1.1", otherwise
        // we send a 404 HTTP response
        if (lines.front() == http_valid_get)
          out.insert(out.end(), std::begin(http_ok), std::end(http_ok));
        else
          out.insert(out.end(), std::begin(http_error), std::end(http_error));
        // write data and close connection
        self->flush(hdl);
        self->quit();
      }
    },
    [=](const connection_closed_msg&) {
      self->quit();
    },
    others >> [=] {
      aout(self) << "unexpected: "
                 << to_string(self->current_message()) << endl;
    }
  };
}
```

Our HTTP worker receives chunks of 128 bytes. Once it detected the end of the
HTTP header (a blank line), it looks at the first line to see if it is equal to
`http_valid_get`. If so, it sends the OK message, otherwise it sends the 404.

A minimal application using our brokers that always tries to open port 8080 is
a three-liner:

```cpp
int main() {
  spawn_io_server(server, 8080);
  await_all_actors_done();
  shutdown();
}
```

# Testing the HTTP Broker

Without further ado, here is our complete unit test for the example
application.

```cpp
namespace {

class fixture {
public:
  fixture() {
    // note: the middleman will take ownership of mpx_, but using
    //       this pointer is safe at any point before calling `shutdown`
    mpx_ = new network::test_multiplexer;
    set_middleman(mpx_);
    // spawn the actor-under-test
    aut_ = spawn_io(server);
    // assign the acceptor handle to the AUT
    aut_ptr_ = static_cast<abstract_broker*>(actor_cast<abstract_actor*>(aut_));
    mpx_->assign_tcp_doorman(aut_ptr_, acceptor_);
    // "open" a new connection to our server
    mpx_->add_pending_connect(acceptor_, connection_);
    mpx_->assign_tcp_scribe(aut_ptr_, connection_);
    mpx_->accept_connection(acceptor_);
  }

  ~fixture() {
    anon_send_exit(aut_, exit_reason::kill);
    // run the exit message and other pending messages explicitly,
    // since we do not invoke any "IO" from this point on that would
    // trigger those messages implicitly
    mpx_->flush_runnables();
    await_all_actors_done();
    shutdown();
  }

  // helper class for a nice-and-easy "mock(...).expect(...)" syntax
  class mock_t {
  public:
    mock_t(fixture* thisptr) : this_(thisptr) {
      // nop
    }

    mock_t(const mock_t&) = default;

    mock_t& expect(const std::string& what) {
      auto& buf = this_->mpx_->output_buffer(this_->connection_);
      CAF_REQUIRE((buf.size() >= what.size()));
      CAF_REQUIRE((std::equal(buf.begin(), buf.begin() + what.size(),
                              what.begin()));
      buf.erase(buf.begin(), buf.begin() + what.size()));
      return *this;
    }

    fixture* this_;
  };

  // mocks some input for our AUT and allows to
  // check the output for this operation
  mock_t mock(const char* what) {
    std::vector<char> buf;
    for (char c = *what++; c != '\0'; c = *what++)
      buf.push_back(c);
    mpx_->virtual_send(connection_, std::move(buf));
    return {this};
  }

  actor aut_;
  abstract_broker* aut_ptr_;
  network::test_multiplexer* mpx_;
  accept_handle acceptor_ = accept_handle::from_int(1);
  connection_handle connection_ = connection_handle::from_int(1);
};

} // namespace <anonymous>

CAF_TEST_FIXTURE_SCOPE(http_tests, fixture)

CAF_TEST(valid_response) {
  // write a GET message and expect an OK message as result
  mock(http_get).expect(http_ok);
}

CAF_TEST(invalid_response) {
  // write a GET with invalid path and expect a 404 message as result
  mock("GET /kitten.gif HTTP/1.1\r\n\r\n").expect(http_error);
}

CAF_TEST_FIXTURE_SCOPE_END()
```

The constructor of our fixture spawns the server that we are going to test.
Since we need an actual pointer to the broker, we need to use `actor_cast` to
get a pointer and downcast it afterwards. Using the pointer, we can setup a
connection using the test multiplexer. These steps will create a
`new_connection_msg` for the server that spawns a `http_worker` in response.

With the `mock` member function, we create a simple API that allows us to
correlate inputs and outputs. The `mock` function is a simple wrapper around
`virtual_send` and `expect` compares what output a broker has written in its
`output_buffer` with the output we expect.

The test uses the unit test framework from CAF. Using a different test
framework, e.g., Boost.Test, is straightforward.  The complete source code can
be found [at
GitHub](https://github.com/actor-framework/actor-framework/blob/4a97de85c8d01f5de0f3ba314fb836f220f8de27/libcaf_io/test/http_broker.cpp).

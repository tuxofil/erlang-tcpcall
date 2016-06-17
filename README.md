# TCP-based Request-Reply Bridge for Erlang nodes.

## Summary

Provides an API to make Request-Reply and Request-only
(responseless) interactions between an Erlang nodes using the
TCP/IP network protocol.

The bridge does not any encoding/decoding of the payload
data and assumes request and reply are given as binaries.
This is done to reduce memory copies of a potentially big
Erlang terms between processes.

The bridge consists of two sides: client and server.
The server is listening for incoming TCP connections on a
configured port number. The client connects to the server.
A request can be send only from the client to the server.

The request from the client, once transferred through the
network to the server side, is relayed to a so called
receiver. The receiver can be defined as arbitrary Erlang
process which will receive special messages or as a functional
object, which will be applied in a new Erlang process each
time when new request arrives.

Communication between client and server implemented in such
way to not block until reply from the server will arrive.
You can use same tcpcall connection from many Erlang processes
simultaneously - all requests will be multiplexed on the
client side before sending them to the server. All server
replies will be demultiplexed and sent back to appropriate
caller processes. This is the thing which differs tcpcall
from ØMQ REQ-REP. Latter will fail when you try to send
two simultaneous requests to the socket, but tcpcall will not.

The client side part does automatic reconnect when TCP connection
closed from the another side. The time until client will
reconnect to the server all calls to tcpcall:call/3 and
tcpcall:cast/2 will return {error, not_connected}.

For more API details see examples below and a description
of the tcpcall Erlang module.

## Main features

* easy and efficient way to build RPC-like or send-only interactions
 between two Erlang nodes without need to bind the nodes
 into an Erlang cluster;
* does not any complex data processing, as it operates only
 with binaries (the user must implement payload encoding and
 deconding by himself).

## Example with message passing

On the server node:

```
{ok, Pid} = tcpcall:listen([{bind_port, 5000}, {receiver, self()}]),
...
receive
    {tcpcall_req, BridgePid, RequestRef, EncodedRequest} ->
        Request = binary_to_term(EncodedRequest),
        Reply = Request * 2,
        EncodedReply = term_to_binary(Reply),
        ok = tcpcall:reply(BridgePid, RequestRef, EncodedReply);
    {tcpcall_cast, BridgePid, EncodedRequest} ->
        Request = binary_to_term(EncodedRequest),
        %% do something with request
        ...
```

On the client node:

```
{ok, Pid} = tcpcall:connect([{host, "server.com"}, {port, 5000}]),
EncodedRequest = term_to_binary(5),
{ok, EncodedReply} = tcpcall:call(Pid, EncodedRequest, 1000),
10 = binary_to_term(EncodedReply),

EncodedCast = term_to_binary({my_sophisticated_cast, 5, ["abc", make_ref()]}),
ok = tcpcall:cast(Pid, EncodedCast),
...
```

## Example with callback function

On the server node:

```
{ok, Pid} =
    tcpcall:listen(
        [{bind_port, 5000},
         {receiver,
          fun(Request) ->
              case binary_to_term(Request) of
                  Integer when is_integer(Integer) ->
                      term_to_binary(Integer * 2)
                  Cast ->
                      %% do something with cast request
                      ...
              end
          end}]),
...
```

On the client node:

```
{ok, Pid} = tcpcall:connect([{host, "server.com"}, {port, 5000}]),
EncodedRequest = term_to_binary(5),
{ok, EncodedReply} = tcpcall:call(Pid, EncodedRequest, 1000),
10 = binary_to_term(EncodedReply),

EncodedCast = term_to_binary({my_sophisticated_cast, 5, ["abc", make_ref()]}),
ok = tcpcall:cast(Pid, EncodedCast),
...
```

Note when you use functional object for processing casts (asynchronous
requests without a response), return of the function will be silently
discarded.

## Client and server as part of the supervision tree

Here is example for starting tcpcall server as part of the supervision tree of
your Erlang application:

```
%% @hidden
%% @doc Callback for application supervisor.
init(_Args) ->
    {ok, {
       {one_for_one, 5, 1},
       [
        ...
        {tcpcall_server,
         {tcpcall, listen, [[{name, my_server},
                             {bind_port, 5001},
                             {receiver, fun mymod:process_request/1}
                            ]]},
         permanent, brutal_kill, worker, [tcpcall]},
        ...
       ]
      }}.
```

Here is example for starting tcpcall client as part of the supervision tree of
your Erlang application:

```
%% @hidden
%% @doc Callback for application supervisor.
init(_Args) ->
    {ok, {
       {one_for_one, 5, 1},
       [
        ...
        {tcpcall_client,
         {tcpcall, connect, [[{name, my_client}, {host, "10.0.0.1"}, {port, 5001}]]},
         permanent, brutal_kill, worker, [tcpcall]},
        ...
       ]
      }}.
```

Now you can use tcpcall client from any process of your application like:

```
...
case tcpcall:call(my_client, Request, Timeout) of
    {ok, Reply} ->
        ...;
    {error, timeout} ->
        %% remote side doesn't respond within timeout
        ...;
    {error, overload} ->
        %% tcpcall client overloaded with incoming requests
        ...;
    {error, not_connected} ->
        %% connection to server is not alive
        ...
    {error, OtherError} ->
        %% something bad happen (network error or remote request processor crashed)
        ...
end,
...
```

or send casts like:

```
...
case tcpcall:cast(my_client, Request) of
    ok ->
        ...;
    {error, overload} ->
        %% tcpcall client overloaded with incoming requests
        ...;
    {error, not_connected} ->
        %% connection to server is not alive
        ...
    {error, OtherError} ->
        %% something bad happen (network error)
        ...
end,
...
```

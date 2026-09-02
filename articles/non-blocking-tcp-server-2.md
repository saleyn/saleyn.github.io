# Building a Non-blocking TCP server using OTP principles (part 2)

**Author**: Serge Aleynikov <saleyn at gmail.com>

## Overview

In my post, ["Building a Non-blocking TCP Server Using OTP Principles"](non-blocking-tcp-server.md), I covered how to write a TCP server using OTP non-blocking principles. That articles explains how to overcome the synchronization race condition during server socket accept states (specifically separating socket ownership, asynchronous `gen_tcp` listening, and worker process spawning). In order to make the server fully non-blocking, I had to use a hack - calling the undocumented `prim_inet:async_accept/2` function - to make sure that the acceptor can be invoked without blocking the `gen_server` process.

Below is a spiritual successor to that article, updating the concept for modern Erlang/OTP using the **`socket`** module (introduced via [EEP 153](https://www.google.com/search?q=https://www.erlang.org/eeps/eep-0053.html) and added natively in OTP 21+), which removes the need of calling any undocumented functions for non-blocking compliance.

# Building an Asynchronous TCP Server Using Erlang's `socket` Module

With modern Erlang/OTP, the NIF-based **`socket` module** fundamentally changes low-level networking. It exposes direct, non-blocking OS I/O operations and integrates with BEAM's asynchronous event engine without requiring `gen_tcp` driver wrappers.

Here is how to design a modern, non-blocking TCP acceptor using OTP principles and Erlang’s `socket` API.

## The Core Concept: Asynchronous `select` Workflows

In the classic `gen_tcp` world, you either blocked in `gen_tcp:accept/1` or set `{active, once/true}` and relied on process message delivery.

The modern `socket` API relies on explicit asynchronous operation completion, returning `{select, SelectInfo}` when a socket operation cannot be completed immediately without blocking.

1. **`socket:open/2` & `socket:bind/2**`: Initialize and bind the listener socket.
2. **`socket:listen/1`**: Put the socket into listening mode.
3. **`socket:accept/2`**: Attempt an immediate, non-blocking accept. If no client is pending, it returns `{select, SelectInfo}`.
4. **`erlang:select` messages**: When a connection arrives, the process receives a message containing the reference tag from `SelectInfo`:
```erlang
{'$socket', Socket, select, SelectRef}
```

5. **Direct Socket Transfer**: Unlike `gen_tcp`, where the listening socket process had to transfer ownership via `controlling_process/2`, `socket` descriptors can be passed directly to new worker processes without intermediate `inet` state synchronization.

## Implementation: The Modern Non-Blocking Acceptor

Below is a complete, minimal `gen_server` implementation using the modern `socket` API. Note that handling of client connections can also be done using a `gen_server` behavior, however here for the sake of simplicity we spawn a process that calls `worker_loop/1` and recursively fetches messages from its mailbox:

```erlang
-module(async_tcp_server).
-behaviour(gen_server).

%% API
-export([start_link/1]).

%% gen_server callbacks
-export([init/1, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).

-define(DEFAULT_PORT, 8080).

-record(state, {
    listen_socket   :: socket:socket(),
    client_sockopts :: list(),
    accept_ref      :: reference() | undefined
}).

%%%--------------------------------------------------------------------
%%% API Functions
%%%--------------------------------------------------------------------

start_link(Port, ClientSockOpts) ->
    gen_server:start_link({local, ?MODULE}, ?MODULE, [Port, ClientSockOpts], []).

%%%--------------------------------------------------------------------
%%% gen_server Callbacks
%%%--------------------------------------------------------------------

init([Port, ClientSockOpts]) ->
    %% 1. Open an IPv4 stream TCP socket
    case socket:open(inet, stream, tcp) of
        {ok, ListenSocket} ->
            %% 2. Set socket options (e.g., reuse address)
            %%
            %% For TCP, the window scale is negotiated during the three-way
            %% handshake, so SO_RCVBUF / SO_SNDBUF must be set on the listening
            %% socket before `listen/1`. Setting them on the accepted socket
            %% after `accept/2` is too late to affect the initial window.
            {LSockOpts, CliSockOpts} =
                lists:partition(ClientSockOpts, fun({Opt, _}) ->
                    lists:member(Opt, [rcvbuf, sndbuf])
                end),
            [ok = socket:setopt(ListenSocket, socket, Opt, Val) || {Opt, Val} <- LSockOpts],

            ok = socket:setopt(ListenSocket, socket, reuseaddr, true),

            %% 3. Bind to specified port
            Addr = #{family => inet, addr => any, port => Port},
            case socket:bind(ListenSocket, Addr) of
                {ok, _} ->
                    %% 4. Start listening
                    ok = socket:listen(ListenSocket),

                    %% 5. Begin non-blocking accept loop
                    State = #state{
                        listen_socket   = ListenSocket,
                        client_sockopts = CliSockOpts
                    },
                    {ok, start_accept(State)};
                {error, Reason} ->
                    {stop, {bind_error, Reason}}
            end;
        {error, Reason} ->
            {stop, {open_error, Reason}}
    end.

handle_call(_Request, _From, State) ->
    {reply, {error, bad_request}, State}.

handle_cast(_Msg, State) ->
    {noreply, State}.

%% Handle notification that socket is ready to accept a client
handle_info({'$socket', ListenSocket, select, SelectRef},
            #state{listen_socket = ListenSocket, accept_ref = SelectRef} = State) ->
    {noreply, start_accept(State#state{accept_ref = undefined})};

handle_info(Info, State) ->
    logger:notice("Unexpected info message: ~p", [Info]),
    {noreply, State}.

terminate(_Reason, #state{listen_socket = ListenSocket}) ->
    case ListenSocket of
        undefined -> ok;
        _ -> socket:close(ListenSocket)
    end.

%%%--------------------------------------------------------------------
%%% Internal Functions
%%%--------------------------------------------------------------------

%% Initiates an asynchronous accept request
start_accept(#state{listen_socket = ListenSocket, client_sockopts = CSockOpts} = State) ->
    %% 'nowait' tells socket:accept/2 to perform non-blocking I/O immediately
    case socket:accept(ListenSocket, nowait) of
        {ok, ClientSocket} ->
            %% Connection accepted immediately, spawn a handling process
            spawn_worker(ClientSocket, CSockOpts),
            start_accept(State);

        {select, {select_info, _Flags, SelectRef}} ->
            %% No incoming connection available right now.
            %% A '$socket' notification once a client connects.
            State#state{accept_ref = SelectRef};

        {error, Reason} ->
            %% If reasons are `emfile` or `enfile` - the OS ran out of file
            %% descriptors - you need to adjust/set `ulimit -n`
            logger:error("Failed to accept socket: ~p", [Reason]),
            State
    end.

%% Spawns a dedicated connection handler process
spawn_worker(ClientSocket, CSockOpts) ->
    spawn(fun() ->
        set_sock_opts(ClientSocket, CSockOpts),
        worker_loop(ClientSocket)
    end).

%% Asynchronous worker reading from client
worker_loop(ClientSocket) ->
    %% Attempt non-blocking read of up to 1024 bytes
    case socket:recv(ClientSocket, 1024, nowait) of
        {ok, Data} ->
            %% For the sake of this example just echo the data back to client
            ok = socket:send(ClientSocket, Data),
            worker_loop(ClientSocket);

        {select, {select_info, _Flags, _SelectRef}} ->
            %% Socket buffer empty; wait for readable notification
            receive
                {'$socket', ClientSocket, select, _Ref} ->
                    worker_loop(ClientSocket)
            end;

        {error, closed} ->
            socket:close(ClientSocket);

        {error, Reason} ->
            logger:error("Worker socket error: ~p", [Reason]),
            socket:close(ClientSocket)
    end.

set_sock_opts(ClientSocket, CSockOpts) ->
    try
        lists:foreach(CSockOpts, fun({Opt, Value}) ->
            case socket:setopt(ClientSocket, Opt, Value) of
                ok              -> ok;
                {error, closed} -> ok;
                {error, Reason} -> erlang:error({Opt, Reason})
            end
        end)
    catch _:Reason:StackTrace ->
        logger:error("Error setting client socket options ~p: ~p~n  ~p",
            [CSockOpts, Reason, StackTrace])
    end.
```

## Architectural Comparison: Classic `gen_tcp` vs Modern `socket`

| Design Factor | Classic `gen_tcp` Approach | Modern `socket` Module |
| --- | --- | --- |
| **Driver Overhead** | Port driver wrapper managed by `inet_drv`. | Direct C-NIF bindings executing close to kernel system calls (`sys_api`). |
| **Accept Mechanism** | Blocking `gen_tcp:accept/1` or async via hidden `prim_inet` tricks. | Explicit non-blocking `socket:accept(Listen, nowait)` returning `{select, Ref}`. |
| **Socket Transfer** | Required explicit execution of `gen_tcp:controlling_process/2`. | `socket:socket()` references are simple terms that can be used directly across processes. |
| **Active Mode Options** | `{active, true | once | N}` byte streams mapped to messages. | Fine-grained non-blocking reads using `socket:recv(Socket, Length, nowait)` and explicit select mechanics. |

## Why This Approach Fits OTP Principles

1. **Non-blocking Event Loop Integration**: The `socket:accept(..., nowait)` call eliminates process thread blocking. The `gen_server` remains responsive to incoming system/supervisor calls while awaiting network activity.
2. **Zero Race Conditions**: Because `socket` descriptors do not enforce strict process ownership in the legacy driver sense, worker processes can read from sockets as soon as they receive the term. There is no need for socket transfer handshakes.
3. **Kernel-Native Flow Control**: By specifying explicit chunk sizes and utilizing `nowait`, server processes consume memory on demand rather than letting uncontrolled `{active, true}` streams flood the message queue.

## Improvement Ideas

1. For strict OTP compliance the client connection handling processes need to implement `gen_server` behavior.
2. The fact that the client connection hanling processes get spawned unlinked and unmonitored theoretically opens the door for process leaks or DOS attacks. In production servers some rate-throttling may be needed, as well as process linking to make sure the processes are cleaned up when the listener process exits.

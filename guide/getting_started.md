Getting started
===============

Erlang is more than a language, it is also an operating system
for your applications. Erlang developers rarely write standalone
modules, they write libraries or applications, and then bundle
those into what is called a release. A release contains the
Erlang VM plus all applications required to run the node, so
it can be pushed to production directly.

This chapter walks you through all the steps of setting up
Cowboy, writing your first application and generating your first
release. At the end of this chapter you should know everything
you need to push your first Cowboy application to production.

Application skeleton
--------------------

Let's start by creating this application. We will simply call it
`hello_erlang`. This application will have the following directory
structure:

```
hello_erlang/
    src/
        hello_erlang.app.src
        hello_erlang_app.erl
        hello_erlang_sup.erl
        hello_handler.erl
    erlang.mk
    Makefile
    relx.config
```

Once the release is generated, we will also have the following
files added:

```
hello_erlang/
    ebin/
        hello_erlang.app
        hello_erlang_app.beam
        hello_erlang_sup.beam
        hello_handler.beam
    _rel/
    relx
```

As you can probably guess, the `.app.src` file end up becoming
the `.app` file, and the `.erl` files are compiled into `.beam`.
Then, the whole release will be copied into the `_rel/` directory.

The `.app` file contains various informations about the application.
It contains its name, a description, a version, a list of modules,
default configuration and more.

Using a build system like [erlang.mk](https://github.com/extend/erlang.mk),
the list of modules will be included automatically in the `.app` file,
so you don't need to manually put them in your `.app.src` file.

For generating the release, we will use [relx](https://github.com/erlware/relx)
as it is a much simpler alternative to the tool coming with Erlang.

First, create the `hello_erlang` directory. It should have the same name
as the application within it. Then we create the `src` directory inside
it, which will contain the source code for our application.

``` bash
$ mkdir hello_erlang
$ cd hello_erlang
$ mkdir src
```

Let's first create the `hello_erlang.app.src` file. It should be pretty
straightforward for the most part. You can use the following template
and change what you like in it.

``` erlang
{application, hello_erlang, [
    {description, "Hello world with Cowboy!"},
    {vsn, "0.1.0"},
    {modules, []},
    {registered, [hello_erlang_sup]},
    {applications, [
        kernel,
        stdlib,
        cowboy
    ]},
    {mod, {hello_erlang_app, []}},
    {env, []}
]}.
```

The `modules` line will be replaced with the list of modules during
compilation. Make sure to leave this line even if you do not use it
directly.

The `registered` value indicates which processes are registered by this
application. You will often only register the top-level supervisor
of the application.

The `applications` value lists the applications that must be started
for this application to work. The Erlang release will start all the
applications listed here automatically.

The `mod` value defines how the application should be started. Erlang
will use the `hello_erlang_app` module for starting the application.

The `hello_erlang_app` module is what we call an application behavior.
The application behavior must define two functions: `start/2` and
`stop/1`, for starting and stopping the application. A typical
application module would look like this:

``` erlang
-module(hello_erlang_app).
-behavior(application).

-export([start/2]).
-export([stop/1]).

start(_Type, _Args) ->
    hello_erlang_sup:start_link().

stop(_State) ->
    ok.
```

That's not enough however. Since we are building a Cowboy based
application, we also need to initialize Cowboy when we start our
application.

Setting up Cowboy
-----------------

Cowboy does nothing by default.

Cowboy uses Ranch for handling the connections and provides convenience
functions to start Ranch listeners.

The `cowboy:start_http/4` function starts a listener for HTTP connections
using the TCP transport. The `cowboy:start_https/4` function starts a
listener for HTTPS connections using the SSL transport.

Listeners are a group of processes that are used to accept and manage
connections. The processes used specifically for accepting connections
are called acceptors. The number of acceptor processes is unrelated to
the maximum number of connections Cowboy can handle. Please refer to
the [Ranch guide](http://ninenines.eu/docs/en/ranch/HEAD/guide/)
for in-depth information.

Listeners are named. They spawn a given number of acceptors, listen for
connections using the given transport options and pass along the protocol
options to the connection processes. The protocol options must include
the dispatch list for routing requests to handlers.

The dispatch list is explained in greater details in the
[Routing](routing.md) chapter. For the purpose of this example
we will simply map all URLs to our handler `hello_handler`,
using the wildcard `_` for both the hostname and path parts
of the URL.

This is what the `hello_erlang_app:start/2` function looks like
with Cowboy initialized.

``` erlang
start(_Type, _Args) ->
    Dispatch = cowboy_router:compile([
        %% {URIHost, list({URIPath, Handler, Opts})}
        {'_', [{'_', hello_handler, []}]}
    ]),
    %% Name, NbAcceptors, TransOpts, ProtoOpts
    cowboy:start_http(my_http_listener, 100,
        [{port, 8080}],
        [{env, [{dispatch, Dispatch}]}]
    ),
    hello_erlang_sup:start_link().
```

Do note that we told Cowboy to start listening on port 8080.
You can change this value if needed.

Our application doesn't need to start any process, as Cowboy
will automatically start processes for every incoming
connections. We are still required to have a top-level supervisor
however, albeit a fairly small one.

``` erlang
-module(hello_erlang_sup).
-behavior(supervisor).

-export([start_link/0]).
-export([init/1]).

start_link() ->
    supervisor:start_link({local, ?MODULE}, ?MODULE, []).

init([]) ->
    {ok, {{one_for_one, 10, 10}, []}}.
```

Finally, we need to write the code for handling incoming requests.

Handling HTTP requests
----------------------

Cowboy features many kinds of handlers. For this simple example,
we will just use the plain HTTP handler, which has three callback
functions: `init/3`, `handle/2` and `terminate/3`. You can find more
information about the arguments and possible return values of these
callbacks in the
[cowboy_http_handler function reference](http://ninenines.eu/docs/en/cowboy/HEAD/manual/cowboy_http_handler).

Our handler will only send a friendly hello back to the client.

``` erlang
-module(hello_handler).
-behavior(cowboy_http_handler).

-export([init/3]).
-export([handle/2]).
-export([terminate/3]).

init(_Type, Req, _Opts) ->
    {ok, Req, undefined_state}.

handle(Req, State) ->
    {ok, Req2} = cowboy_req:reply(200, [
        {<<"content-type">>, <<"text/plain">>}
    ], <<"Hello World!">>, Req),
    {ok, Req2, State}.

terminate(_Reason, _Req, _State) ->
    ok.
```

The `Req` variable above is the Req object, which allows the developer
to obtain information about the request and to perform a reply.
Its usage is documented in the
[cowboy_req function reference](http://ninenines.eu/docs/en/cowboy/HEAD/manual/cowboy_req).

The code for our application is ready, so let's build a release!

Compiling
---------

First we need to download `erlang.mk`.

``` bash
$ wget https://raw.github.com/extend/erlang.mk/master/erlang.mk
$ ls
src/
erlang.mk
```

Then we need to create a Makefile that will include `erlang.mk`
for building our application. We need to define the Cowboy
dependency in the Makefile. Thankfully `erlang.mk` already
knows where to find Cowboy as it features a package index,
so we can just tell it to look there.

``` Makefile
PROJECT = hello_erlang

DEPS = cowboy
dep_cowboy = pkg://cowboy master

include erlang.mk
```

Note that when creating production nodes you will most likely
want to use a specific version of Cowboy instead of `master`,
and properly test your release every time you update Cowboy.

If you type `make` in a shell now, your application should build
as expected. If you get compilation errors, double check that you
haven't made any typo when creating the previous files.

``` bash
$ make
```

Generating the release
----------------------

That's not all however, as we want to create a working release.
For that purpose, we need to create a `relx.config` file. When
this file exists, `erlang.mk` will automatically download `relx`
and build the release when you type `make`.

In the `relx.config` file, we only need to tell `relx` that
we want the release to include the `hello_erlang` application,
and that we want an extended start script for convenience.
`relx` will figure out which other applications are required
by looking into the `.app` files for dependencies.

``` erlang
{release, {hello_erlang, "1"}, [hello_erlang]}.
{extended_start_script, true}.
```

The `release` value is used to specify the release name, its
version, and the applications to be included.

We can now build and start the release.

``` bash
$ make
$ ./_rel/bin/hello_erlang console
```

If you then access `http://localhost:8080` using your browser,
you should receive a nice greet!

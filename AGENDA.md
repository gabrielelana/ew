# Elixir Workshop Agenda

## List Functions


## Robozzle(I)


## Diamond Kata


## Parallel

* Naked send and receive solution
  * Fibonacci implementation
  * Explain `PID`, `self`, `spawn`, `spawn_link`, `send`, `receive` and mailboxes
  * `0..35 |> Enum.map(&Fibonacci.of/1) |> IO.inspect`
  * `0..35 |> Parallel.map(&Fibonacci.of/1) |> IO.inspect`
* Task solution
  * `&Task.async(fn -> f.(&1) end)` and `&Task.await/1`
  * Problems:
    * We wait for the results in order, a long task could block other, show with `Waste.ms`
    * It's not easy to force a global timeout, useful when you have a limited amount of time
    * It's an all or nothing solution, if a task crashes, the master crash and so the other tasks
* Collect as soon as possible
  * `Enum.map(&Task.async(fn -> f.(&1) end)) |> collect`, use `Task.find(tasks, message)`
  * Show what is `message`
  * Results are in reverse order, but maybe the initial order must be kept…
* Collect as soon as possible keeping order **[CHALLENGE]**
  * Use `Enum.zip(1..Enum.count(enumerable))`, `Enum.sort_by(&elem(&1, 0))`
* Handle timeout
  * Send timeout message `Process.send_after(self, {:timeout, ref}, 2_000)`
  * `try do collect` then `after :erlang.cancel_timer(timer)`
  * Explain `receive` with `after 0` (`after` in try is different than `after` in receive)
  * Tag results `{:ok, results}` and `{:timeout, count, results}`
  * Sort results with `sort` function
  * Show timeout with `0..300 |> Enum.map(fn _ -> :random.uniform(35) end) |> Parallel.map(&Fibonacci.of/1)`
  * Show timeout with `0..10 |> Enum.map(fn _ -> :random.uniform(2_000) end) |> Parallel.map(&Waste.ms/1, timeout: 1_000)`
* Introduction to distributed Elixir
  * Nodes, names, cookies, `epmd`
  * `iex` -> `Node.alive?` is false
  * `iex --sname foo` -> `Node.alive?` is true (`sname` stands for short name)
  * We can also start with a full name `iex --name "foo@0.0.0.0"` if you want
  * Nodes starts to know each other at the first ping `Node.ping(:"foo@apollo")`
  * List of nodes with `Node.list`
  * Nodes keep pinging each other, try to kill a node and it's gone `Node.list`
  * Send message to a registered process in another node `Process.register(self, :shell)` and `send {:shell, "foo@apollo"}, "Hello"`
    * Send the pid and show the "location transparency"
  * The great magic of `Node.spawn(:"foo@apollo", fn -> IO.puts("Hello") end)`
  * You cannot run code that doesn't exists on other nodes `Node.spawn(:"foo@apollo", Fibonacci, :of, [13])`
  * `[{Fibonacci, b}, _] = Code.load_file("parallel.exs")`
  * `Node.spawn(:"foo@apollo", :code, :load_binary, [:"Elixir.Fibonacci", 'fibonacci.erl', b])`
    * same as `:rpc.call("foo@apollo", :code, :load_binary, [:"Elixir.Fibonacci", 'fibonacci.erl', b])`
  * Run `Fibonacci.of(13)` in `foo@apollo`
* Distributed solution
  * Replace `&Task.async(fn)` with `&spread(&1, f)`, change `receive` in `collect/3`, we have the same thing as before
  * Connect nodes `:net_kernel.start([:self, :shortnames])`, ping other nodes, show `[Node.self | Node.list]`
  * Try to run code on a random node `nodes |> Enum.random |> Node.spawn_link` crashes because we don't have the code on the other node!
  * `{:module, _, module, _} = defmodule XXX ...` and `:rpc.call(n, :code, :load_binary, [:"Elixir.ModuleName", 'module_name.erl', module])`
  * Try to run code and it crashes again, explain difference between function compiled and evaluated, `{waste_ms, _} = Code.eval_string("fn(e) -> Waste.ms(e) end")`
* Handle errors
  * Back to `collect-asap`
  * `spawn` not `spawn_link`
  * Monitor PIDs and keep them with references
  * Receive `{:DOWN, _, _, pid, _}` and remove with `List.keydelete`


## Echo Server

* Create an app skeleton and look into it `mix new echo` (`mix.exs`, `config/config.exs`, `test/test_helper.exs`, `test/echo_test.exs`)
* Difference between: Application, Library and Project
  * Application: started and stopped as a unit and reused in other systems
  * Library: like applications but cannot be started and stopped
  * Project: structure of files organized in a way that `Mix` understands
* `Echo` as `GenServer` to accept connections and echo back
  * Socket is not active, `GenServer.cast(self, :accept)`
* Run with `iex -S mix` and `Echo.start_link(4242)`, try with `netcat localhost 4242`, close the client and everything blow up
* Handle closed socket
* Run the server more gracefully
  * Make it an application `Echo`, `Echo.Listener`, add `mod: {Echo, []}, env: [port: 4242]` to `application` in `mix.exs`
  * Replace `IO.puts` with `Logger.info` after `require Logger`
  * Start with `mix run --no-halt`
* Handle more than one client at the time
  * Extract `Echo.Server`, give socket client after `init/1` because you need to have the pid to give the socket ownership
  * Start multiple clients, it works, but when you close one of them... Crash! Because the process are linked
* Supervise them all
  * Add `Echo.Supervisor` and `Echo.Server.Supervisor`, then app should start `Echo.Supervisor`
* Release and update
  * Handle timeout in `:gen_tcp.accept` and `:gen_tcp.recv`, implement `code_change/3`
  * Add `{:exrm, "~> 0.19.9}` to the dependencies, then `mix do deps.get, compile`, look at the new release related tasks with `mix help`
  * Package `env MIX_ENV=prod mix release`
  * Look at `rel/echo` directory
  * Deploy `mkdir ~/tmp/echo`, `cp rel/echo/releases/0.0.1/echo.tar.gz ~/tmp/echo`, `cd ~/tmp/echo`, `tar -xzvf echo.tar.gz`
  * Start `bin/echo start`
  * Attach `bin/echo attach` (`CTRL-D` to detach)
  * Leave a `netcat` session open
  * Echo back in uppercase and increment the version number
    * Create release `env MIX_ENV=prod mix release`
    * Deploy `mkdir ~/tmp/echo/releases/0.0.2`, `cp rel/echo/releases/0.0.2/echo.tar.gz ~/tmp/echo/releases/0.0.2`
    * Upgrade `bin/echo upgrade "0.0.2"`, client is still alive and now it's uppercase
    * Downgrade `bin/echo downgrade "0.0.1"` client is still alive


## Robozzle(II)

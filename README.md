# Elixir Workshop Agenda

## List Functions

* Initial **[start]**
  * Explain `*.exs`
  * Explain `ExUnit.start` with `pending`
  * Explain `ExUnit.Case, async: true`
  * Explain `assert X == Y`
  * Explain `@spec`
  * Make it green
    * Explain KO `concat(ll), do: do_concat(ll, [])` with `do_concat(t, append(r, h))`
    * Explain OK `concat(ll), do: do_concat(reverse(ll), [])` with `do_concat(t, append(h, r))`
* Reduce **[reduce]**
  * Implement all functions using reduce


## Robozzle(I)

* Explain [Robozzle](http://robozzle.com)
* Parse one line scenario **[one-line-scenario]**
  * Explain types, tests and run `dly`
* Parse multiple lines scenario **[multiple-lines-scenario]** **[CHALLENGE]**
* Run move commands with `rc/3` **[move-commands]**
* Pick stars with `rc/3` **[pick-stars]**
* Run paint commands with `rc/3` **[paint-commands]** **[CHALLENGE]**
  * Change command type definition
* Run conditional commands with `rc/3` **[conditional-commands]**
  * Change command type definition
* Run multiple commands `run/3` **[run-commands]**
  * Add `complete?/1`
* Check for out of stage in `run/3` **[out-of-stage]** **[CHALLENGE]**
* Run multiple functions `run/3` **[run-functions]**
  * Show `functions` and `stack` types
  * Show add `{:call, f}` to command type
  * Show how tests are changed `run/3` and new test for call command
  * Change the execution model, we stop to run when stack is empty
    * Change type specification for `run/4` as `@spec run(functions, ship, stage, stack) :: {outcome, ship, stage}`
    * Change type specification for `rc/3` add return option `{ship, stage, function_name}`
  * Add the `{:call, f}` command
* Check for stack overflow `run/3` **[stack-overflow]** **[CHALLENGE]**
  * Show tests that already pass
* Check for out of time `run/3` **[out-of-time]** **[CHALLENGE]**
  * Add steps type and parameter to `run/3`
* Extract `assert_scenario` **[assert-scenario]** **[CHALLENGE]**
  * Parse the empty tile ".."
* Create `scenario` macro **[scenario-macro]**
  * Explain macros
    * Functions executed at compile time
    * Functions from AST to AST
    * They are called until no macros are left to call
    * `quote do if(cool), do: IO.puts("COOOL!") end`
    * `Macro.to_string(v(7))`
    * `Macro.expand(v(7), __ENV__)`
    * `Macro.expand(v(7), __ENV__) |> Macro.to_string |> IO.puts`
  * Create `Robozzle.AcceptanceTest.Macro`
  * Require module in `Robozzle.AcceptanceTest`
  * Define `defmacro scenario(outcome, args)` and inspect input
  * It would be awful to describe what we want directly in AST, but we have our friend `quote` to help us
  * Describe inside `quote do/end` what output do you want
  * Refine `defmacro scenario(outcome, {:__block__, _, block}`
  * Define `scenario_from(block)` as `Enum.find(&is_binary/1)` in block list
  * Define `function_from(:f1, block)`
  * Define `defmacro __using__(_opts) do` to import things
  * Inside `__using__` add `Module.put_attribute __MODULE__, :acceptance_counter, 0`
  * Inside quote use `Module.get_attribute(__MODULE, :acceptance_counter)`
  * Support `scenario(outcome, title) do/end` **[CHALLENGE]**
    * Call a macro from the other macro -> macro expands until there's no macro


## Parallel

* Naked send and receive solution **[simple]**
  * Fibonacci implementation
  * `0..35 |> Enum.map(&Fibonacci.of/1) |> IO.inspect`
  * We want to create a `Parallel.map` that uses all the cores we have
  * Explain `PID`, `self`, `spawn`, `spawn_link`, `send`, `receive` and mailboxes in the shell
  * `0..35 |> Parallel.map(&Fibonacci.of/1) |> IO.inspect`
  * try with `:random.seed(:os.timestamp)` and `:random.uniform(25) + 10`
* Task solution **[task]**
  * `&Task.async(fn -> f.(&1) end)` and `&Task.await/1`
  * Problems:
    * We wait for the results in order, a long task could block other, show with `Waste.ms`
    * It's not easy to force a global timeout, useful when you have a limited amount of time
    * It's an all or nothing solution, if a task crashes, the master crash and so the other tasks
* Collect as soon as possible **[collect-asap]**
  * `Enum.map(&Task.async(fn -> f.(&1) end)) |> collect`, use `Task.find(tasks, message)`
  * Show what is `message`
  * Results are in reverse order, but maybe the initial order must be kept…
* Collect as soon as possible keeping order **[keep-order]** **[CHALLENGE]**
  * Show with `IO.inspect` before and after `Parallel.map(&Waste.ms/1)` the order is different
  * Use `Enum.zip(1..Enum.count(enumerable))`, `Enum.sort_by(&elem(&1, 0))`
* Handle timeout **[handle-timeout]**
  * Send timeout message `Process.send_after(self, {:timeout, ref}, 2_000)` in `collect`
  * Always cancel timer after `try do collect` then `after :erlang.cancel_timer(timer)`
  * Explain `receive` with `after 0` (`after` in try is different than `after` in receive)
  * Tag results `{:ok, results}` and `{:timeout, count, results}`
  * Add `timeout` as option to `Parallel.map`
  * Show timeout with `0..10 |> Enum.map(fn _ -> :random.uniform(2_000) + 1_000 end) |> Parallel.map(&Waste.ms/1, timeout: 1_000)`
* Handle failures **[handle-failures]**
  * The simplest approach would be to `try/catch` around `f.(e)` in `Task.async` ignoring errors **[CHALLENGE]**
    * Fail fast is good when you don't know what to do in case of error/failure, otherwise catch the error!
    * Warning: `catch` doesn't catch exit signals, you don't know what it's going on inside `f`
    * The task is linked to the master process, if the task crashes then the master is going to crash to
    * We can trap exits but it's not a good idea to trap exits temporarily
      * `iex> spawn_link fn -> raise "BOOOM!!!" end`
      * `iex> Process.flag(:trap_exit, true)`
      * `iex> spawn_link fn -> raise "BOOOM!!!" end`
    * We can monitor the tasks
  * Replace `Task.async` with `task` function that uses `spawn`
    * `Enum.map(&task(&1, f))` returns reference and pid `pid = spawn(fn -> send(me, {reference, f.(e)}) end); {reference, pid}`
    * In collect use `List.keytake(tasks, task_ref, 0) => {_, tasks}` in `receive {task_ref, result}`
    * We are in the same condition as before
  * Monitor PIDs in `Parallel.task` with `Process.monitor(pid)`
  * Receive `{:DOWN, _, _, pid, _}` and remove with `List.keytake(tasks, pid, 1)`
* Introduction to distributed Elixir
  * Nodes, names, cookies, `epmd`
  * `iex` -> `Node.alive?` is false
  * `iex --sname foo` -> `Node.alive?` is true (`sname` stands for short name)
  * We can also start with a full name `iex --name "foo@0.0.0.0"` if you want
  * We can also start after with `:net_kernel.start([:foo, :shortnames])`
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
* Distributed solution **[distributed]**
  * Connect nodes `:net_kernel.start([:self, :shortnames])`, ping other nodes `Node.ping(:"foo@apollo")`, show `[Node.self | Node.list]`
  * Try to run code on a random node `[Node.self|Node.list] |> Enum.random |> Node.spawn` crashes because we don't have the code on the other node!
  * Capture the binary code of the modules with `{:module, _, module, _} = defmodule XXX ...`
  * Send the binary code to the other nodes `Node.list |> Node.map(fn(n) -> :rpc.call(n, :code, :load_binary, [:"Elixir.ModuleName", 'module_name.erl', module]) end)`
  * Run the code and show in `task` the current node with `IO.inspect(Node.self)`
  * Show that failure handling works transparently across nodes


## Counter Server
* Create an app skeleton `mix new counter` and look into it (`mix.exs`, `config/config.exs`, `test/test_helper.exs`, `test/counter_test.exs`)
* Difference between: Application, Library and Project
  * Application: started and stopped as a unit and reused in other systems
  * Library: like applications but cannot be started and stopped
  * Project: structure of files organized in a way that `Mix` understands
* Create a process that starts with `n` and count down to `0`
* Create a process that implements a key/value store **[CHALLENGE]**
* What could go wrong? What are the problems?
  * Use ref to ensure uniqueness of the request
  * Server may die after message sent
  * Server name is not registered
  * Timeout to avoid deadlocks
* Turn it into a `GenServer`
  * Show `:sys.trace(:counter, true)` and `:sys.trace(:counter, false)`
  * Show `:sys.statistics(:counter, true)`, `:sys.statistics(:counter, :get)` and `:sys.statistics(:counter, false)`
  * Show `:sys.get_status(:counter)`
  * Show `:sys.get_state(:counter)`
* Turn it into an `Agent`


## Echo Server
* Create a `GenServer` `Echo` **[echo-server]**
  * `Echo.start_link(port)`
  * In `init/1` accept connections with `:gen_tcp.listen(port, options)` with options `[:binary, packet: :line, active: false, reuseaddr: true]`
    * Socket is not active, so `{:ok, socket_server} -> GenServer.cast(self, :accept)` and reply with `{:ok, socket_server}`
  * `handle_cast(:accept, status)`
    * `{:ok, socket_client} = :gen_tcp.accept(socket_server)`
    * `serve(socket_client)`
    * after serve keep accepting with `GenServer.cast(self, :accept)`
    * `{:noreply, socket_server}`
  * `serve(socket_client)`
    * `socket_client |> read_line |> echo_line |> serve`
    * `read_line` -> `:gen_tcp.recv(socket_client, 0)`
    * `echo_line` -> `:gen_tcp.send(socket_client, line)`
  * Run with `iex -S mix` and `Echo.start_link(4242)`
    * Try with `netcat localhost 4242`
    * It works but when close the client and everything blow up, why?
* Handle closed socket **[handle-closed-socket]** **[CHALLENGE]**
  * Handle `{:error, _}` in `read_line` and `echo_line`
  * Propagate `{:error, _}` in `echo_line`
  * Reply with `:ok` in `serve` if `{:error, _}`
* Make it an application **[application]**
  * Show `iex> Application.started_applications`
  * Add application module callback in `mix.exs`, add `mod: {Echo, []}` to `application`
  * Add default port configuration in environment, add `env: [port: 4242]` to `application`
  * Save `lib/echo.ex` in `lib/echo/server.ex`
  * Turn `Echo` in an application
    * `use Application`
    * `def start(type, [])` -> `start(type, Application.get_env(:echo, :port)`
    * `def start(_type, port)` -> `Echo.Server.start_link(port)`
  * Start with `iex -S mix` or `mix run --no-halt`
  * Replace `IO.puts` with `Logger.info` after `require Logger`
* Handle multiple clients **[multiple-clients]**
  * Extract `Echo.Listener`
    * `def serve(client_socket)`
      * `{:ok, server} = Echo.Server.start_link`
      * `:gen_tcp.controlling_process(client_socket, server)`
      * `Echo.Server.meet(server, client_socket)`
    * We can't give the socket at `start_link` because you need to have the pid to give the socket ownership
  * Make `Echo.Server` work **[CHALLENGE]**
    * `def init(_), do: {:ok, :waiting}`
    * `handle_cast({:meet, socket_client}, :waiting)` -> `GenServer.cast(self, :serve)` and `{:noreply, socket_client}`
    * `handle_cast(:serve, :waiting)` -> `socket_client |> read_line |> echo_line |> continue`
    * `def continue({:error, reason}), do: {:stop, reason}`
    * `def continue(socket_client)` -> `GenServer.cast(self, :serve)` and `{:noreply, socket_client}`
  * Change `Echo.start_link` from `Echo.Server` to `Echo.Listener`
  * Start multiple clients, it works, but when you close one of them... Crash! Because the process are linked
* Enter the supervisors **[supervisors]**
  * Add `Echo.Supervisor`
    * `def start_link(port)` -> `Supervisor.start_link(__MODULE__, port)`
    * `def init(port)`
      * `children = [worker(Echo.Listener, [port])]`
      * `supervise(children, strategy: :one_for_one)`
    * The application should start the supervisor `Echo.start` -> `Echo.Supervisor.start_link`
    * Now it blows again but the server restarts!
  * Add `Echo.Server.Supervisor`
    * `def start_link(args, opts)` -> `Supervisor.start_link(__MODULE__, args, opts)`
    * `def init(_)`
      * `children = [worker(Echo.Server, [])]`
      * `supervise(children, strategy: :simple_one_for_one)`
    * Add `supervisor(Echo.Server.Supervisor, [[], [name: Echo.Server.Supervisor])` in `Echo.Supervisor` children
    * Start child in listener `Supervisor.start_child(Echo.Server.Supervisor, [])`
  * Start multiple clients, when you close one you still see a bad log
  * When the socket close then it's normal to stop the server
    * In `Echo.Server.continue` handle `{:error, :closed, s}` -> `{:stop, :normal, s}`
* Release and update **[release]**
  * There's a problem, right now, both `Echo.Server` and `Echo.Listener` are blocking on `accept` and `recv`, they can't handle a release upgrade!!!
    * In `Echo.Listener` use `:gen_tcp.accept(ss, 500)` when `{:error, :timeout}` -> `GenServer.cast(self, :accept)`
    * In `Echo.Server` use `:gen_tcp.recv(sc, 0, 500)` when `{:error, :timeout}` -> `{:timeout, sc}`
      * `Echo.Server.echo_line` handle `{:timeout, socket_client}` -> `socket_client`
  * Add `{:exrm, "~> 0.19.9"}` to the dependencies, then `mix do deps.get, compile`, look at the new release related tasks with `mix help`
  * Package `env MIX_ENV=prod mix release`
  * Look at `rel/echo` directory
  * Deploy `mkdir ~/tmp/echo`, `cp rel/echo/releases/0.0.1/echo.tar.gz ~/tmp/echo`, `cd ~/tmp/echo`, `tar -xzvf echo.tar.gz`
  * Start `bin/echo start`
  * Attach `bin/echo attach` (`CTRL-D` to detach)
  * Leave a `netcat` session open
  * Change `Echo.Server.echo_line` put line in `String.upcase(line)`
    * Bump the version number to `0.0.2`
    * Create release `env MIX_ENV=prod mix release`
    * Deploy `mkdir ~/tmp/echo/releases/0.0.2`, `cp rel/echo/releases/0.0.2/echo.tar.gz ~/tmp/echo/releases/0.0.2`
    * Upgrade `bin/echo upgrade "0.0.2"`, client is still alive and now it's uppercase
    * Downgrade `bin/echo downgrade "0.0.1"` client is still alive


## Robozzle(II)
* Application **[end-I]**
  * Generate the application with `mix new ew-robozzle --sup --app robozzle` and look at the directories
  * Create `Robozzle.Runner` and `Robozzle.Parser`
  * Import all the tests
* Runner server **[runner-server]** **[CHALLENGE]**
  * `Robozzle.Server.run(pid, fs, ship, stage)`
  * `Robozzle.Server.start_link(opts \\ [])` -> `GenServer.start_link(__MODULE__, :ok, opts)`
  * Add worker `worker(Robozzle.Runner.Server, [[name: :runner]])`
  * `iex -S mix` show `Application.started_applications`
  * `Robozzle.Runner.Server.run(:runner, %{f1: [:forward]}, {{0,0}, :east}, %{{0,0} => :blue, {1,0} => {:blue, :star}})`
  * Try it in a test with `setup` function. Why we don't need to stop the server? Why can we run all test in parallel?
* Runner as interactive server **[interactive-server]** **[CHALLENGE]**
  * Look at the tests in `test/robozzle_runner_server_test.exs`
  * Implement `Robozzle.Runner.step` without changing `Robozzle.Runner.run`
  * Implement `Robozzle.Runner.Server.load` with specs
  * Implement `Robozzle.Runner.Server.step` with specs
* Runners in pool **[poolboy]**
  * Look at the test in `test/robozzle_test.exs`
  * Add poolboy dependency `{:poolboy, github: "devinus/poolboy"}` `mix do deps.get, deps.compile`
  * Configure poolboy in `lib/robozzle.ex`
  * Start `Robozzle.Server` as worker in `lib/robozzle`
  * `mix test` are still working, why?
  * `:poolboy.transaction(:runners, &Runner.Server.run(&1, fs, ship, stage))`
  * Add `Robozzle.run` application level function
  * We have a problem: we have a pool but we are running one runner at the time
* Use all runners in pool **[parallel-runners]** **[CHALLENGE]**
  * We can block the client of `GenServer.call/2` keeping responsive the server with `{:noreply, state}` and the later use of `GenServer.reply/2`
  * `spawn_link(fn -> :poolboy.transaction(...) end); {:noreply, state}`
  * Demonstrate with `iex -S mix` and `import_file "priv/parallel_runners.exs"`. Add `:timer.sleep(500)` in `Robozzle.Runner.Server` in run handler
* Solver **[solver]**
  * `Robozzle.Server.solve(server, ship, stage) :: :busy | :timeout | {:solution, Runner.functions}`
  * Add `:busy` return value for `Robozzle.Server.run` and `Robozzle.run`
    * Reply with `:busy` when state is `%{solving: _}`, default is `%{}`
  * Add `handle_call({:solve, …})`
    * `Process.send_after(self, :timeout, 1_000 * 60 * 5)`
    * `Robozzle.Drone.explore(scenario, [], constraints, ship, stage, self)`
    * `{:noreply, %{solving: scenario, from: from}}`
  * Add `Robozzle.Server.solved?(server, scenario)`
  * Add `Robozzle.Server.report_solution(server, scenario, solution)`
    * `GenServer.reply(from, {:solution, solution})`
  * `Drone.explore`
    * `spawn`
    * Check if unfit for constraints -> return
    * Check if already found a solution -> return
    * Run in pool `:poolboy.transaction(:runners, &Runner.Server.run())`
    * Handle outcome
      * `:complete` -> `Server.report_solution`
      * `:out_of_stage` -> `:ok`
      * otherwise -> for each next solution -> explore
* Single runner solver: run multiple solver in parallel until runners are available in pool **[CHALLENGE]**

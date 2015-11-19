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

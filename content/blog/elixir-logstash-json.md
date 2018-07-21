---
title: "Sending Elixir logs to Logstash as JSON"
date: 2018-07-21T17:04:34+02:00
draft: false
---

Adopting Elixir was a pleasure - it fit nicely into our microservice architecture and most of our tech stack. The only missing piece was our [ELK-based logging infrastructure](https://www.elastic.co/elk-stack), where we sent logs to Logstash formatted in JSON, an easily machine-readable format. As there was no library at the time that did this, I decided to write one myself!

In this post, we will cover how to create your own Elixir logger backend, how to send JSON logs to Logstash via TCP, unit testing, and strategies for handling log spikes.

You can find __logstash-json__ on [GitHub](https://github.com/svetob/logstash-json).

# Creating an Elixir logging backend

Let's start with the basics and, from an empty `mix new` project, create a simple console JSON logger.

An Elixir Logger backend is simply a GenEvent event manager. So all we have to do is create an standard `:gen_event` event handler.

First we need to handle initalization. Our logger should be configurable via `config.exs`, so on `init` we should read the application environment and store it in the logger's state. To make sure we handle restarts after errors without losing state, we persist the state back to the application environment. For now, the only parameter we need is the log level, which we can default to `:info`.

We must also support reconfiguring the logger during runtime by handling the event `{:configure, opts}`.

```elixir
# lib/jsonlogger_console.ex

defmodule JsonLogger.Console do
  @moduledoc """
  Logger backend which logs to console in JSON format.
  """

  @behaviour :gen_event

  def init({__MODULE__, name}) do
    {:ok, configure(name, [])}
  end

  def handle_call({:configure, opts}, %{name: name}) do
    {:ok, :ok, configure(name, opts)}
  end

  defp configure(name, opts) do
    env = Application.get_env(:logger, name, [])
    opts = Keyword.merge(env, opts)
    Application.put_env(:logger, name, opts)

    level = Keyword.get(opts, :level, :info)

    %{level: level, name: name}
  end
```

Now we are ready to handle log messages!

Log message events arrive in the format `{level, group_leader, {Logger, message, timestamp, metadata}}`. We should first compare the log level with our configuration to see if the message should be logged. Then we simply build our log message and print it to console. In this example i'm using [Poison](https://github.com/devinus/poison) to serialize the message to a JSON string.

There is an interesting parameter called `group_leader`. Each BEAM process belongs to a group. Each group has a [group leader](http://erlang.org/doc/man/erlang.html#group_leader-0) which handles all I/O for that group. According to the [Logger documentation](https://hexdocs.pm/logger/Logger.html#content):

> It is recommended that handlers ignore messages where the group leader is in a different node than the one where the handler is installed.

The idea is logs from one node should not be printed to the console of another node. If we are logging to Logstash, we ignore this advice because all our logs are sent to an external system. Now we are printing to console though, so let's pattern match for this case.

The only other noteworthy surprise here is the mind-numbing approach to getting the system timezone. Really, Elixir?

```elixir
  def handle_event({_level, group_leader, _info}, state)
      when node(group_leader) != node() do
    {:ok, state}
  end

  def handle_event({level, group_leader, {Logger, msg, ts, md}}, state) do
    if Logger.compare_levels(level, state.level) != :lt do
      log(event(level, msg, ts, md), state)
    end
    {:ok, state}
  end

  def handle_info(_msg, state) do
    {:ok, state}
  end

  def event(level, message, timestamp, metadata) do
    %{
      "@timestamp": format_date(timestamp) <> timezone(),
      level: level,
      message: to_string(message),
      module: metadata[:module],
      function: metadata[:function],
      line: metadata[:line]
    }
  end

  defp log(event, _state) do
    case Poison.encode(event) do
      {:ok, msg} ->
        IO.puts msg

      {:error, reason} ->
        IO.puts "Serialize error: #{inspect reason}, event: #{inspect event}"
    end
  end

  ## Timestamp shenanigans

  defp format_date({{year, month, day}, {hour, min, sec, millis}}) do
    {:ok, ndt} = NaiveDateTime.new(year, month, day,
                                   hour, min, sec, {millis, 3})
    NaiveDateTime.to_iso8601(ndt, :extended)
  end

  defp timezone() do
    offset = timezone_offset()
    minute = offset |> abs() |> rem(3600) |> div(60)
    hour   = offset |> abs() |> div(3600)
    sign(offset) <> zero_pad(hour, 2) <> ":" <> zero_pad(minute, 2)
  end

  defp timezone_offset() do
    t_utc = :calendar.universal_time()
    t_local = :calendar.universal_time_to_local_time(t_utc)

    s_utc = :calendar.datetime_to_gregorian_seconds(t_utc)
    s_local = :calendar.datetime_to_gregorian_seconds(t_local)

    s_local - s_utc
  end

  defp zero_pad(val, count) do
    num = Integer.to_string(val)
    :binary.copy("0", count - byte_size(num)) <> num
  end

  defp sign(total) when total < 0, do: "-"
  defp sign(_),                    do: "+"
end
```

Now we can test our logging module! Let's update our `config.exs` file to use it:

```elixir
# config/config.exs

use Mix.Config

config :logger,
  backends: [
    {JsonLogger.Console, :console}
  ]

config :logger, :console,
  level: :info
```

... and then quickly test it with `iex -S mix`:

![Console output](/img/blog/elixir-logstash-json/console_loggerbackend.png)

Nice! Not the most human readable format though.

The complete module, with some extra sprinkles on top, can be found [here](https://github.com/svetob/logstash-json/blob/5f1fdce838b2b6e1732e63c920454409d73e4e9b/lib/logstash_json_console.ex).

# Sending logs to Logstash via TCP

Now, instead of printing to console, we want to sent these logs to Logstash.

## Setting up Logstash with a JSON consumer

First things first!

To make this easy, here is a docker compose setup which starts Logstash locally, set up to read JSON input over TCP.

`docker-compose.yml`
```
logstash:
  image: docker.elastic.co/logstash/logstash:6.3.1
  volumes:
    - "./docker/logstash.conf:/usr/share/logstash/pipeline/logstash.conf"
  ports:
    - "5044:5044"
  environment:
    XPACK_MONITORING_ENABLED: "false"
```

`docker/logstash.conf`
```
# docker/logstash.conf

input {
  tcp {
    port => 5044
    codec => json
  }
}
output {
  stdout {
    codec => rubydebug
  }
}
```

Create these two files and run `docker-compose up`. Now you have a running Logstash instance, listening to JSON messages at TCP port 5044.

## Connection

Now we can begin building our TCP connection. Logstash's TCP interface is very simple, all we need to do is open a TCP socket and send newline-delimited JSON messages. But, we also need to nicely handle connection failures, service being unavailable and other expected errors. This should be a common problem, so perhaps there is already a solution available?

Yup - [Connection](https://github.com/fishcakez/connection)! This library is a behaviour for connection processes. It will handle connection, disconnection, attempt reconnection on errors and has an optional backoff between attempts. It even comes with a [TCP connection example](https://github.com/fishcakez/connection/blob/master/examples/tcp_connection/lib/tcp_connection.ex) right out of the box. Just what we need! We will base our work on this example.

You can test the example as is and see your logs arrive in logstash:

![TCPConnection to Logstash](/img/blog/elixir-logstash-json/console_tcpconnection.png)

Let's modify our code to use this. Let's copy `lib/jsonlogger_console.ex` and create a new module, `JsonLogger.TCP` in `lib/jsonlogger_tcp.ex`. The first step is to launch a TCP connection to our logstash host, with configurable host/port.

```elixir
# lib/jsonlogger_tcp.ex

defmodule JsonLogger.TCP do

  # ...

  defp configure(name, opts) do
    env = Application.get_env(:logger, name, [])
    opts = Keyword.merge(env, opts)
    Application.put_env(:logger, name, opts)

    level = Keyword.get(opts, :level, :info)
    host = Keyword.get(opts, :host)
    port = Keyword.get(opts, :port)
    connection = Keyword.get(opts, :connection)

    # Close previous connection
    if connection != nil do
      :ok = TCPConnection.close(connection)
    end

    {:ok, connection} = TCPConnection.start_link(host, port, [active: false, mode: :binary])

    %{level: level, name: name, connection: connection}
  end
```


Then we edit `log/2` to send the message to our TCP connection genserver:

```elixir
# lib/jsonlogger_tcp.ex

  defp log(event, state) do
    case Poison.encode(event) do
      {:ok, msg} ->
        TCPConnection.send(state.connection, msg <> "\n")

      {:error, reason} ->
        IO.puts "Serialize error: #{inspect reason}, event: #{inspect event}"
    end
  end
```

...and update our `config.exs`:

```elixir
# config/config.exs

use Mix.Config

config :logger,
  backends: [
    {JsonLogger.Console, :json},
    {JsonLogger.TCP, :logstash}
  ]

config :logger, :json,
  level: :info

config :logger, :logstash,
  level: :debug,
  host: 'localhost',
  port: 5044
```

Note for `host: 'localhost'` that we use single quotes (_iolist_), not double quotes (_binary_) because this is what `:gen_tcp` expects.

Now, you should be able to send logs and see them in Logstash's output.

## Pooling and buffering

So far this works well, but it won't handle high throughput in a good way. There is only one connection, which limits log delivery speed. There is also no buffer on log messages, so any sudden increase in log volume will immediately throttle the application.

This also brings us to the topic of handling large log volumes. If we produce logs faster than our backend can handle them, or if Logstash becomes temporarily unavailable, we will be faced with more logs than we can send or keep in memory.

There is no middle ground here - if Logstash becomes unavailable or we produce too much logs too fast, we will have to either __drop logs__ or risk __blocking the application__ until we can successfully send more logs. Dropping logs when your message buffer fills up is normal. For our use case, it was important not to lose any logs, so we implemented blocking behaviour.

Thus our next and final step is to create a pool of TCP connections, which read messages from a [BlockingQueue](https://github.com/joekain/BlockingQueue). Sizing the connection pool right will increase our throughput to Logstash, and the queue will act as a buffer to handle varying log volumes.

We will need to modify our TCPConnection to read messages from a queue. We can do this by creating a worker process which reads from the queue and sends it to the TCPConnection process.

Copy the TCPConnection example to your project and add this module to it:

```elixir
# lib/connection/tcp.ex

defmodule TCPConnection.Worker do
  @moduledoc """
  Worker that reads log messages from a BlockingQueue and writes them to
  Logstash using a TCP connection.
  """

  def start_link(conn, queue) do
    spawn_link(fn -> consume_messages(conn, queue) end)
  end

  defp consume_messages(conn, queue) do
    msg = BlockingQueue.pop(queue)
    TCPConnection.send(conn, msg)
    consume_messages(conn, queue)
  end
end
```

Then we just need to start this worker process when the `TCPConnection` module intializes, with the following edits:

```elixir
# lib/connection/tcp.ex

defmodule TCPConnection do

  ...

  def start_link(host, port, queue, opts \\ [], timeout \\ 5000) do
    Connection.start_link(__MODULE__, {host, port, queue, opts, timeout})
  end

  def init({host, port, queue, opts, timeout}) do
    TCPConnection.Worker.start_link(self(), queue)

    state = %{host: host, port: port, opts: opts, timeout: timeout, sock: nil}
    {:connect, :init, state}
  end
```

If you would rather drop logs than block your application, you can change the BlockingQueue to any queue implementation which drops new messages when it hits max size.

With these changes ready, we move back to `JsonLogger.TCP` to create a queue and a connection pool when the logger backend is started:

```elixir
# lib/jsonlogger_tcp.ex

  # Standard tcp_connection socket options
  @connection_opts [active: false, mode: :binary]

  defp configure(name, opts) do
    env = Application.get_env(:logger, name, [])
    opts = Keyword.merge(env, opts)
    Application.put_env(:logger, name, opts)

    level = Keyword.get(opts, :level, :info)
    host = Keyword.get(opts, :host)
    port = Keyword.get(opts, :port)
    queue = Keyword.get(opts, :queue) || nil
    buffer_size = Keyword.get(opts, :buffer_size) || 10_000
    workers = Keyword.get(opts, :workers) || 4
    worker_pool = Keyword.get(opts, :worker_pool) || nil

    # Create new queue
    if queue == nil do
      {:ok, queue} = BlockingQueue.start_link(buffer_size)
    end

    # Close previous worker pool
    if worker_pool != nil do
      :ok = Supervisor.stop(worker_pool)
    end

    # Create worker pool
    children = 1..workers |> Enum.map(& tcp_worker(&1, host, port, queue))
    {:ok, worker_pool} = Supervisor.start_link(children,
      [strategy: :one_for_one])

    # Store opts in application env
    opts = Keyword.merge(opts, [queue: queue, worker_pool: worker_pool])
    Application.put_env(:logger, name, opts)

    %{level: level, name: name, queue: queue}
  end

  defp tcp_worker(id, host, port, queue) do
    Supervisor.Spec.worker(TCPConnection,
      [host, port, queue, @connection_opts], id: id)
  end
```

Now, if you start your application and open the BEAM observer, you should see your queue and connections up and running in the process tree.

```
$ iex -S mix

iex(1)> :observer.start()
:ok
```

![Application tree](/img/blog/elixir-logstash-json/observer_tcpworkers.png)

Finally, we again edit `log/2`, this time to push logs to our queue:

```elixir
# lib/jsonlogger_tcp.ex

  defp log(event, state) do
    case Poison.encode(event) do
      {:ok, msg} ->
        BlockingQueue.push(state.queue, msg <> "\n")

      {:error, reason} ->
        IO.puts "Serialize error: #{inspect reason}, event: #{inspect event}"
    end
  end
```


And we are done! Now you will see your JSON logs both on your console and appearing in Logstash's output.

# Conclusion

We saw how to implement an Elixir Logger backend. We used [Connection](https://github.com/fishcakez/connection) and its TCP connection example to send logs to Logstash as JSON via TCP. Finally we made our library faster and more resilient by creating a pool of TCP connection workers, reading messages from a [BlockingQueue](https://github.com/joekain/BlockingQueue) message buffer.

Thanks for reading!

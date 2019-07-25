---
title: "Elixir Findings: Asynchronous Task Streams"
date: 2019-07-24T09:56:13+01:00
draft: true
---

The other day I was solving an Exercism.io exercise that involved calculating
the frequency of letters in multiple texts in parallel using only a specific
number of "workers". This same exercise helped me stumble into
`Task.async_stream/3` and `Task.async_stream/5`.

For those unfamiliar with the `Task` module here's the description you can find on Elixir's documentation:

> Conveniences for spawning and awaiting tasks.
> Tasks are processes meant to execute one particular action throughout their lifetime, often with little or no communication with other processes. The most common use case for tasks is to convert sequential code into concurrent code by computing a value asynchronously.

Let's say we want to increase a list of numbers by 1 in parallel, up until now I'd mainly use the Task module in the following way to accomplish this task:

```elixir
1..10
|> Enum.map(fn(number) -> Task.async(fn -> number + 1 end) end)
|> Enum.map(fn(task) -> Task.await(task) end)
```

This is a little cumbersome because you basically have to define a nested anonymous function, the one inside the first call to `Enum.map`, and you have to basically make two function calls to start the task and fetch the result. First, we create the task with `Task.async`, and then we fetch the result using `Task.await`.

## Simplicity

Luckily, `Task.async_stream` let us do this in a more simple, cleaner way.

Using `Task.async_stream` will allow you to create a stream of asynchronous
tasks, where each task will run a specific function on each element of the
provided enumerable, in this case, the list of numbers.

Here's how you'd implement the same example shown above but using the
`Task.async_stream/3` function:

```elixir
Task.async_stream(1..10, fn(number) -> number + 1 end)
```

Here's what I'd say is the first main advantage you get by using
`Task.async_stream`, you can see that the code looks way cleaner, and is
simpler to reason about, at least for this specific example.

Furthermore, if you wish you can pass the reference to a given function in
`Task.async_stream` instead of providing an anonymous function, for example:

```elixir
defmodule Increaser do
  def increase(number) do
    number + 1
  end
end
```

## Lazy Evaluation

If you run the previous example on iex you'll get output similar to this:

```elixir
#Function<1.111840141/2 in Task.build_stream/3>
```

This is because `Task.async_stream`, contrary to `Task.await`, uses lazy
evaluation, which means that the tasks won't be executed until you actually
need their results. However, you won't need to call `Task.await` this time
around.

In order to get the result of running the tasks, you can use something like
`Enum.to_list/1`:

```elixir
1..10
|> Task.async_stream(fn(number) -> number + 1 end)
|> Enum.to_list()
```

Running this on iex will return the following output:

```elixir
[ok: 2, ok: 3, ok: 4, ok: 5, ok: 6, ok: 7, ok: 8, ok: 9, ok: 10, ok: 11]
```

You might be wondering why the final result is a Keyword list, however, the
result is actually a list of tuples. Every task returns a tuple where the first element is either `:ok` or `:error` and the second element is the result of the
function call, or `:timeout` as we'll see in a future section.

With this in mind if you want to retrieve only the result of the function calls
you can instead run:

```elixir
1..10
|> Task.async_stream(fn(number) -> number + 1 end)
|> Enum.map(fn({:ok, result}) -> result end)
```

This will return the following list:

```elixir
[2, 3, 4, 5, 6, 7, 8, 9, 10, 11]
```

## Concurrency

Another feature that `Task.async_stream` provides is to define how many tasks
you want to run in parallel at any given time. You can define this by specifying a value for `:max_concurrency` on the function options, which defaults to
`System.schedulers_online/0`.

Here's an example that will let us confirm this behaviour, allowing only 2 tasks to be run in parallel at any given time:

```elixir
defmodule Processor do
  def process(number) do
    :timer.sleep(1000)
    {number, :os.system_time(:millisecond)}
  end
end

1..10
|> Task.async_stream(&Processor.process/1, max_concurrency: 2)
|> Enum.to_list()
```


In the example above, on `Processor.process/1` we're returning a tuple where the first element is the argument that was provided to the function and the second
one is the current time in milliseconds.

Using a `:max_concurrency` of 2 should let us see that, for example, the time
differences between number 1 and number 3 should be of roughly 1000
milliseconds, the same goes for between 2 and 4, 6 and 8, etc.

If you run this example in iex you'll get output similar to this:

```elixir
[
  ok: {1, 1549319667100},
  ok: {2, 1549319667100},
  ok: {3, 1549319668102},
  ok: {4, 1549319668102},
  ok: {5, 1549319669102},
  ok: {6, 1549319669102},
  ok: {7, 1549319670103},
  ok: {8, 1549319670103},
  ok: {9, 1549319671104},
  ok: {10, 1549319671104}
]
```

We can confirm that the result is what we were expecting. In short, seeing as
we set `:max_concurrency` to 2, the tasks are processed sort of like in
"bundles", 1 is processed at the same time as 2, 3 is processed at the same
time as 4, 5 is processed at the same time as 6, etc.

{{<figure src="/elixir_findings.png" alt="Task Execution Time" caption="Task Execution Times">}}

Notice that this only happens because the time to execute `Processor.process/1`
is always the same, the tasks are not dependent on one another, so no task
waits for another to finish.

## Timeouts

You can limit the runtime of the tasks that are created with `Task.async_stream`
, the default value is of 5000 milliseconds. I'm going to user `:timer.sleep` with a value greater than 5000 (the default timeout value) to force a task to timeout and see what's the behaviour when that happens:

```elixir
defmodule Processor do
  def process(number) do
    :timer.sleep(6000)
  end
end

1..10
|> Task.async_stream(&Processor.process/1)
|> Enum.to_list()
```

If you run this example on iex you can confirm that the process that spawned the tasks exits with an output like:

```elixir
** (exit) exited in: Task.Supervised.stream(5000)
    ** (EXIT) time out
    (elixir) lib/task/supervised.ex:276: Task.Supervised.stream_reduce/7
    (elixir) lib/enum.ex:3015: Enum.reverse/1
    (elixir) lib/enum.ex:2649: Enum.to_list/1
```

In this case, it doesn't matter which task times out, if one of the task times
out the process exits, even if a number of tasks completed successfully. If we
don't want the task to timeout we can specify the `:timeout` value in the
`Task.async_stream/3` function options like so:

```elixir
defmodule Processor do
  def process(number) do
    :timer.sleep(6000)
  end
end

1..10
|> Task.async_stream(&Processor.process/1, timeout: 7000)
|> Enum.to_list()
```


However, `Task.async_stream/3` allows us to specify different behaviours for
when tasks time out by changing the value of `:on_timeout` on the function
options.

Here's the list of values that can be used on `:on_timeout`:

* `:exit` - This is the default value. In this situation the process that spawned the task exits
* `:kill_task` - In this case, only the task that timed out is killed. If the first task in the stream fails the others might be able to still finish. The value emitted for this task is the tuple `{:exit, :timeout}`

Let's check how using `:kill_task` only terminates the task that times out
while letting the other tasks finish:

```elixir
defmodule Processor do
  def process(number) do
    :timer.sleep(10000 - (number * 1000))
    number
  end
end

1..10
|> Task.async_stream(&Processor.process/1, on_timeout: :kill_task)
|> Enum.to_list()
```

In the example above we see that for each number we make the following call 
`:timer.sleep(10000 - (number * 1000))`. Provided that our list of numbers is
from 1 to 10 here's what the value passed into `:timer.sleep` is for each number:

* 1 - 9000 milliseconds
* 2 - 8000 milliseconds
* 3 - 7000 milliseconds
* 4 - 6000 milliseconds
* 5 - 5000 milliseconds
* 6 - 4000 milliseconds
* 7 - 3000 milliseconds
* 8 - 2000 milliseconds
* 9 - 1000 milliseconds
* 10 - 0 milliseconds

Taking into account that the default timeout value is of 5000 milliseconds than
we should see that only the tasks for 6 or greater are completed (5 shouldn't be completed because of the overhead of setting up the task).

Running the example on `iex` yields the following result:

```elixir
[
  exit: :timeout,
  exit: :timeout,
  exit: :timeout,
  exit: :timeout,
  exit: :timeout,
  ok: 6,
  ok: 7,
  ok: 8,
  ok: 9,
  ok: 10
]
```

This is what we expected, only the tasks with a number greater than or equal to
6 were terminated while the other ones were killed because they timed out.
You can confirm this because the result for the tasks that were started with a
number less than or equal to 5 all returned the tuple `{:error, :timeout}`.

With this in mind use `on_timeout: :kill_task` whenever you want to allow tasks
to finish independently of other tasks timing out.

## Conclusion

In conclusion, I think `Task.async_stream` allows you to apply a certain
function call, in parallel, to an enumerable without having to make calls to
`Task.async` and `Task.await` which, in my opinion, leads to cleaner code.

On the other hand, you also get different timeout behaviours, which is not
possible using `Task.await`, although `Task.await` also allows you to specify
timeouts.

Finally, you also get the possibility of specifying the maximum number of tasks
you want to run in parallel at any given time. I think this can be super useful, for example, in a situation where you're making multiple requests to a given
service and you want to control the load you're putting into it, without
compromising its availability.

If you want to check the full documentation for both `Task.async_stream/3` and `Task.async_stream/5` check the [Elixir Docs](https://hexdocs.pm/elixir/Task.html?source=post_page---------------------------#async_stream/3).




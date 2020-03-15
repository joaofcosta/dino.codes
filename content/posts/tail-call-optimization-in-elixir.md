---
title: "Tail Call Optimization in Elixir"
date: 2020-03-09T19:14:45Z
draft: true
description: "Blog post describing how Tail Call Optimization in Elixir can help reduce memory
footprint."
---

```elixir
defmodule Benchmark do
  def measure(function) do
    function
    |> :timer.tc
    |> elem(0)
    |> Kernel./(1_000_000)
  end
end

defmodule Factorial do
  def factorial(0), do: 1
  def factorial(n), do: n * factorial(n - 1)
end

defmodule TailFactorial do
  def factorial(n, accumulator \\ 0)
  def factorial(0, accumulator), do: accumulator
  def factorial(n, accumulator), do: factorial(n - 1, accumulator * n)
end
```

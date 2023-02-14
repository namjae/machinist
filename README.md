# Machinist

[![CI](https://github.com/norbajunior/machinist/actions/workflows/ci.yml/badge.svg)](https://github.com/norbajunior/machinist/actions/workflows/ci.yml)
[![Hex.pm Version](https://img.shields.io/hexpm/v/machinist?color=blueviolet)](https://hex.pm/packages/machinist)

This small library allows you to implement finite-state machines
with Elixir in a simple way. It provides a simple DSL to write combinations of
transitions based on events.

* [Installation](#Installation)
* [Usage](#Usage)

## Installation

You can install `machinist` by adding it  to your list of dependencies in `mix.exs`:

```elixir
def deps do
  [
    {:machinist, "~> 2.1"}
  ]
end
```

## Usage

<!-- MDOC -->

A good example is how we would implement the behaviour of a door. With
`machinist` would be this way:

```elixir
defmodule Door do
  defstruct [state: :locked]

  use Machinist

  transitions do
    from :locked,   to: :unlocked, event: "unlock"
    from :unlocked, to: :locked,   event: "lock"
    from :unlocked, to: :opened,   event: "open"
    from :opened,   to: :closed,   event: "close"
    from :closed,   to: :opened,   event: "open"
    from :closed,   to: :locked,   event: "lock"
  end
end
```

By defining these rules with `transitions` and `from` macros, `machinist`
generates and injects into the module `Door`, `transit/2` functions like this
one:

```elixir
def transit(%Door{state: :locked} = struct, event: "unlock") do
  {:ok, %Door{struct | state: :unlocked}}
end
```

So that we can transit between states by relying on the **state** + **event**
pattern matching.

Let's see this in practice:

By default, our `Door` is `locked`:

```elixir
iex> door_locked = %Door{}
%Door{state: :locked}
```

So let's change its state to `unlocked` and `opened`:

```elixir
iex> {:ok, door_unlocked} = Door.transit(door_locked, event: "unlock")
{:ok, %Door{state: :unlocked}}
iex> {:ok, door_opened} = Door.transit(door_unlocked, event: "open")
{:ok, %Door{state: :opened}}
```

If we try to make a transition that does not follow the rules, we get an error:

```elixir
iex> Door.transit(door_opened, event: "lock")
{:error, :not_allowed}
```

## Guard conditions

We could also implement a state machine for an electronic door which should validate a passcode to unlock it. In this scenario, the `machinist` allows us to provide a function to evaluate a condition and return the new state.

Check out the diagram below representing it:

![state-machine-diagram](./assets/check-passcode.png)

And to have this condition for the `unlock` event, use the `event` macro passing the `guard` option with a one-arity function:

```elixir
# ..
transitions do
  event "unlock", guard: &check_passcode/1 do
    from :locked, to: :unlocked
    from :locked, to: :locked
  end
end

defp check_passcode(door) do
  if some_condition do
    :unlocked
  else
    {:error, "invalid passcode"}
  end
end
```

So when we call `Door.transit(%Door{state: :locked}, event: "unlock")` the guard function `check_passcode/1` will be called with the struct door as the first parameter and returns the new state to be set or a transition error tuple.

### Setting a different attribute name that holds the state

By default, `machinist` expects the struct being updated to hold a `state`
attribute, if you have state in a different attribute, pass the name as an
atom, as follows:

```elixir
transitions attr: :door_state do
  # ...
end
```

And then `machinist` will set the state in that attribute.

```elixir
iex> Door.transit(door, event: "unlock")
{:ok, %Door{door_state: :unlocked}}
```

### Implementing different versions of a state machine

Let's suppose we want to build a selection process app that handles
applications of candidates, and they may go through different
versions of the process. For example:

A Selection Process **V1** with the following sequence of stages:
[Registration] -> [**Code test**] -> [Enrollment]

And a Selection Process **V2** with these ones: [Registration] ->
[**Interview**] -> [Enrollment]

The difference here is in **V1** candidates must take a **Code Test** and V2 an
**Interview**.

So, we could have a `%Candidate{}` struct that holds these attributes:

```elixir
defmodule SelectionProcess.Candidate do
  defstruct [:name, :state, test_score: 0]
end
```

And a `SelectionProcess` module that implements the state machine. Notice this
time we don't want to implement the rules in the module that holds the state,
in this case, it makes more sense for the `SelectionProcess` to keep the rules, also
because we want more than one state machine version handling candidates as
mentioned before. This is our **V1** of the process:

```elixir
defmodule SelectionProcess.V1 do
  use Machinist

  alias SelectionProcess.Candidate

  @minimum_score 70

  transitions Candidate do
    from :new,           to: :registered,    event: "register"
    from :registered,    to: :started_test,  event: "start_test"

    event "send_test", guard: &check_score/1 do
      from :started_test, to: :approved
      from :started_test, to: :reproved
    end

    from :approved, to: :enrolled, event: "enroll"
  end

  defp check_score(%Candidate{test_score: score}) do
    if score >= @minimum_score, do: :approved, else: :reproved
  end
end
```

In this code, we pass the `Candidate` module as a parameter to `transitions` to
tell `machinist` that we expect `V1.transit/2` functions with a `%Candidate{}`
struct as first argument and not the `%SelectionProcess.V1{}` which would be by
default.

```elixir
def transit(%Candidate{state: :new} = struct, event: "register") do
  {:ok, %Candidate{struct | state: :registered}}
end
```

Also notice we provided the *function* `&check_score/1` to the option `to:`
instead of an *atom*, to decide the state based on the candidate
`test_score` value.

In **version 2**, we replaced the `Code Test` stage with the `Interview`
which has different state transitions:

```elixir
defmodule SelectionProcess.V2 do
  use Machinist

  alias SelectionProcess.Candidate

  transitions Candidate do
    from :new,                 to: :registered,          event: "register"
    from :registered,          to: :interview_scheduled, event: "schedule_interview"
    from :interview_scheduled, to: :approved,            event: "approve_interview"
    from :interview_scheduled, to: :repproved,           event: "reprove_interview"
    from :approved,            to: :enrolled,            event: "enroll"
  end
end
```

Now let's see how we can test it:

**V1:** A `registered` candidate wants to start his test.

```elixir
iex> candidate1 = %Candidate{name: "Ada", state: :registered}
iex> SelectionProcess.V1.transit(candidate1, event: "start_test")
%{:ok, %Candidate{state: :test_started}}
```

**V2:** A `registered` candidate wants to schedule the interview

```elixir
iex> candidate2 = %Candidate{name: "Jose", state: :registered}
iex> SelectionProcess.V2.transit(candidate1, event: "schedule_interview")
%{:ok, %Candidate{state: :interview_scheduled}}
```

That's great because we also can implement many state machines for only one
entity and test different scenarios, evaluate and collect data for deciding
which one is better.

`machinist` gives us this flexibility since it's just pure Elixir.

### Transiting from any state to another

Sometimes we need to define a `from` _any state_ transition.

Still, in the selection process example, candidates can abandon the process in
a given state, and we want to be able to transit them to
`application_expired` from any state. To do so, we just define a `from` with an
underscore variable for the current state to be ignored.

```elixir
defmodule SelectionProcess.V2 do
  use Machinist

  alias SelectionProcess.Candidate

  transitions Candidate do
    # ...
    from _state, to: :application_expired, event: "application_expired"
  end
end
```

### Code formatter

Elixir formatter (`mix format`) puts parenthesis around the macros.

```elixir
from(:some_state, to: :another, event: "some_event")

from :some_state do
  to(:another, event: "some_event")
end
```

However, the machinist's`.formatter.exs` is configured to not use parenthesis. In order to follow the code style without parenthesis you can export the machinist config in your project.

In your `.formatter.exs` file, just add the following:

```elixir
[
  # ...
  import_deps: [:machinist]
]
```

And you're good to go 🧙‍♀️.

## How does the DSL works?

The use of `transitions` in combination with each `from` statement will be
transformed in functions that will be injected into the module that is using
`machinist`.

This implementation:

```elixir
defmodule Door do
  defstruct state: :locked

  use Machinist

  transitions do
    from :locked,   to: :unlocked, event: "unlock"
    from :unlocked, to: :locked,   event: "lock"
    from :unlocked, to: :opened,   event: "open"
    from :opened,   to: :closed,   event: "close"
    from :closed,   to: :opened,   event: "open"
    from :closed,   to: :locked,   event: "lock"
  end
end
```

It is the same as:

```elixir
defmodule Door do
  defstruct state: :locked

  def transit(%__MODULE__{state: :locked} = struct, event: "unlock") do
    {:ok, %__MODULE__{struct | state: :unlocked}}
  end

  def transit(%__MODULE__{state: :unlocked} = struct, event: "lock") do
    {:ok, %__MODULE__{struct | state: :locked}}
  end

  def transit(%__MODULE__{state: :unlocked} = struct, event: "open") do
    {:ok, %__MODULE__{struct | state: :opened}}
  end

  def transit(%__MODULE__{state: :opened} = struct, event: "close") do
    {:ok, %__MODULE__{struct | state: :closed}}
  end

  def transit(%__MODULE__{state: :closed} = struct, event: "open") do
    {:ok, %__MODULE__{struct | state: :opened}}
  end

  def transit(%__MODULE__{state: :closed} = struct, event: "lock") do
    {:ok, %__MODULE__{struct | state: :locked}}
  end
  # a catchall function in case of unmatched clauses
  def transit(_, _), do: {:error, :not_allowed}
end
```

So, as we can see, we can eliminate a lot of boilerplate with `machinist`
making it easier to maintain and less prone to errors.

<!-- MDOC -->

## Contributing

Feel free to contribute to this lib. If you have any suggestions or bug reports
just open an issue or a PR.

## License

[MIT License](https://github.com/norbajunior/machinist/blob/main/LICENSE)

## Donate

If this project helps you reduce time to develop, you can give me a cup of coffee :)

<a href="https://www.paypal.com/donate/?hosted_button_id=LVPBU9CT3Z7DN" target="_blank">
  <img src="assets/donate-btn.svg" height="40">
</a>

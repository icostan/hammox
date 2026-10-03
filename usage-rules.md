# Hammox Usage Rules

## Overview
Hammox provides automated dynamic contract testing for Elixir mocks and implementations based on behaviour typespecs. It is fully backwards-compatible with Mox, enforcing that mock invocations, arguments, and return values strictly adhere to declared callback specs while also enabling real implementations to be decorated with identical runtime contract checks.

## Setup
Remove `:mox` from `mix.exs` if present (Hammox bundles and re-exports Mox), and add `:hammox` to test dependencies:

```elixir
def deps do
  [
    {:hammox, "~> 1.0", only: :test}
  ]
end
```

## Core Usage Patterns

- **Drop-in replacement for Mox**: Replace `import Mox` with `import Hammox` in test files. Use `Hammox.defmock/2`, `Hammox.expect/4`, `Hammox.stub/3`, and `Hammox.deny/3`.
  ```elixir
  Hammox.defmock(DatabaseMock, for: Database)
  Hammox.expect(DatabaseMock, :get_users, fn -> ["joe", "jim"] end)
  ```
- **Explicitly deny mock calls**: Use `Hammox.deny/3` to assert that a mock callback is never called with the given arity:
  ```elixir
  Hammox.deny(DatabaseMock, :delete_user, 1)
  ```
- **Protect real implementations against contracts**: Use `Hammox.protect/2` with an MFA tuple to decorate real implementation functions with dynamic typespec checks:
  ```elixir
  get_users = Hammox.protect({RealDatabase, :get_users, 0}, Database)
  assert {:ok, ["alice"]} == get_users.()
  ```
- **Batch-protect implementations in test setup**: Pass implementation and behaviour modules to `Hammox.protect/2` in `setup_all` to generate context maps with keys named `:<function>_<arity>`:
  ```elixir
  setup_all do
    Hammox.protect(RealDatabase, Database)
  end

  test "returns users", %{get_users_0: get_users_0} do
    assert {:ok, ["alice"]} == get_users_0.()
  end
  ```
- **Protect across multiple behaviours**: Pass a list of behaviours as the second argument to `Hammox.protect/2` to merge all callbacks into a single map:
  ```elixir
  Hammox.protect(TestCalculator, [Calculator, AdditionalCalculator])
  ```
- **Use single-module shortcuts when behaviour and implementation coincide**: If a module defines both callbacks and implementations, omit the behaviour argument:
  ```elixir
  Hammox.protect({Calculator, :add, 2})
  Hammox.protect(Calculator)
  ```
- **Filter protected functions**: Pass an explicit keyword list of functions and arities to `Hammox.protect/3`:
  ```elixir
  Hammox.protect(RealDatabase, Database, get_users: 0, find_user: 1)
  ```
- **Import protected functions via `use Hammox.Protect`**: In test modules, define protected delegate functions directly in the test scope instead of passing context maps:
  ```elixir
  use Hammox.Protect, module: RealDatabase, behaviour: Database

  test "fetches users" do
    assert {:ok, ["alice"]} == get_users()
  end
  ```
- **Bypass contract checks when necessary**: Hammox interoperates with Mox. Call `Mox.expect/4` or `Mox.stub/3` directly on instances where dynamic typespec enforcement must be skipped.

## Configuration

- **Telemetry**: Disabled by default via a no-op handler. Enable it in `config/test.exs` to diagnose test suite performance bottlenecks:
  ```elixir
  config :hammox, enable_telemetry?: true
  ```
- **Supported Telemetry Spans**: Hammox emits `[:hammox, <action>, :start | :stop | :exception]` spans for `:expect`, `:allow`, `:run_expect`, `:check_call`, `:match_args`, `:match_return_value`, `:fetch_typespecs`, `:cache_put`, `:stub`, `:verify_on_exit!`, and `:deny`.

## Common Mistakes to Avoid

- **Keeping `:mox` in dependencies**: Do not include both `{:mox, ...}` and `{:hammox, ...}` in `mix.exs`. Hammox depends on Mox directly.
- **Assuming function typespec arguments are checked**: For anonymous function types (`(arg -> return)`), Hammox only checks arity, not argument types or return types.
- **Passing non-implementing values for protocol types**: Typespecs referencing `Protocol.t()` (e.g., `Enumerable.t()`) dynamically check `Protocol.impl_for/1`. Values not implementing the protocol raise `Hammox.TypeMatchError`.
- **Misnaming context keys from batch protection**: `Hammox.protect/2,3` appends the arity to the function name (e.g., `%{get_users_0: ...}`, `%{find_user_1: ...}`). Matching without `_<arity>` will fail.
- **Using `Hammox.Protect` without callbacks**: `use Hammox.Protect` raises `ArgumentError` if `:module` is missing or if the target behaviour defines no `@callback` attributes.
- **Functions with arity > 253**: Protected functions cannot exceed arity 253 due to BEAM's 255 argument limit combined with internal closure state.
- **Mocks without typespecs**: Invocations to functions without typespecs on the behaviour pass through to Mox and will not be validated by Hammox.

## Testing

- **Define mocks in `test/test_helper.exs`**:
  ```elixir
  Hammox.defmock(MyApp.WeatherMock, for: MyApp.Weather)
  ```
- **Verify mock expectations on exit**: Call `verify_on_exit!/1` in test callbacks or case templates:
  ```elixir
  setup :set_mox_from_context
  setup :verify_on_exit!
  ```
- **Handle asynchronous tests**: Use `Hammox.set_mox_from_context/1` with `async: true` tests, or explicit allowances via `Hammox.allow/3`:
  ```elixir
  Hammox.allow(WeatherMock, self(), task_pid)
  ```

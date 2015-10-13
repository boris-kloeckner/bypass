# Bypass

Bypass provides a quick way to create a custom plug that can be put in place instead of an actual
HTTP server to return prebaked responses to client requests. This is most useful in tests, when you
want to create a mock HTTP server and test how your HTTP client handles different types of
responses from the server.


## Installation

If [available in Hex](https://hex.pm/docs/publish), the package can be installed as:

  1. Add bypass to your list of dependencies in mix.exs:

     ```elixir
     def deps do
       [{:bypass, "~> 0.0.1", only: :test}]
     end
     ```

     It is not recommended to add `:bypass` to the list of applications in your `mix.exs`. See below
     for usage info.


## Usage

Start Bypass in your `test/test_helper.exs` file to make it available in tests:

```elixir
ExUnit.start
Application.ensure_all_started(:bypass)
```

To use Bypass in a test case, open a connection and use its port to connect your client to it.

If you want to test what happens when the HTTP server goes down, use `Bypass.down/1` and
`Bypass.up/1`, which guarantee that the TCP port will be closed, respective open, after returning:

In this example `SomeAPIClient` reads its endpoint URL from the `Application`'s configuration:

```elixir
defmodule MyClientTest do
  use ExUnit.Case

  setup do
    bypass = Bypass.open
    Application.put_env(:my_app, :some_api_endpoint, "http://localhost:#{bypass.port}/")
    context = %{bypass: bypass}
    {:ok, context}
  end

  test "client can handle an error response", context do
    Bypass.expect context.bypass, fn conn ->
      assert "/no_idea" == conn.request_path
      assert "POST" == conn.method
      Plug.Conn.send_resp(conn, 400, "Please make up your mind!")
    end
    {:ok, client} = SomeAPIClient.start_link()
    assert {:error, {400, "Please make up your mind!"}} == SomeAPIClient.post_no_idea(client, "")
  end
  test "client can recover from server downtime", context do
    Bypass.expect context.bypass, fn conn ->
      # We don't care about `request_path` or `method` for this test.
      Plug.Conn.send_resp(conn, 200, "ohai")
    end
    {:ok, client} = SomeAPIClient.start_link()

    assert {:ok, {200, "ohai"}} == SomeAPIClient.ping(client, "")

    Bypass.down(context[:bypass])

    assert {:error, :noconnect} == SomeAPIClient.ping(client, "")

    Bypass.up(context[:bypass])

    assert {:ok, {200, "ohai"}} == SomeAPIClient.ping(client, "")
  end
end
```

That's all you need to do. Bypass automatically sets up an `on_exit` hook to close its socket when
the test finishes running.

Multiple concurrent Bypass instances are supported, all will have a different unique port.

```elixir
defmodule MyClientTest do
  use ExUnit.Case

  setup do
    bypass = Bypass.open
    Application.put_env(:my_app, :some_api_endpoint, "http://localhost:#{bypass.port}/")
    context = %{bypass: bypass}
    {:ok, context}
  end

end
```


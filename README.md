# Pow

[![Build Status](https://travis-ci.org/danschultzer/pow.svg?branch=master)](https://travis-ci.org/danschultzer/pow) [![hex.pm](http://img.shields.io/hexpm/v/pow.svg?style=flat)](https://hex.pm/packages/pow)

Pow is a powerful, modular, and extendable authentication and user management solution for Phoenix and Plug based apps.

## Features

* User registration
* Session based authorization
* Per Endpoint/Plug configuration
* Extendable
* I18n

## Installation

Add Pow to your list of dependencies in `mix.exs`:

```elixir
def deps do
  [
    # ...
    {:pow, "~> 0.1"}
    # ...
  ]
end
```

Run `mix deps.get` to install it.

## Getting started (Phoenix)

Install the necessary files:

```bash
mix pow.install
```

This will add the following files to your app:

```bash
LIB_PATH/users/user.ex
PRIV_PATH/repo/migrations/TIMESTAMP_create_user.ex
```

Update `config/config.ex` with the following:

```elixir
config :my_app, :pow,
  user: MyApp.Users.User,
  repo: MyApp.Repo
```

Set up `WEB_PATH/endpoint.ex` to enable session based authentication:

```elixir
defmodule MyAppWeb.Endpoint do
  use Phoenix.Endpoint, otp_app: :my_app

  # ...

  plug Plug.Session,
    store: :cookie,
    key: "_my_project_demo_key",
    signing_salt: "secret"

  plug Pow.Plug.Session, otp_app: :my_app

  # ...
end
```

Add Pow routes to `WEB_PATH/router.ex`:

```elixir
defmodule MyAppWeb.Router do
  use MyAppWeb, :router
  use Pow.Phoenix.Router

  # ...

  pipeline :protected do
    plug Pow.Plug.RequireAuthenticated,
      error_handler: Pow.Phoenix.PlugErrorHandler
  end

  scope "/" do
    pipe_through :browser

    pow_routes()
  end

  # ...

  scope "/", MyAppWeb do
    pipe_through [:browser, :protected]

    # Protected routes ...
  end
end
```

That's it! Run `mix ecto.setup`, and you can now visit `http://localhost:4000/registrations/new`, and create a new user.

By default, Pow will only expose files that are absolutely necessary. If you wish to modify the templates, you can generate them (and the view files) using the `mix pow.phoenix.gen.templates` command.

## Extensions

Pow is made so it's easy to extend the functionality with your own complimentary library. The following extensions are included in this library:

* `PowResetPassword`
* `PowEmailConfirmation`

### Add extensions support

To keep it easy to understand and configure Pow, you'll have to enable the extensions yourself.

First, install extension migrations by running `mix pow.extension.ecto.gen.migrations --extension PowResetPassword --extension PowEmailConfirmation`.

Update `config/config.ex` with the `:extensions` key:

```elixir
config :my_app, :pow,
  user: MyApp.Users.User,
  repo: MyApp.Repo,
  extensions: [PowResetPassword, PowEmailConfirmation]
```

Update `LIB_PATH/users/user.ex` with the following:

```elixir
defmodule MyApp.Users.User do
  use Ecto.Schema
  use Pow.Ecto.Schema, otp_app: :my_app
  use Pow.Extension.Ecto.Schema

  # ...

  def changeset(user_or_changeset, attrs) do
    user_or_changeset
    |> pow_changeset(attrs)
    |> pow_extensions_changeset(attrs)
  end
end
```

Add Pow extension routes to `WEB_PATH/router.ex`:

```elixir
defmodule MyAppWeb.Router do
  use MyAppWeb, :router
  use Pow.Phoenix.Router
  use Pow.Extension.Phoenix.Router, otp_app: :my_app

  # ...

  scope "/" do
    pipe_through :browser

    pow_routes()
    pow_extension_routes()
  end

  # ...
end
```

### Mailer support

Many extensions requires a mailer to have been set up. Let's create the mailer in `WEB_PATH/mailer.ex` using [swoosh](https://github.com/swoosh/swoosh):

```elixir
defmodule MyAppWeb.Mailer do
  use Pow.Phoenix.Mailer
  use Swoosh.Mailer, otp_app: :my_app_web
  import Swoosh.Email

  def cast(%{user: user, subject: subject, text: text, html: html}) do
    %Swoosh.Email{}
    |> to({"", email.user.email})
    |> from({"My App", "myapp@example.com"})
    |> subject(subject)
    |> html_body(html)
    |> text_body(text)
  end

  def process(email) do
    deliver(email)
  end
end
```

Update `config/config.ex` with `:backend_mailer` key:

```elixir
config :my_app, :pow,
  user: MyApp.Users.User,
  repo: MyApp.Repo,
  extensions: [PowResetPassword, PowEmailConfirmation],
  backend_mailer: MyAppWeb.Mailer
```

That's it!

## Configuration

Pow is build to be modular, and easy to configure. Configuration is primarily passed through method calls, and plug options and they will take priority over any environment configuration. This is ideal in case you've an umbrella app with multiple separate user domains.

The most easy way to use Pow with Phoenix is to create a Pow.Phoenix module. This will keep a persistent fallback Pow configuration that you configure in one place. This is automatically generated when you run `mix pow.install`.

### Module groups

Pow has three main groups of modules that each can used individually, or in conjunction with each other:

#### Pow.Plug

This group will handle the plug connection. The configuration will be assigned to `conn.private[:pow_config]` and passed through the controller to the users context module. The Plug module have methods to authenticate, create, update, and delete users, and will generate/renew the session automatically.

#### Pow.Ecto

This group contains all modules related to the Ecto based user schema and context. By default, Pow will use the `Pow.Ecto.Context` module to authenticate, create, update and delete users with lookups to the database. However, it's very simple to extend, or write your own user context. You can do this by setting the `:users_context` configuration key.

#### Pow.Phoenix

This contains the controllers, views and templates for Phoenix. The only thing you need to set up in your own app, is the `router.ex`. Views and templates are not generated by default, but instead the compiled views and templates in Pow will be used. However, you can generate these by running `mix pow.phoenix.gen.templates`. Flash messages and routes can also be customized by creating your own using `:messsages_backend` and `:routes_backend`. The registration and session controllers can be modified too. Since the routes are compiled, you'll have to update `router.ex` if you decide to create your own controllers, but for minor differences in handling you can use the `:controller_callbacks` option. This setup exist to make it easier modify flow with extensions (e.g. send a confirmation email upon user registration).

### Pow.Extension

This module helps build extensions for Pow. There's two extension mix tasks to generate ecto migrations and phoenix templates.

```bash
mix pow.extension.ecto.gen.migrations
```

```bash
mix pow.extension.phoenix.gen.templates
```

### Authorization plug

Pow ships with a session plug module. You can easily switch it out with a different one. As an example, here's how you do that with [Guardian](https://github.com/ueberauth/guardian):

```elixir
defmodule MyAppWeb.Pow.Plug do
  use Pow.Plug.Base

  def fetch(conn, config) do
    user = MyApp.Guardian.Plug.current_resource(conn)

    {conn, user}
  end

  def create(conn, user, config) do
    conn = MyApp.Guardian.Plug.sign_in(conn, user)

    {conn, user}
  end

  def delete(conn, config) do
    MyApp.Guardian.Plug.signout(conn)
  end
end

defmodule MyAppWeb.Endpoint do
  # ...

  plug MyAppWeb.Pow.Plug,
    repo: MyApp.Repo,
    user: MyApp.Users.User
end
```

### Ecto changeset

The user module has a fallback `changeset/2` method. If you want to add custom validations, you can use the `pow_changeset/2` method like so:

```elixir
defmodule MyApp.Users.User do
  use Ecto.Schema
  use Pow.Ecto.Schema

  schema "users" do
    field :custom, :string

    pow_user_fields()

    timestamps()
  end

  def changeset(user_or_changeset, attrs) do
    user
    |> pow_changeset(attrs)
    |> Ecto.Changeset.cast(attrs, [:custom])
    |> Ecto.Changeset.validate_required([:custom])
  end
end
```

### Phoenix controllers

If you wish to change the flow of the `RegistrationController` and `SessionController`, the best way is to simply create your own and modify `router.ex`. Controllers in Pow are very slim, and consists of just one `Pow.Plug` method call, and then response/render handling.

To make it easier to integrate extension, you can add callbacks to the controllers:

```elixir
defmodule MyCustomExtension.Pow.ControllerCallbacks do
  use Pow.Extension.Phoenix.ControllerCallbacks.Base

  def before_respond(Pow.Phoenix.RegistrationController, :create, {:ok, user, conn}, _config) do
    # send email

    {:ok, user, conn}
  end
end
```

You can add methods for `before_process` (before the action happens) and `before_respond` (before parsing the results from the action).

## Plugs

### Pow.Plug.Session

Enables session based authorization. The user struct will be collected from a cache store through a GenServer using a unique token generated for the session. The token will be reset every time the authorization level changes (handled by `Pow.Plug`).

#### Cache store

By default `Pow.Store.EtsCache` is started automatically and can be used in development and test environment.

For production environment a distributed, persistent cache store should be used. Pow makes this easy with `Pow.Store.MnesiaCache`. As an example, to start MnesiaCache in your Phoenix app, you just have to set up your `application.ex`:

```elixir
defmodule MyAppWeb.Application do
  use Application

  def start(_type, _args) do
    import Supervisor.Spec

    children = [
      supervisor(MyAppWeb.Endpoint, []),
      worker(Pow.Store.MnesiaCache, [nodes: nodes()])
    ]

    opts = [strategy: :one_for_one, name: MyAppWeb.Supervisor]
    Supervisor.start_link(children, opts)
  end

  # ...
end
```

### Pow.Plug.RequireAuthenticated

Will halt connection if no current user is not found in assigns. Expects an `:error_handler` option.

### Pow.Plug.RequireNotAuthenticated

Will halt connection if a current user is found in assigns. Expects an `:error_handler` option.

## Pow security practices

* The `user_id_field` value is always treated as case insensitve
* If the `user_id_field` is `:email`, it'll be validated based on RFC 5322 (excluding IP validation)
* The `:password` has a minimum length of 10 characters
* The `:password` has a maximum length of 4096 bytes [to prevent DOS attacks against Pbkdf2](https://github.com/riverrun/pbkdf2_elixir/blob/master/lib/pbkdf2.ex#L21)
* The `:password_hash` is generated with `Pbpdf2`
* The session value contains a UUID token that is used to pull credentials through a GenServer
* The credentials are stored in an ETS key-value storage with ttl of 30 minutes
* The credentials and session will be reset every 15 minutes if activity is detected
* The credentials and session are reset after user updates

Some of the above is based on [https://www.owasp.org/](OWASP) recommendations.

## LICENSE

(The MIT License)

Copyright (c) 2018 Dan Schultzer & the Contributors Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the 'Software'), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED 'AS IS', WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

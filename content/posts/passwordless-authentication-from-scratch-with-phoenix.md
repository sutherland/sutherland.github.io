+++
title = 'Passwordless Authentication from Scratch with Phoenix'
date = 2019-06-19T12:57:52.000000Z
aliases = ['/passwordless-authentication-from-scratch-with-phoenix']
+++

![](/a4934a55-812a-4c21-ba6c-852b0ff18d90.png)

In this tutorial, I'll show you how I implemented passwordless authentication with Phoenix for [Proseful](https://proseful.com), the beautifully simple blogging platform that I recently launched.

**tl;dr:** Check out the [example repo](https://github.com/sutherland/passwordless-phoenix-example).

## What is passwordless authentication?

Passwordless authentication is exactly that: an authentication flow that lets a user log in without a password. Instead of a password, some other credential is used to authenticate access. For example, a code sent via SMS or a private link sent via email. In this tutorial, we'll use email for our passwordless flow.

So, why would you want to go passwordless? First, there's no need to create and remember yet another password. This can be a convenience to your users when signing up and logging in, especially on mobile. Second, passwordless removes various attack vectors, such as weak passwords, common passwords, brute force attacks and rainbow table attacks (should your data be compromised). As well, there is likely less code to write and maintain, which means less opportunity to mess something up security-wise.

But, isn't this less secure? Passwordless (via email) is no-less secure than using passwords with an email reset flow. If someone has access to the email account, they can log in. Ideally, to help mitigate this, you'll implement two-factor authentication once it makes sense for your app.

## Requirements

Before we start coding, let's review our needs:

* Allow a user to sign up with only an email address.
* Allow a user to log in with only an email address.
* Deliver private login links via email.
* Prevent login links from being used more than once.
* Expire login links if left unused (e.g. after 15 minutes).
* Record and remember a user's session once logged in.
* Allow a user to log out.

## Setup

To do this from scratch, we'll start with a fresh Phoenix app. I'm using Phoenix 1.4.8.

```
mix phx.new passwordless
```

Follow Phoenix's prompts and instructions to complete the setup and start the app server.

## Home page

We'll create a simple home page as a starting point. Create a controller file at `lib/passwordless_web/controllers/home_controller.ex`:

```
defmodule PasswordlessWeb.HomeController do
  use PasswordlessWeb, :controller

  def index(conn, _params) do
    render(conn, "index.html")
  end
end
```

Create a view at `lib/passwordless_web/views/home_view.ex`:

```
defmodule PasswordlessWeb.HomeView do
  use PasswordlessWeb, :view
end
```

Create a template at `lib/passwordless_web/templates/home/index.html.exs`:

```
<h1>Home</h1>

<%= link "Sign up", to: "#todo" %>
```

Note that we'll update the link with the correct path later. I use `to: "#todo"` elsewhere in this tutorial as well.

And update the router:

```
scope "/", PasswordlessWeb do
  pipe_through :browser

  get "/", HomeController, :index
end
```

Visit [http://localhost:4000](http://localhost:4000) to ensure the home page works.

## User schema

Next, we'll need a `User` schema for our users. To keep things simple, we'll add only an `email` field to our user records.

Generate a migration:

```
mix ecto.gen.migration create_users
```

Edit the migration:

```
defmodule Passwordless.Repo.Migrations.CreateUsers do
  use Ecto.Migration

  def up do
    execute "CREATE EXTENSION IF NOT EXISTS citext"

    create table(:users) do
      add :email, :citext, null: false

      timestamps()
    end

    create unique_index(:users, \[:email\])
  end

  def down do
    drop table(:users)
  end
end
```

Note that we're using the Postgres [`citext`](https://www.postgresql.org/docs/current/citext.html) extension for case-insensitive emails as a convenience. This will allow a user to capitalize their email however they please. An index is added to ensure email uniqueness.

Run the migration:

```
mix ecto.migrate
```

Let's define our `User` schema. We'll use an `Accounts` context for user functionality. Create a file at `lib/passwordless/accounts/user.ex` for the schema:

```
defmodule Passwordless.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :email, :string

    timestamps()
  end

  def changeset(user, attrs) do
    user
    |> cast(attrs, \[:email\])
    |> validate_required(\[:email\])
    |> unique_constraint(:email)
  end
end
```

## Sign up page

Next, let's add a sign up page for our users. We'll need a few methods to work with user records. Rather than dumping all of our logic into a single, bloated context file, we'll define separate modules.

Create a `Users` module at `lib/passwordless/accounts/users.ex`:

```
defmodule Passwordless.Accounts.Users do
  alias Passwordless.Accounts.User
  alias Passwordless.Repo

  def change(%User{} = user) do
    User.changeset(user, %{})
  end

  def create(attrs \\\\\\\\ %{}) do
    %User{}
    |> User.changeset(attrs)
    |> Repo.insert()
  end
end
```

Create a controller at `lib/passwordless_web/controllers/user_controller.ex`:

```
defmodule PasswordlessWeb.UserController do
  use PasswordlessWeb, :controller

  alias Ecto.Changeset
  alias Passwordless.Accounts.User
  alias Passwordless.Accounts.Users

  def new(conn, _params) do
    changeset = Users.change(%User{})

    conn
    |> assign(:changeset, changeset)
    |> render("new.html")
  end

  def create(conn, %{"user" => user_params}) do
    case Users.create(user_params) do
      {:ok, _user} ->
        conn
        |> put_flash(:info, "Signed up successfully.")
        |> redirect(to: Routes.home_path(conn, :index))

      {:error, %Changeset{} = changeset} ->
        conn
        |> assign(:changeset, changeset)
        |> render("new.html")
    end
  end
end
```

Create a view at `lib/passwordless_web/views/user_view.ex`:

```
defmodule PasswordlessWeb.UserView do
  use PasswordlessWeb, :view
end
```

Create a template at `lib/passwordless_web/templates/user/new.html.eex`:

```
<h1>Sign up</h1>

<%= form_for @changeset, Routes.user_path(@conn, :create), fn f -> %>
  <%= if @changeset.action do %>
    <div class="alert alert-danger">
      Oops, something went wrong! Please check the errors below.
    </div>
  <% end %>

  <%= label f, :email %>
  <%= text_input f, :email %>
  <%= error_tag f, :email %>

  <%= submit "Save" %>
<% end %>

<%= link "Back to home", to: Routes.home_path(@conn, :index) %>
```

Update the router:

```
scope "/", PasswordlessWeb do
  ...
  resources "/users", UserController, only: \[:create, :new\]
end
```

We also need to update the "Sign up" link in the home page template:

```
<h1>Home</h1>

<%= link "Sign up", to: Routes.user_path(@conn, :new) %>
```

Visit [http://localhost:4000](http://localhost:4000) and make sure a user record is created successfully on sign up.

## Login Request schema

Now we need a way to track requests to log in, remove these requests after they've been used, and expire them if not used. For this, we'll define a `LoginRequest` schema.

Generate a migration:

```
mix ecto.gen.migration create_login_requests
```

Edit the migration:

```
defmodule Passwordless.Repo.Migrations.CreateLoginRequests do
  use Ecto.Migration

  def change do
    create table(:login_requests) do
      add :user_id, references(:users, on_delete: :delete_all), null: false

      timestamps()
    end

    create index(:login_requests, \[:user_id\])
  end
end
```

Note that we've used [`Ecto.Migration.references/2`](https://hexdocs.pm/ecto_sql/Ecto.Migration.html#references/2) to add a foreign key constraint to ensure referential integrity of our data. We've also added an index on the `user_id` foreign key column as best practice.

Run the migration:

```
mix ecto.migrate
```

Define the `LoginRequest` schema:

```
defmodule Passwordless.Accounts.LoginRequest do
  use Ecto.Schema

  alias Passwordless.Accounts.User

  schema "login_requests" do
    timestamps()

    belongs_to :user, User
  end
end
```

And update the `User` schema with the association:

```
defmodule Passwordless.Accounts.User do
  ...

  alias Passwordless.Accounts.LoginRequest

  schema "users" do
    ...
    has_one :login_request, LoginRequest
  end

  ...
end
```

## Log in page

Next, let's add a page for our users to log in. For this, we need the ability to lookup a user by email. Update the `Users` module:

```
defmodule Passwordless.Accounts.Users do
  ...

  def get_by_email(email) do
    Repo.get_by(User, email: email)
  end
end
```

We also need the ability to create a login request for a user. Create a `LoginRequests` module at`lib/passwordless/accounts/login_requests.ex`:

```
defmodule Passwordless.Accounts.LoginRequests do
  alias Ecto.Multi
  alias Passwordless.Accounts.Users
  alias Passwordless.Repo

  def create(email) do
    with user when not is_nil(user) <- Users.get_by_email(email) do
      {:ok, changes} =
        Multi.new()
        |> Multi.delete_all(:delete_login_requests, Ecto.assoc(user, :login_request))
        |> Multi.insert(:login_request, Ecto.build_assoc(user, :login_request))
        |> Repo.transaction()

      {:ok, changes, user}
    else
      nil -> {:error, :not_found}
    end
  end
end
```

We're using [`Ecto.Multi`](https://hexdocs.pm/ecto/Ecto.Multi.html) to perform multiple operations within a database transaction. For security purposes, we delete any existing login requests belonging to the user.

Create a controller at `lib/passwordless_web/controllers/login_request_controller.ex`:

```
defmodule PasswordlessWeb.LoginRequestController do
  use PasswordlessWeb, :controller

  alias Passwordless.Accounts.LoginRequests

  def new(conn, _params) do
    render(conn, "new.html")
  end

  def create(conn, %{"login_request" => %{"email" => email}}) do
    case LoginRequests.create(email) do
      {:ok, _changes, _user} ->
        conn
        |> put_flash(:info, "We just emailed you a temporary login link. Please check your inbox.")
        |> redirect(to: Routes.home_path(conn, :index))

      {:error, :not_found} ->
        conn
        |> put_flash(:error, "Oops, that email does not exist.")
        |> render("new.html")
    end
  end
end
```

Create a view at `lib/passwordless_web/views/login_request_view.ex`:

```
defmodule PasswordlessWeb.LoginRequestView do
  use PasswordlessWeb, :view
end
```

Create a template at `lib/passwordless_web/templates/login_request/new.html.eex`:

```
<h1>Log in</h1>

<%= form_for @conn, Routes.login_request_path(@conn, :create), \[as: :login_request\], fn f -> %>
  <%= label f, :email %>
  <%= text_input f, :email %>

  <%= submit "Continue" %>
<% end %>

<%= link "Back to home", to: Routes.home_path(@conn, :index) %>
```

Update the router:

```
scope "/", PasswordlessWeb do
  ...
  resources "/login-requests", LoginRequestController, only: \[:create, :new\]
end
```

Also, let's add a "Log in" link to the home page template:

```
<h1>Home</h1>

<%= link "Sign up", to: Routes.user_path(@conn, :new) %> or
<%= link "Log in", to: Routes.login_request_path(@conn, :new) %>
```

Visit [http://localhost:4000](http://localhost:4000) and make sure that a login request record is created for a successful log in.

## Deliver login email

We need to deliver an email whenever a user wants to log in. This email includes a link with a signed token that will log the user in.

First, we need a module to sign and verify tokens for login requests. Create a `Tokens` module at `lib/passwordless/accounts/tokens.ex`:

```
defmodule Passwordless.Accounts.Tokens do
  alias Phoenix.Token
  alias PasswordlessWeb.Endpoint

  @login_request_max_age 60 \* 15 # 15 minutes
  @login_request_salt "login request salt"

  def sign_login_request(login_request) do
    Token.sign(Endpoint, @login_request_salt, login_request.id)
  end

  def verify_login_request(token) do
    Token.verify(Endpoint, @login_request_salt, token, max_age: @login_request_max_age)
  end
end
```

Note that [`Phoenix.Token.verify/4`](https://hexdocs.pm/phoenix/Phoenix.Token.html#verify/4) will return an appropriate error if the token is invalid or has expired.

We'll use [Bamboo](https://github.com/thoughtbot/bamboo) to deliver emails. Add `bamboo` to your app's dependencies in `mix.exs`:

```
defp deps do
  [
    ...,
    {:bamboo, "~> 1.2"}
  ]
end
```

Update the dependencies:

```
$ mix deps.get
```

Create a `Mailer` module at `lib/passwordless/mailer.ex`:

```
defmodule Passwordless.Mailer do
  use Bamboo.Mailer, otp_app: :passwordless
end
```

Bamboo needs to be configured with an adapter that determines how emails are delivered. We'll use the `Bamboo.LocalAdapter` for our development purposes. Update `config/config.exs`:

```
...

config :passwordless, Passwordless.Mailer, adapter: Bamboo.LocalAdapter

...
```

And restart your app server.

Create an `Email` module at `lib/passwordless/email.ex`:

```
defmodule Passwordless.Email do
  use Bamboo.Phoenix, view: PasswordlessWeb.EmailView
  import Bamboo.Email

  alias Passwordless.Accounts.Tokens

  def login_request(user, login_request) do
    new_email()
    |> to(user.email)
    |> from("support@example.com")
    |> subject("Log in to Passwordless")
    |> assign(:token, Tokens.sign_login_request(login_request))
    |> render("login_request.html")
  end
end
```

Create a view at `lib/passwordless_web/views/email_view.ex`:

```
defmodule PasswordlessWeb.EmailView do
  use PasswordlessWeb, :view
end
```

Create a template at `lib/passwordless_web/templates/email/login_request.html.eex`:

```
<p>Hello!</p>

<p>Use the link below to log in to your Passwordless account. <strong>This link expires in 15 minutes.</strong></p>

<p><%= link "Log in to Passwordless", to: "#todo" %></p>

Update the login request controller to deliver the email:

defmodule PasswordlessWeb.LoginRequestController do
  ...

  alias Passwordless.Email
  alias Passwordless.Mailer

  ...

  def create(conn, %{"login_request" => %{"email" => email}}) do
    case LoginRequests.create(email) do
      {:ok, %{login_request: login_request}, user} ->
        user
        |> Email.login_request(login_request)
        |> Mailer.deliver_now()

        ...
    end
  end
end
```

Here we use the synchronous [`Bamboo.Mailer.deliver_now/1`](https://hexdocs.pm/bamboo/Bamboo.Mailer.html#deliver_now/1) to ensure the email was successfully handed off for delivery.

Bamboo comes with a handy plug for viewing emails sent in development. Let's add it to the router:

```
defmodule PasswordlessWeb.Router do
  ...

  if Mix.env == :dev do
    forward "/sent-emails", Bamboo.SentEmailViewerPlug
  end

  ...
end
```

Visit [http://localhost:4000](http://localhost:4000) and try to log in. Take a look at [http://localhost:4000/sent-emails](http://localhost:4000/sent-emails) to verify that the login email was sent.

## Session schema

Now we need a way to track user sessions. For this, we'll create a `Session` schema. Generate a migration:

```
mix ecto.gen.migration create_sessions
```

Edit the migration:

```
defmodule Passwordless.Repo.Migrations.CreateSessions do
  use Ecto.Migration

  def change do
    create table(:sessions) do
      add :user_id, references(:users, on_delete: :delete_all), null: false

      timestamps()
    end

    create index(:sessions, \[:user_id\])
  end
end
```

Run the migration:

```
mix ecto.migrate
```

Create the `Session` schema at `lib/passwordless/accounts/session.ex`:

```
defmodule Passwordless.Accounts.Session do
  use Ecto.Schema

  alias Passwordless.Accounts.User

  schema "sessions" do
    timestamps()

    belongs_to :user, User
  end
end
```

Update the `User` schema with the association:

```
defmodule Passwordless.Accounts.User do
  ...

  alias Passwordless.Accounts.Session

  schema "users" do
    ...
    has_many :sessions, Session
  end

  ...
end
```

## Redeem login request

Next, let's add the ability to redeem a login request for a new session. Update the `LoginRequests` module:

```
defmodule Passwordless.Accounts.LoginRequests do
  ...

  alias Passwordless.Accounts.LoginRequest
  alias Passwordless.Accounts.Tokens

  ...

  def redeem(token) do
    with {:ok, id} <- Tokens.verify_login_request(token),
         login_request when not is_nil(login_request) <- Repo.get(LoginRequest, id),
         %{user: user} <- Repo.preload(login_request, :user)
    do
      Multi.new()
      |> Multi.delete_all(:delete_login_requests, Ecto.assoc(user, :login_request))
      |> Multi.insert(:session, Ecto.build_assoc(user, :sessions))
      |> Repo.transaction()
    else
      nil ->
        {:error, :not_found}

      {:error, :expired} ->
        {:error, :expired}
    end
  end
end
```

Here, we delete any existing login requests belonging to the user for security purposes. We're also surfacing errors when a login request no longer exists or has expired.

Update the login request controller:

```
defmodule PasswordlessWeb.LoginRequestController do
  ...

  def show(conn, %{"token" => token}) do
    case LoginRequests.redeem(token) do
      {:ok, _changes} ->
        conn
        |> put_flash(:info, "Logged in successfully.")
        |> redirect(to: Routes.home_path(conn, :index))

      {:error, :expired} ->
        conn
        |> put_flash(:error, "Oops, that login link has expired.")
        |> render("new.html")

      {:error, :not_found} ->
        conn
        |> put_flash(:error, "Oops, that login link is not valid anymore.")
        |> render("new.html")
    end
  end
end
```

Update the login request routes:

```
scope "/", PasswordlessWeb do
  ...
  resources "/login-requests", LoginRequestController, only: \[:create, :new, :show\], param: "token"
end
```

We also need to update the "Log in" link in the email template:

```
...

<p><%= link "Log in to Passwordless", to: Routes.login_request_url(PasswordlessWeb.Endpoint, :show, @token) %></p>
```

Visit [http://localhost:4000](http://localhost:4000) and verify that a session record is created after clicking the link in a login email.

## Cookie session storage

We'll use cookie storage to remember the ID of a user's session. Note that behind the scenes, the [`Plug.Conn`](https://hexdocs.pm/plug/Plug.Conn.html) signs the cookie to ensure it can’t be tampered with.

Update the login requests controller:

```
defmodule PasswordlessWeb.LoginRequestController do
  ...

  def show(conn, %{"token" => token}) do
    case LoginRequests.redeem(token) do
      {:ok, %{session: session}} ->
        conn
        |> put_flash(:info, "Logged in successfully.")
        |> put_session(:session_id, session.id)
        |> configure_session(renew: true)
        |> redirect(to: Routes.home_path(conn, :index))

        ...
    end
  end
end
```

We need the ability to find a session by ID. Create a `Sessions` module at `lib/passwordless/accounts/sessions.ex`:

```
defmodule Passwordless.Accounts.Sessions do
  alias Passwordless.Accounts.Session
  alias Passwordless.Repo

  def get(id) do
    Repo.get(Session, id)
  end
end
```

Next, we need a method to check if a user is logged in. Create an `AuthHelper` module at `lib/passwordless_web/helpers/auth_helper.ex`:

```
defmodule PasswordlessWeb.AuthHelper do
  alias Plug.Conn
  alias Passwordless.Accounts.Sessions

  def logged_in?(conn) do
    with session_id when not is_nil(session_id) <- Conn.get_session(conn, :session_id),
         session when not is_nil(session) <- Sessions.get(session_id)
    do
      true
    else
      nil -> false
    end
  end
end
```

For convenience, we'll import the `AuthHelper` in all of our views. Update `lib/passwordless_web`:

```
defmodule PasswordlessWeb do
  ...

  def view do
    quote do
      ...
      import PasswordlessWeb.AuthHelper
      ...
    end
  end

  ...
end
```

When a user is logged in, we'll show a "Log out" link. Update the home page template:

```
<h1>Home</h1>

<%= if logged_in?(@conn) do %>
  <%= link "Log out", to: "#todo" %>
  <% else %>
  <%= link "Sign up", to: Routes.user_path(@conn, :new) %> or
  <%= link "Log in", to: Routes.login_request_path(@conn, :new) %>
<% end %>
```

Visit [http://localhost:4000](http://localhost:4000) and make sure that a user's session is remembered after logging in.

## Logging out

Finally, we'll give users the ability to log out. This is done by deleting the session record and dropping the session cookie.

Update the `Sessions` module:

```
defmodule Passwordless.Accounts.Sessions do
  ...

  def delete(session) do
    Repo.delete(session)
  end
end
```

Create a controller at `lib/passwordless_web/controllers/session_controller.ex`:

```
defmodule PasswordlessWeb.SessionController do
  use PasswordlessWeb, :controller

  alias Passwordless.Accounts.Sessions

  def delete(conn, _params) do
    with id when not is_nil(id) <- get_session(conn, :session_id),
         session when not is_nil(session) <- Sessions.get(id),
         {:ok, _session} <- Sessions.delete(session)
    do
      log_out(conn)
    else
      nil -> log_out(conn)
    end
  end

  defp log_out(conn) do
    conn
    |> configure_session(drop: true)
    |> redirect(to: Routes.home_path(conn, :index))
  end
end
```

Update the router:

```
scope "/", PasswordlessWeb do
  ...
  resources "/sessions", SessionController, only: \[:delete\], singleton: true
end
```

Update the "Log out" link in the home page template:

```
<h1>Home</h1>

<%= if logged_in?(@conn) do %>
  <%= link "Log out", to: Routes.session_path(@conn, :delete), method: :delete %>
<% else %>

...
```

Visit [http://localhost:4000](http://localhost:4000) and make sure that logging out works as expected.

## Next steps

Success! We've built a complete passwordless authentication flow from scratch. That said, there are a few other important pieces you'll need or want to consider:

* Configure Bamboo with an adapter for an email delivery service.
* Log a user in automatically after sign up.
* Write an authentication plug to protect private endpoints.
* Send cookies only over HTTPS in production.
* Notify a user when their email address has changed.
* Rate limit login requests to help prevent abuse.

Check out the final code in the [example repo](https://github.com/sutherland/passwordless-phoenix-example) on Github. If you found this tutorial useful or have questions or comments, feel free to [hit me up on Twitter](https://twitter.com/jonsutherland/status/1141330779353624576).

And… don't forget to check out [Proseful](https://proseful.com). ✌️

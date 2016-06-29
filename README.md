# OmniAuth Heroku

[![Build Status](https://travis-ci.org/heroku/omniauth-heroku.svg?branch=master)](https://travis-ci.org/heroku/omniauth-heroku)


[OmniAuth](https://github.com/intridea/omniauth) strategy for authenticating
Heroku users.

Mount this with your Rack application (be it Rails or Sinatra) to simplify the
[OAuth flow with Heroku](https://devcenter.heroku.com/articles/oauth).

This is intended for apps already using OmniAuth, for apps that authenticate
against more than one service (eg: Heroku and GitHub), or apps that have
specific needs on session management. If your app doesn't fall in any of these
you should consider using [Heroku
Bouncer](https://github.com/heroku/heroku-bouncer) instead.


## Configuration

OmniAuth works as a Rack middleware. Mount this Heroku adapter with:

```ruby
use OmniAuth::Builder do
  provider :heroku, ENV['HEROKU_OAUTH_ID'], ENV['HEROKU_OAUTH_SECRET']
end
```

Obtain a `HEROKU_OAUTH_ID` and `HEROKU_OAUTH_SECRET` by creating a client with
the [Heroku OAuth CLI plugin](https://github.com/heroku/heroku-oauth).

Your Heroku OAuth client should be set to receive callbacks on
`/auth/heroku/callback`.


## Usage

Initiate the OAuth flow sending users to `/auth/heroku`.

Once the authorization flow is complete and the user is bounced back to your
application, check `env["omniauth.auth"]["credentials"]`. It contains both a
refresh token and an access token (identified just as `"token"`) to the
account.

We recommend using this access token together with
[Heroku.rb](https://github.com/heroku/heroku.rb) to make API calls on behalf of
the user.

Refer to the examples below to see how these work.


### Basic account information

If you want this middleware to fetch additional Heroku account information like
the user email address and name, use the `fetch_info` option, like:

```ruby
use OmniAuth::Builder do
  provider :heroku, ENV['HEROKU_OAUTH_ID'], ENV['HEROKU_OAUTH_SECRET'],
    fetch_info: true
end
```

This sets name and email in the [omniauth auth
hash](https://github.com/intridea/omniauth/wiki/Auth-Hash-Schema). You can
access it from your app via `env["omniauth.auth"]["info"]`.

It will also add [additional Heroku account
info](https://devcenter.heroku.com/articles/platform-api-reference#account) to
`env["omniauth.auth"]["extra"]`.

### OAuth scopes

[Heroku supports different OAuth
scopes](https://devcenter.heroku.com/articles/oauth#scopes). By default this
strategy will request global access to the account, but you're encouraged to
request for less permissions when possible.

To do so, configure it like:

```ruby
use OmniAuth::Builder do
  provider :heroku, ENV['HEROKU_OAUTH_ID'], ENV['HEROKU_OAUTH_SECRET'],
    scope: "identity"
end
```

This will trim down the permissions associated to the access token given back
to you.

The Oauth scope can also be decided dynamically at runtime. For example, you could use a `scope` GET parameter if it exists, and revert to a default `scope` if it does not:

```ruby
use OmniAuth::Builder do
  provider :heroku, ENV['HEROKU_OAUTH_ID'], ENV['HEROKU_OAUTH_SECRET'],
    scope: lambda { |request| request.params['scope'] || 'identity' }
end
```


## Example - Sinatra

```ruby
class Myapp < Sinatra::Application
  use Rack::Session::Cookie, secret: ENV["SESSION_SECRET"]

  use OmniAuth::Builder do
    provider :heroku, ENV["HEROKU_OAUTH_ID"], ENV["HEROKU_OAUTH_SECRET"]
  end

  get "/" do
    redirect "/auth/heroku"
  end

  get "/auth/heroku/callback" do
    access_token = env['omniauth.auth']['credentials']['token']
    # DO NOT store this token in an unencrypted cookie session
    # Please read "A note on security" below!
    heroku_api = Heroku::API.new(api_key: access_token)
    "You have #{heroku_api.get_apps.body.size} apps"
  end
end
```

Note that we're explicitly calling `Rack::Session::Cookie` with a secret. Using
`enable :sessions` is not recommended because the secret is generated randomly,
and not reused across processes – so your users can lose their session whenever
your app restarts.


## Example - Rails

Under `config/initializers/omniauth.rb`:

```ruby
Rails.application.config.middleware.use OmniAuth::Builder do
  provider :heroku, ENV['HEROKU_OAUTH_ID'], ENV['HEROKU_OAUTH_SECRET']
end
```

Then add to `config/routes.rb`:

```ruby
Example::Application.routes.draw do
  get "login" => "sessions#new"
  get "/auth/:provider/callback" => "sessions#create"
end
```

Controller support:

```ruby
class SessionsController < ApplicationController
  def new
    redirect_to "/auth/heroku"
  end

  def create
    access_token = request.env['omniauth.auth']['credentials']['token']
    # DO NOT store this token in an unencrypted cookie session
    # Please read "A note on security" below!
    heroku_api = Heroku::API.new(api_key: access_token)
    @apps = heroku_api.get_apps.body
  end
end
```

And view:

```erb
<h1>Your Heroku apps:</h1>

<ul>
  <% @apps.each do |app| %>
    <li><%= app["name"] %></li>
  <% end %>
</ul>
```

## A note on security

**Make sure your cookie session is encrypted before storing sensitive
information on it, like access tokens**.
[encrypted_cookie](https://github.com/cvonkleist/encrypted_cookie) is a popular
gem to do that in Ruby.

Both Rails and Sinatra take a cookie secret, but that is only used to protect
against tampering; any information stored on standard cookie sessions can
easily be read from the client side, which can be further exploited to leak
credentials off your app.


## Meta

Released under the MIT license.

Created by Pedro Belo.

---
title: "Adding Authentication in Rails 6 with Rodauth"
tags: rodauth
---

In this tutorial, we'll show how to add fully functional authentication and
account management functionality into a Rails 6 app, using the **[Rodauth]**
authentication framework. Rodauth has many advantages over the mainstream
alternatives such as Devise, Sorcery, Clearance, and Authlogic, see my
[previous article][rodauth intro] for an introduction.

We'll be working with a fresh Rails app that has basic posts CRUD and
[Bootstrap] installed:

```sh
$ git clone https://gitlab.com/janko-m/rails_bootstrap_starter.git rodauth_blog
$ cd rodauth_blog
$ bin/setup
```

## Installing Rodauth

Let's start by adding the [rodauth-rails] gem to our Gemfile:

```sh
$ bundle add rodauth-rails
```

Next, we'll run the `rodauth:install` generator provided by rodauth-rails:

```sh
$ rails generate rodauth:install

# create  db/migrate/20200820215819_create_rodauth.rb
# create  config/initializers/rodauth.rb
# create  config/initializers/sequel.rb
# create  app/lib/rodauth_app.rb
# create  app/controllers/rodauth_controller.rb
# create  app/models/account.rb
```

This will create the Rodauth app with default authentication features,
configure [Sequel] which Rodauth uses for database interaction to [reuse Active
Record's database connection][sequel-activerecord_connection], and generate a
migration that will create tables for the loaded Rodauth features. Let's run
the migration:

```sh
$ rails db:migrate

# == CreateRodauth: migrating ====================================
# -- create_table(:accounts)
# -- create_table(:account_password_hashes)
# -- create_table(:account_password_reset_keys)
# -- create_table(:account_verification_keys)
# -- create_table(:account_login_change_keys)
# -- create_table(:account_remember_keys)
# == CreateRodauth: migrated ===========================
```

If everything was installed successfully, we should be able to open the
`/create-account` page and see Rodauth's default registration form.

![Rodauth create account page](/images/rodauth-create-account.png)

## Adding authentication links

Rodauth configuration generated by rodauth-rails provides several routes for
authentication and account management:

```sh
$ rails rodauth:routes

# /login                   rodauth.login_path
# /create-account          rodauth.create_account_path
# /verify-account-resend   rodauth.verify_account_resend_path
# /verify-account          rodauth.verify_account_path
# /logout                  rodauth.logout_path
# /remember                rodauth.remember_path
# /reset-password-request  rodauth.reset_password_request_path
# /reset-password          rodauth.reset_password_path
# /change-password         rodauth.change_password_path
# /change-login            rodauth.change_login_path
# /verify-login-change     rodauth.verify_login_change_path
# /close-account           rodauth.close_account_path
```

Let's use this information to add some main authentication links to our
navigation header:

```erb
<!-- app/views/application/_navbar.html.erb -->
<!-- ... --->
<% if rodauth.logged_in? %>
  <div class="dropdown">
    <%= link_to current_account.email, "#", class: "btn btn-info dropdown-toggle", data: { toggle: "dropdown" } %>
    <div class="dropdown-menu dropdown-menu-right">
      <%= link_to "Change password", rodauth.change_password_path, class: "dropdown-item" %>
      <%= link_to "Change email", rodauth.change_login_path, class: "dropdown-item" %>
      <div class="dropdown-divider"></div>
      <%= link_to "Close account", rodauth.close_account_path, class: "dropdown-item text-danger" %>
      <%= link_to "Sign out", rodauth.logout_path, method: :post, class: "dropdown-item" %>
    </div>
  </div>
<% else %>
  <div>
    <%= link_to "Sign in", rodauth.login_path, class: "btn btn-outline-primary" %>
    <%= link_to "Sign up", rodauth.create_account_path, class: "btn btn-success" %>
  </div>
<% end %>
<!-- ... --->
```

Rodauth doesn't define the `#current_account` method, so let's copy-paste the
example provided in the rodauth-rails README:

```rb
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  before_action :current_account, if: -> { rodauth.logged_in? }

  private

  def current_account
    @current_account ||= Account.find(rodauth.session_value)
  rescue ActiveRecord::RecordNotFound
    rodauth.logout
    rodauth.login_required
  end
  helper_method :current_account
end
```

Now our application will show login and registration links when the user is not
logged in:

![Rodauth login and registration links](/images/rodauth-login-registration-links.png)

While logged in users will see some basic account management links:

![Rodauth account management links](/images/rodauth-account-management-links.png)

## Requiring authentication

Now that we have working authentication, we'll likely want to require the user
to be authenticated for certain parts of our application. In our case, we want
to authenticate the posts controller.

We could add a `before_action` callback to the controller, but Rodauth allows
us to do this inside the Rodauth app's route block, which is called before each
Rails route. This way we can keep our authentication logic contained in a
single place.

```rb
# app/lib/rodauth_app.rb
class RodauthApp < Rodauth::Rails::App
  # ...
  route do |r|
    # ...
    if r.path.start_with?("/posts")
      rodauth.require_authentication
    end
  end
end
```

Now visiting the `/posts` page will redirect the user to the `/login` page if
they're not logged in.

![Rodauth login required](/images/rodauth-login-required.png)

We'll also want to associate the posts to the `accounts` table:

```sh
$ rails generate migration add_account_id_to_posts account:references
$ rails db:migrate
```
```rb
# app/models/account.rb
class Account < ApplicationRecord
  has_many :posts
  # ...
end
```

And scope them to the current account in the posts controller:

```rb
# app/controllers/posts_controller.rb
class PostsController < ApplicationController
  # ...
  def index
    @posts = current_account.posts.all
  end
  # ...
  def create
    @post = current_account.posts.build(post_params)
    # ...
  end
  # ...
  private
    def set_post
      @post = current_account.posts.find(params[:id])
    end
    # ...
end
```

## Adding new fields

To have something other than an email address to display our users, let's
require users to enter their name during registration. This will also give us
an opportunity to see how Rodauth can be configured.

Since we'll need to edit the registration form, let's first copy Rodauth's HTML
templates into our Rails application:

```sh
$ rails generate rodauth:views

# create  app/views/rodauth/_field.html.erb
# create  app/views/rodauth/_field_error.html.erb
# create  app/views/rodauth/_login_field.html.erb
# create  app/views/rodauth/_login_display.html.erb
# create  app/views/rodauth/_password_field.html.erb
# create  app/views/rodauth/_submit.html.erb
# create  app/views/rodauth/_login_form.html.erb
# create  app/views/rodauth/_login_form_footer.html.erb
# create  app/views/rodauth/_login_form_header.html.erb
# create  app/views/rodauth/login.html.erb
# create  app/views/rodauth/multi_phase_login.html.erb
# create  app/views/rodauth/logout.html.erb
# create  app/views/rodauth/_login_confirm_field.html.erb
# create  app/views/rodauth/_password_confirm_field.html.erb
# create  app/views/rodauth/create_account.html.erb
# create  app/views/rodauth/_login_hidden_field.html.erb
# create  app/views/rodauth/verify_account_resend.html.erb
# create  app/views/rodauth/verify_account.html.erb
# create  app/views/rodauth/reset_password_request.html.erb
# create  app/views/rodauth/reset_password.html.erb
# create  app/views/rodauth/_new_password_field.html.erb
# create  app/views/rodauth/change_password.html.erb
# create  app/views/rodauth/change_login.html.erb
# create  app/views/rodauth/close_account.html.erb
```

We can now open the `create_account.erb` template and add a new `name` field:

```erb
<!-- app/views/rodauth/create_account.erb -->
<%= form_tag rodauth.create_account_path, method: :post do %>
  <!-- new "name" field -->
  <div class="form-group">
    <%= label_tag "name", "Name" %>
    <%= render "field", name: "name", id: "name" %>
  </div>

  <%= render "login_field" %>
  <%= render "login_confirm_field" if rodauth.require_login_confirmation? %>
  <%= render "password_field" if rodauth.create_account_set_password? %>
  <%= render "password_confirm_field" if rodauth.create_account_set_password? && rodauth.require_password_confirmation? %>
  <%= render "submit", value: "Create Account" %>
<% end %>
```

Since the user's name won't be used for authentication, let's store it in a new
`profiles` table, and associate the `profiles` table to the `accounts` table.

```sh
$ rails generate model Profile account:references name:string
$ rails db:migrate
```
```rb
# app/models/account.rb
class Account < ApplicationRecord
  has_one :profile
  # ...
end
```

We now need our Rodauth app to actually handle the new `name` parameter. We'll
validate that it's filled in and create the associated profile record after the
account is created.

```rb
# app/lib/rodauth_app.rb
class RodauthApp < Rodauth::Rails::App
  configure do
    # ...
    before_create_account do
      # Validate presence of the name field
      throw_error_status(422, "name", "must be present") unless param_or_nil("name")
    end
    after_create_account do
      # Create the associated profile record with name
      Profile.create!(account_id: account_id, name: param("name"))
    end
    after_close_account do
      # Delete the associated profile record
      Profile.find_by!(account_id: account_id).destroy
    end
    # ...
  end
end
```

Now we can update our navigation header to use the user's name instead of their
email address:

```diff
-     <%= link_to current_account.email, "#", class: "btn btn-info dropdown-toggle", data: { toggle: "dropdown" } %>
+     <%= link_to current_account.profile.name, "#", class: "btn btn-info dropdown-toggle", data: { toggle: "dropdown" } %>
```

![Displayed new account name](/images/rodauth-account-name.png)

## Sending emails asynchronously

Rodauth will send emails as part of account verification, email change,
password change, and password reset. By default, these emails will be sent
synchronously via an internal mailer, but for performance reasons we should
send these emails asynchronously inside a background job.

Since we'll want to modify Rodauth's default email templates eventually, let's
create our own mailer with the default templates:

```sh
$ rails generate rodauth:mailer

# create  app/mailers/rodauth_mailer.rb
# create  app/views/rodauth_mailer/email_auth.text.erb
# create  app/views/rodauth_mailer/password_changed.text.erb
# create  app/views/rodauth_mailer/reset_password.text.erb
# create  app/views/rodauth_mailer/unlock_account.text.erb
# create  app/views/rodauth_mailer/verify_account.text.erb
# create  app/views/rodauth_mailer/verify_login_change.text.erb
```
```rb
class RodauthMailer < ApplicationMailer
  def verify_account(recipient, email_link)
    # ...
  end
  def reset_password(recipient, email_link)
    # ...
  end
  def verify_login_change(recipient, old_login, new_login, email_link)
    # ...
  end
  def password_changed(recipient)
    # ...
  end
end
```

Now, to have our mailer automatically called by Rodauth and to deliver emails
in the background, let's uncomment the following lines in our Rodauth app:

```rb
# app/lib/rodauth_app.rb
class RodauthApp < Rodauth::Rails::App
  configure do
    # ...
    create_reset_password_email do
      RodauthMailer.reset_password(email_to, reset_password_email_link)
    end
    create_verify_account_email do
      RodauthMailer.verify_account(email_to, verify_account_email_link)
    end
    create_verify_login_change_email do |login|
      RodauthMailer.verify_login_change(login, verify_login_change_old_login, verify_login_change_new_login, verify_login_change_email_link)
    end
    create_password_changed_email do
      RodauthMailer.password_changed(email_to)
    end
    send_email do |email|
      db.after_commit { email.deliver_later }
    end
    # ...
  end
end
```

We enqueue the email deliveries after the database transaction commits, to
ensure that any database changes made before Rodauth called into our mailer
have been applied when the background job is picked up.

## Closing words

In this tutorial we've gradually built out a complete authentication and
account management flow using the Rodauth authentication framework. It
supports login & logout, account creation with email verification and a grace
period, password change & password reset, email change with email verification,
and close account functionality. We've seen how to require authentication for
certain routes, add new fields to the registration form, and send
authentication emails asynchrously.

I'm personally very excited about Rodauth, as it has an impressive featureset
and a refreshingly clean design, and also it's not tied to Rails. I've been
working hard on [rodauth-rails] to make it as easy as possible to get started
with in Rails, so hopefully it will help Rodauth gain more traction in the
Rails community.

[Rodauth]: https://github.com/jeremyevans/rodauth
[rodauth intro]: https://janko.io/rodauth-a-refreshing-authentication-solution-for-ruby/
[rodauth-rails]: https://github.com/janko/rodauth-rails
[Sequel]: https://github.com/jeremyevans/sequel
[sequel-activerecord_connection]: https://github.com/janko/sequel-activerecord_connection
[Bootstrap]: https://getbootstrap.com/
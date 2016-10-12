# Two factor authentication for Devise

[![Gitter](https://badges.gitter.im/Join%20Chat.svg)](https://gitter.im/Houdini/two_factor_authentication?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

[![Build Status](https://travis-ci.org/Houdini/two_factor_authentication.svg?branch=master)](https://travis-ci.org/Houdini/two_factor_authentication)
[![Code Climate](https://codeclimate.com/github/Houdini/two_factor_authentication.png)](https://codeclimate.com/github/Houdini/two_factor_authentication)

## Features

* Support for 2 types of OTP codes
 1. Codes delivered directly to the user
 2. TOTP (Google Authenticator) codes based on a shared secret (HMAC)
* Configurable OTP code digit length
* Configurable max login attempts
* Customizable logic to determine if a user needs two factor authentication
* Configurable period where users won't be asked for 2FA again
* Option to encrypt the TOTP secret in the database, with iv and salt

## Configuration

### Initial Setup

In a Rails environment, require the gem in your Gemfile:

    gem 'two_factor_authentication'

Once that's done, run:

    bundle install

Note that Ruby 2.1 or greater is required.

### Installation

#### Automatic initial setup

To set up the model and database migration file automatically, run the
following command:

    bundle exec rails g two_factor_authentication MODEL

Where MODEL is your model name (e.g. User or Admin). This generator will add
`:two_factor_authenticatable` to your model's Devise options and create a
migration in `db/migrate/`, which will add the following columns to your table:

- `:second_factor_attempts_count`
- `:encrypted_otp_secret_key`
- `:encrypted_otp_secret_key_iv`
- `:encrypted_otp_secret_key_salt`
- `:direct_otp`
- `:direct_otp_sent_at`
- `:totp_timestamp`

#### Manual initial setup

If you prefer to set up the model and migration manually, add the
`:two_factor_authentication` option to your existing devise options, such as:

```ruby
devise :database_authenticatable, :registerable, :recoverable, :rememberable,
       :trackable, :validatable, :two_factor_authenticatable
```

Then create your migration file using the Rails generator, such as:

```
rails g migration AddTwoFactorFieldsToUsers second_factor_attempts_count:integer encrypted_otp_secret_key:string:index encrypted_otp_secret_key_iv:string encrypted_otp_secret_key_salt:string direct_otp:string direct_otp_sent_at:datetime totp_timestamp:timestamp
```

Open your migration file (it will be in the `db/migrate` directory and will be
named something like `20151230163930_add_two_factor_fields_to_users.rb`), and
add `unique: true` to the `add_index` line so that it looks like this:

```ruby
add_index :users, :encrypted_otp_secret_key, unique: true
```
Save the file.

#### Complete the setup

Run the migration with:

    bundle exec rake db:migrate

Add the following line to your model to fully enable two-factor auth:

    has_one_time_password(encrypted: true)

Set config values in `config/initializers/devise.rb`:

```ruby
config.max_login_attempts = 3  # Maximum second factor attempts count.
config.allowed_otp_drift_seconds = 30  # Allowed TOTP time drift between client and server.
config.otp_length = 6  # TOTP code length
config.direct_otp_valid_for = 5.minutes  # Time before direct OTP becomes invalid
config.direct_otp_length = 6  # Direct OTP code length
config.remember_otp_session_for_seconds = 30.days  # Time before browser has to perform 2fA again. Default is 0.
config.otp_secret_encryption_key = ENV['OTP_SECRET_ENCRYPTION_KEY']
config.second_factor_resource_id = 'id' # Field or method name used to set value for 2fA remember cookie
```
The `otp_secret_encryption_key` must be a random key that is not stored in the
DB, and is not checked in to your repo. It is recommended to store it in an
environment variable, and you can generate it with `bundle exec rake secret`.

Override the method in your model in order to send direct OTP codes. This is
automatically called when a user logs in unless they have TOTP enabled (see
below):

```ruby
def send_two_factor_authentication_code(code)
  # Send code via SMS, etc.
end
```

### Customisation and Usage

By default, second factor authentication is required for each user. You can
change that by overriding the following method in your model:

```ruby
def need_two_factor_authentication?(request)
  request.ip != '127.0.0.1'
end
```

In the example above, two factor authentication will not be required for local
users.

This gem is compatible with [Google Authenticator](https://support.google.com/accounts/answer/1066447?hl=en).
To enable this a shared secret must be generated by invoking the following
method on your model:

```ruby
user.generate_totp_secret
```

This must then be shared via a provisioning uri:

```ruby
user.provisioning_uri # This assumes a user model with an email attribute
```

This provisioning uri can then be turned in to a QR code if desired so that
users may add the app to Google Authenticator easily.  Once this is done, they
may retrieve a one-time password directly from the Google Authenticator app.

#### Overriding the view

The default view that shows the form can be overridden by adding a
file named `show.html.erb` (or `show.html.haml` if you prefer HAML)
inside `app/views/devise/two_factor_authentication/` and customizing it.
Below is an example using ERB:


```html
<h2>Hi, you received a code by email, please enter it below, thanks!</h2>

<%= form_tag([resource_name, :two_factor_authentication], :method => :put) do %>
  <%= text_field_tag :code %>
  <%= submit_tag "Log in!" %>
<% end %>

<%= link_to "Sign out", destroy_user_session_path, :method => :delete %>

```

#### Enable TOTP support for existing users

If you have existing users that need to be provided with a OTP secret key, so
they can use TOTP, create a rake task. It could look like this one below:

```ruby
desc 'rake task to update users with otp secret key'
task :update_users_with_otp_secret_key  => :environment do
  User.find_each do |user|
    user.generate_totp_secret
    user.save!
    puts "Rake[:update_users_with_otp_secret_key] => OTP secret key set to '#{key}' for User '#{user.email}'"
  end
end
```
Then run the task with `bundle exec rake update_users_with_otp_secret_key`

#### Adding the TOTP encryption option to an existing app

If you've already been using this gem, and want to start encrypting the OTP
secret key in the database (recommended), you'll need to perform the following
steps:

1. Generate a migration to add the necessary columns to your model's table:

   ```
   rails g migration AddEncryptionFieldsToUsers encrypted_otp_secret_key:string:index encrypted_otp_secret_key_iv:string encrypted_otp_secret_key_salt:string
   ```

   Open your migration file (it will be in the `db/migrate` directory and will be
   named something like `20151230163930_add_encryption_fields_to_users.rb`), and
   add `unique: true` to the `add_index` line so that it looks like this:

   ```ruby
   add_index :users, :encrypted_otp_secret_key, unique: true
   ```
   Save the file.

2. Run the migration: `bundle exec rake db:migrate`

2. Update the gem: `bundle update two_factor_authentication`

3. Add `encrypted: true` to `has_one_time_password` in your model.
   For example: `has_one_time_password(encrypted: true)`

4. Generate a migration to populate the new encryption fields:
   ```
   rails g migration PopulateEncryptedOtpFields
   ```

   Open the generated file, and replace its contents with the following:
   ```ruby
   class PopulateEncryptedOtpFields < ActiveRecord::Migration
      def up
        User.reset_column_information

        User.find_each do |user|
          user.otp_secret_key = user.read_attribute('otp_secret_key')
          user.save!
        end
      end

      def down
        User.reset_column_information

        User.find_each do |user|
          user.otp_secret_key = ROTP::Base32.random_base32
          user.save!
        end
      end
    end
  ```

5. Generate a migration to remove the `:otp_secret_key` column:
   ```
   rails g migration RemoveOtpSecretKeyFromUsers otp_secret_key:string
   ```

6. Run the migrations: `bundle exec rake db:migrate`

If, for some reason, you want to switch back to the old non-encrypted version,
use these steps:

1. Remove `(encrypted: true)` from `has_one_time_password`

2. Roll back the last 3 migrations (assuming you haven't added any new ones
after them):
   ```
   bundle exec rake db:rollback STEP=3
   ```

### Example App

[TwoFactorAuthenticationExample](https://github.com/Houdini/TwoFactorAuthenticationExample)

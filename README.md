# Pretender

As an admin, there are times you want to see exactly what another user sees or take action on behalf of a user.  Meet Pretender.

- Easy to switch back and forth between roles
- Minimal code changes
- Plays nicely with auditing tools - **this is crucial**

[Rock on](http://www.youtube.com/watch?v=SBjQ9tuuTJQ) :boom:

Pretender is also flexible and lightweight - less than 40 lines of code :-)

Pretender works with Rails 2.3+ and almost any authentication system.
(devise, authlogic, sorcery, and many more - it's agnostic)

[Battle-tested at Instacart](https://www.instacart.com)

## Get started

Add this line to your application's Gemfile:

```ruby
# Gemfile
gem 'pretender'
```

And add this line to your ApplicationController:

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  impersonates :user
end
```

This adds three methods to your controllers:

```ruby
true_user
# returns authenticated user

impersonate_user(user)
# allows you to login as another user

stop_impersonating_user
# become yourself again
  ```

And changes the behavior of another:

```ruby
current_user
# now returns:
# - if impersonating, the impersonated user
# - otherwise, the true user
```

**Note:** the name of this method is configurable (details at the end)

Now we need to setup a way to login as another user.  **Pretender makes no assumptions about how you want to do this**.  I like to add this to my admin dashboard.

### Sample Implementation

```ruby
class Admin::UsersController < ApplicationController
  before_filter :require_admin, :except => [:stop_impersonating]

  def impersonate
    user = User.find(params[:id])
    impersonate_user(user)
    redirect_to root_path
  end

  # do not require admin for this method if access control
  # is performed on the current_user instead of true_user
  def stop_impersonating
    stop_impersonating_user
    redirect_to admin_path
  end
end
```

### Very Important!

Be sure to call `stop_impersonating_user` before the current user signs out.

#### Devise

Assuming you have a `User` model, change your routes to:

```ruby
devise_for :users, controllers: {sessions: "sessions"}
```

Then create a `SessionsController` with:

```ruby
class SessionsController < Devise::SessionsController
  def destroy
    stop_impersonating_user
    super
  end
end
```

#### Other

```ruby
class SessionsController < ApplicationController
  def destroy
    # it's safe to call this regardless of whether the user is being impersonated
    stop_impersonating_user
    # now, log out the user
    # ...
  end
end
```

### Show Admins

You may want to make it obvious to an admin when he / she is signed in as another user.  I like to add this to the application layout.

#### Haml / Slim

```haml
- # app/views/layouts/application.haml
- if current_user != true_user
  .alert
    You (#{true_user.name}) are signed in as #{current_user.name}
    = link_to "Back to admin", stop_impersonating_user_path
```

### Audits

If you keep audit logs with a library like [audited](https://github.com/collectiveidea/audited), make sure it uses the **true user**.

```ruby
Audited.current_user_method = :true_user
```

### Configuration

Pretender is super flexible.  You can change the names of methods and even impersonate multiple roles at the same time.  Here's the default configuration.

```ruby
# app/controllers/application_controller.rb
impersonates :user,
             :method => :current_user,
             :with => proc{|id| User.where(:id => id).first }
```

Mold it to fit your application.

```ruby
# app/controllers/application_controller.rb
impersonates :account,
             :method => :authenticated_account,
             :with => proc{|id| EnterpriseAccount.where(:id => id).first }
```

This creates three methods:

```ruby
true_account
impersonate_account
stop_impersonating_account
```

Also, authenticated_account is overridden with `EnterpriseAccount.where(:id => id).first`

### That's all folks!

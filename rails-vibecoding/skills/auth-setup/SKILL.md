---
name: auth-setup
description: Use when implementing authentication, authorization, or permission patterns in Rails 8. Applies to login, signup, session, password reset, current_user, Current attributes, role-based access, admin panel, multi-tenant scoping, permission isolation testing, Rails credentials. Triggers when user mentions auth, login, signup, session, current_user, password, admin, role, permission, isolation test, multi-tenant, membership, workspace, has_secure_password, credentials, or Rails 8 authentication generator.
---

# Auth Setup — Rails 8 Built-in

> Verified against Rails 8 guides at https://guides.rubyonrails.org/security.html — no devise, no pundit.

## Philosophy

> Rails 8 ships a ~150-line auth generator. It's enough.

- **Authentication** = "who are you?" → login, session, password
- **Authorization** = "what can you do?" → role check, ownership scoping

Two concepts, two layers.

## Generator Command

```bash
bin/rails generate authentication
```

## What the Generator Creates

**Models:**
- `app/models/user.rb` — `has_secure_password`, `email_address`
- `app/models/session.rb` — user ref + IP + user agent (multi-device)
- `app/models/current.rb` — `ActiveSupport::CurrentAttributes` for `Current.user`

**Controllers:**
- `app/controllers/sessions_controller.rb` — login / logout
- `app/controllers/passwords_controller.rb` — password reset flow
- `app/controllers/concerns/authentication.rb` — `require_authentication`, `authenticated?`, sign-in/sign-out helpers

**Views:**
- `app/views/sessions/new.html.erb` — login form
- `app/views/passwords/new.html.erb` + `edit.html.erb` — reset flow

**Mailer:**
- `app/mailers/passwords_mailer.rb` — reset email
- `app/views/passwords_mailer/reset.html.erb` + `.text.erb`

**Migrations:**
- `CreateUsers` (email_address, password_digest)
- `CreateSessions` (user reference, IP address, user agent)

**Other:**
- Adds `bcrypt` to Gemfile
- Modifies `config/routes.rb`

## ⚠️ What the Generator Does NOT Include

**No signup/registration flow.** Student must build:
- `RegistrationsController#new` + `#create`
- Signup form view
- Routes: `resources :registrations, only: [:new, :create]`

This is deliberate — Rails ships login + password reset, you define how people become users.

## ⚠️ Password Reset Requires Email (SMTP)

Generator assumes working mailer. Password reset token = 15-minute validity via `has_secure_password`.

If app has no SMTP (common for v1 indie apps):
1. Delete `PasswordsController`, `PasswordsMailer`, reset views + routes after generating
2. Provide manual admin password reset via admin panel (see below)

## Common Patterns After Generator

### Current.user
Uses CurrentAttributes so `current_user` works anywhere (models, mailers, jobs):
```ruby
# In controller
before_action :resume_session  # or :require_authentication

# Access
Current.user       # anywhere in request
current_user       # in views/controllers (helper)
```

### Authentication in Controllers
```ruby
class ApplicationController < ActionController::Base
  include Authentication
end

class PostsController < ApplicationController
  # require_authentication already applied by Authentication concern
end

class PublicController < ApplicationController
  allow_unauthenticated_access  # opt-out per-controller
end
```

### Rate Limit Login Attempts
Built into Rails 8:
```ruby
class SessionsController < ApplicationController
  allow_unauthenticated_access only: [:new, :create]
  rate_limit to: 10, within: 3.minutes, only: :create,
    with: -> { redirect_to new_session_url, alert: "Too many attempts" }
end
```

### Session Fixation Prevention
Generator uses `reset_session` on login automatically.

### Authenticate Login
```ruby
user = User.authenticate_by(email_address: params[:email], password: params[:password])
```
Returns user if valid, nil otherwise.

## Authorization — Simple Role Checks

No pundit/cancancan. Simple methods:
```ruby
class User < ApplicationRecord
  # t.boolean :admin, default: false
  # OR
  # t.string :role (enum :role)
end

class AdminController < ApplicationController
  before_action :require_admin

  private

  def require_admin
    redirect_to root_path, alert: "Forbidden" unless Current.user.admin?
  end
end
```

## Permission Isolation (Multi-User Apps)

Every query scoped via association:
```ruby
# ❌ BAD — returns any workspace
Workspace.find(params[:id])

# ✅ GOOD — scoped via current_user's memberships
Current.user.workspaces.find(params[:id])
```

### Isolation Test (Mandatory for Multi-User)
```ruby
test "member of workspace A cannot access workspace B" do
  sign_in_as users(:alice_member_of_a)
  get list_path(lists(:workspace_b_list))
  assert_response :not_found  # or :forbidden
end
```

Manual test + automated test. Never trust scope enforcement without verification.

## Rails Credentials (Secrets)

```bash
EDITOR=nano bin/rails credentials:edit
```

- Encrypted: `config/credentials.yml.enc`
- Key: `config/master.key` (`.gitignore`d, never commit)
- Access: `Rails.application.credentials.some_key`

## Admin Panel Pattern

Config-based admin definition (safer than "first user = admin"):
```yaml
# config/admins.yml
- admin@example.com
```

```ruby
class User < ApplicationRecord
  def admin?
    admin_emails = YAML.load_file(Rails.root.join("config/admins.yml"))
    admin_emails.include?(email_address)
  end
end
```

## Security Defaults (Enabled in Rails 8)

- ✅ CSRF protection (`protect_from_forgery with: :exception`)
- ✅ Encrypted session cookies (via `secret_key_base`)
- ✅ HTTPOnly cookie flags
- ✅ SQL injection protection (ActiveRecord parameterization)
- ✅ XSS via default output escaping (ERB auto-escapes)
- ✅ `has_secure_password` with bcrypt

Don't disable these. Use parameterized queries (`where("col = ?", input)`), not string interpolation.

## References

- `references/authentication.md` — passwordless magic links pattern (37signals alternative)
- `references/multi-tenancy.md` — path-based tenancy, middleware, ActiveJob extensions
- `references/security-checklist.md` — XSS, CSRF, SSRF, rate limiting, authorization
- Official Rails security guide: https://guides.rubyonrails.org/security.html

---
name: active-storage
description: Use when handling file uploads, image attachments, or digital file delivery in Rails 8. Applies to has_one_attached, has_many_attached, form file_field, url_for blob, rails_blob_path, image variants, signed URLs (for secure/expiring download), service configuration (Disk for local, S3/R2 for cloud), direct uploads, and file delivery patterns. Triggers when user mentions file upload, image upload, avatar, attachment, Active Storage, signed URL, blob, variant, image resize, download link, digital product delivery, or PDF/file serving.
---

# Active Storage

> Rails built-in file attachment framework. Supports local disk (v1 default) + cloud storage (S3/R2/GCS) via service configuration.

## Setup (One-Time)

```bash
bin/rails active_storage:install
bin/rails db:migrate
```

Creates three tables: `active_storage_blobs`, `active_storage_attachments`, `active_storage_variant_records`.

## Service Configuration

Configure services in `config/storage.yml`:

```yaml
local:
  service: Disk
  root: <%= Rails.root.join("storage") %>

# For production cloud upgrade:
amazon:
  service: S3
  access_key_id: <%= Rails.application.credentials.dig(:aws, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:aws, :secret_access_key) %>
  bucket: my-bucket-<%= Rails.env %>
  region: "us-east-1"

cloudflare_r2:
  service: S3                   # R2 is S3-compatible
  endpoint: https://<account-id>.r2.cloudflarestorage.com
  access_key_id: <%= Rails.application.credentials.dig(:r2, :access_key_id) %>
  secret_access_key: <%= Rails.application.credentials.dig(:r2, :secret_access_key) %>
  bucket: my-bucket
  region: auto
```

Select per environment in `config/environments/*.rb`:

```ruby
# config/environments/development.rb
config.active_storage.service = :local

# config/environments/production.rb
config.active_storage.service = :local   # or :amazon / :cloudflare_r2 for scale
```

## Attachment Macros

### has_one_attached
```ruby
class Post < ApplicationRecord
  has_one_attached :hero_image
end

# Attach
post.hero_image.attach(params[:hero_image])

# Check
post.hero_image.attached?

# Display
<%= image_tag post.hero_image if post.hero_image.attached? %>

# Remove
post.hero_image.purge         # sync
post.hero_image.purge_later   # async via Active Job
```

### has_many_attached
```ruby
class Message < ApplicationRecord
  has_many_attached :images
end

# Attach multiple
message.images.attach(params[:images])

# Display each
<% message.images.each do |img| %>
  <%= image_tag img %>
<% end %>
```

## Forms

### Simple file field
```erb
<%= form_with model: @post do |f| %>
  <%= f.file_field :hero_image %>
  <%= f.submit %>
<% end %>
```

### Multiple files
```erb
<%= form_with model: @message do |f| %>
  <%= f.file_field :images, multiple: true %>
<% end %>
```

### Controller — strong params
```ruby
class PostsController < ApplicationController
  def create
    @post = Post.new(post_params)
    @post.save!
    redirect_to @post
  end

  private

  def post_params
    params.expect(post: [:title, :body, :hero_image])
  end
end
```

For multiple: `params.expect(message: [:body, images: []])`.

## Displaying Files

### url_for (redirect via Active Storage)
```erb
<%= image_tag post.hero_image %>
<%= link_to "Download", post.hero_image %>
```

### rails_blob_path (named route)
```erb
<%= link_to "Download PDF",
      rails_blob_path(post.attachment, disposition: "attachment") %>
```

### Proxy mode (serve through Rails, bypass redirect)
```ruby
# config/initializers/active_storage.rb
Rails.application.config.active_storage.resolve_model_to_route = :rails_storage_proxy
```

## Image Variants (Resize, Format)

### Model-declared variants
```ruby
class User < ApplicationRecord
  has_one_attached :avatar do |attachable|
    attachable.variant :thumb, resize_to_limit: [100, 100]
    attachable.variant :medium, resize_to_limit: [500, 500]
  end
end
```

### Display variant
```erb
<%= image_tag user.avatar.variant(:thumb) %>
```

### Inline variant (one-off)
```erb
<%= image_tag post.hero_image.variant(resize_to_limit: [800, 600]) %>
```

Variants are processed lazily on first request, cached. Use `.processed.url` to force sync processing if needed.

## Signed URLs (Critical for Secure Delivery)

Signed URLs expire and can't be forged. Essential for digital product delivery:

```ruby
# In controller after payment
@download_url = rails_blob_path(@order.product_file, disposition: "attachment")

# Or generate signed ID yourself for custom delivery
signed_id = @order.signed_id(purpose: :download, expires_in: 24.hours)
```

```ruby
# Recipient endpoint
class DownloadsController < ApplicationController
  allow_unauthenticated_access

  def show
    order = Order.find_signed!(params[:token], purpose: :download)
    redirect_to rails_blob_path(order.product_file, disposition: "attachment")
  rescue ActiveSupport::MessageVerifier::InvalidSignature
    head :forbidden
  end
end
```

**Pattern for digital product delivery (no email required):**
1. After payment confirmed, generate signed URL (24-hour expiry)
2. Redirect buyer directly to signed URL
3. Buyer bookmarks or saves (QR code helps)
4. Expired? Admin regenerates via admin panel

## Direct Uploads (Skip App Server)

For large files, bypass Rails server — upload directly from browser to storage:

```erb
<%= javascript_include_tag "activestorage" %>
<%= form.file_field :attachments, multiple: true, direct_upload: true %>
```

Active Storage JS handles presigning + direct upload. App server only gets the blob reference.

## Kamal Deployment — Persistent Volumes

Local disk service needs Kamal volume:

```yaml
# config/deploy.yml
volumes:
  - "storage:/rails/storage"   # persist uploads across deploys
```

Without this, every deploy loses uploads.

## Testing

```ruby
test "attaches avatar" do
  post = Post.new(title: "Hello")
  post.hero_image.attach(
    io: File.open(Rails.root.join("test/fixtures/files/hero.png")),
    filename: "hero.png",
    content_type: "image/png"
  )
  assert post.hero_image.attached?
end

# In controller tests — use file_fixture_upload
test "creates post with image" do
  post posts_url, params: {
    post: { title: "Hello", hero_image: file_fixture_upload("hero.png") }
  }
  assert_redirected_to post_url(Post.last)
end

# Clean up test uploads
class ApplicationSystemTestCase < ActionDispatch::SystemTestCase
  def after_teardown
    super
    FileUtils.rm_rf(ActiveStorage::Blob.service.root)
  end
end
```

## When to Use Local Disk vs Cloud

| Scenario | Service |
|---|---|
| Solo builder, v1 app, < 100 users | **Local disk** — simpler, free |
| Scaling past 1 server | Cloud (S3/R2) — shared storage |
| Large files (video, PDFs > 100MB) | Cloud — offload bandwidth |
| Backup/durability critical | Cloud — S3 is 99.999999999% durable |
| Indonesian market, cost-conscious | Cloudflare R2 — no egress fees |

**Local disk + Kamal persistent volume is fine for v1.** Don't over-engineer — upgrade to cloud when scale demands it.

## Anti-Patterns

- ❌ Storing uploads in `public/uploads/` manually (use Active Storage)
- ❌ Signing download URLs with default (no) expiry (always set `expires_in:`)
- ❌ Exposing blob URLs publicly without auth check (use signed URLs)
- ❌ Forgetting Kamal volume for local disk (deploys lose files)
- ❌ Processing variants synchronously for every request (use cached `.variant(...)`)
- ❌ Using S3 with public-read ACL when signed URLs would work

## Related Skills

- **rails-conventions** — where Active Storage fits in Rails app
- **mayar-payment-integration** — uses Active Storage for signed download after purchase
- **kamal-deployment** — persistent volume config for local disk

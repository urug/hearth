# File Uploads

Handling images, documents, and other attachments in messages.

---

## Active Storage

Rails ships with Active Storage for file attachment handling. It handles uploads, storage
backends, variants (image resizing), and direct uploads to cloud storage.

Already available — no gem needed. `image_processing` is already in Hearth's Gemfile for
image variants.

---

## Attaching Files to Messages

```ruby
# app/models/message.rb
class Message < ApplicationRecord
  has_many_attached :attachments
end
```

```erb
<%# In the message form %>
<%= form.file_field :attachments, multiple: true, direct_upload: true %>
```

---

## Storage Backends

| Backend | Config | Use case |
|---|---|---|
| Local disk | `config/storage.yml` → `local` | Development, small self-hosted deployments |
| Amazon S3 | `service: S3` | Production standard |
| Cloudflare R2 | S3-compatible | S3 API, no egress fees — good value |
| Backblaze B2 | S3-compatible | Cheap storage, S3-compatible |
| MinIO | S3-compatible, self-hosted | Full control, on-prem |

Switch backends per environment in `config/environments/*.rb`:
```ruby
config.active_storage.service = :amazon  # production
config.active_storage.service = :local   # development
```

---

## Direct Uploads

Direct upload sends the file straight to S3 from the browser — the Rails server never handles
the file bytes, only the metadata. Essential for large files and production performance.

Enable in the form:
```erb
<%= javascript_include_tag "@rails/activestorage", type: "module" %>
<%= form.file_field :attachments, multiple: true, direct_upload: true %>
```

The `direct_upload: true` attribute hooks into Active Storage's JS which:
1. Requests a presigned URL from Rails
2. Uploads directly to S3
3. Attaches the blob reference to the form submission

---

## Image Variants

Resize avatars and image previews without serving originals:

```ruby
# In views
<%= image_tag message.attachments.first.variant(resize_to_limit: [800, 600]) %>

# For avatars
<%= image_tag user.avatar.variant(resize_to_fill: [40, 40]) %>
```

Requires `image_processing` gem (already in Gemfile) and either ImageMagick or libvips
(`gem "ruby-vips"` for better performance).

---

## File Type Validation

```ruby
class Message < ApplicationRecord
  has_many_attached :attachments

  validate :acceptable_attachments

  private

  ALLOWED_TYPES = %w[
    image/jpeg image/png image/gif image/webp
    application/pdf text/plain
  ].freeze
  MAX_SIZE = 10.megabytes

  def acceptable_attachments
    attachments.each do |attachment|
      unless attachment.blob.byte_size <= MAX_SIZE
        errors.add(:attachments, "is too large (max 10MB)")
      end

      unless ALLOWED_TYPES.include?(attachment.blob.content_type)
        errors.add(:attachments, "must be an image or PDF")
      end
    end
  end
end
```

---

## Displaying Attachments in Messages

```erb
<% message.attachments.each do |attachment| %>
  <% if attachment.image? %>
    <%= image_tag attachment.variant(resize_to_limit: [600, 400]), class: "message-image" %>
  <% else %>
    <%= link_to attachment.filename, rails_blob_path(attachment, disposition: :attachment) %>
  <% end %>
<% end %>
```

---

## Considerations

- **Virus scanning** — consider ClamAV or a cloud scanning API before serving user uploads
- **Content-Disposition** — serve user uploads as `attachment` or from a separate domain to
  prevent XSS via malicious HTML files
- **Storage costs** — set a per-user or per-room storage quota for long-running communities
- **Pruning orphaned blobs** — run `ActiveStorage::Blob.unattached.where("created_at < ?", 2.days.ago).destroy_all` periodically via a job

---

## References

- [Active Storage overview](https://guides.rubyonrails.org/active_storage_overview.html)
- [Direct uploads guide](https://guides.rubyonrails.org/active_storage_overview.html#direct-uploads)
- [Cloudflare R2](https://www.cloudflare.com/developer-platform/r2/)
- [ruby-vips](https://github.com/libvips/ruby-vips) — faster image processing than ImageMagick

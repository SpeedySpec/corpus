# 0013. Images are made by a Rust editor backend, on the job runner

- **Status:** Accepted, 2026-11-19
- **Decided by:** the designer

## Context

Core has `WP_Image_Editor` (GD and Imagick), resumable `wp_create_image_subsizes()`, `image_editor_output_format`
(WebP since 6.1, AVIF since 6.5), `big_image_size_threshold`, and srcset and lazy loading through
`wp_filter_content_tags()`. Smush, EWWW and Imagify add compression and format conversion.

## Decision

- A Rust image-editor backend, with WebP and AVIF encoders, beside GD and Imagick.
- Sub-size generation moves to the background job runner (0008), so uploads return at once. Core's resumable sub-size
  bookkeeping tracks progress.
- Added: re-compressing the existing library, stripping metadata, and a quality setting per format.
- Kept: `wp_image_editors`, `intermediate_image_sizes_advanced`, `image_editor_output_format`,
  `wp_generate_attachment_metadata` and `wp_calculate_image_srcset`.

## Consequences

- Waits for step 3 of 0003: attachments and the job runner.
- Sub-sizes appear shortly after an upload, not during it.

## Patterns

- **Strategy:** image encoders and editor backends.
- **Command:** each image job on the queue.

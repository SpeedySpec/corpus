# ADR-013: Images are made by a Rust editor backend, on the job runner

**Status:** Accepted
**Date:** 2026-11-19
**Deciders:** the designer
**Exploration:** [Native plugin features](../speedyspec-server/explorations/2026-11-19-native-plugin-features.md)
**Related:** [ADR-001](001-plugin-features-are-behaviour-of-ported-apis.md), [ADR-003](003-native-features-wait-for-api-ports.md), [ADR-005](005-core-hooks-stay.md), [ADR-008](008-background-job-runner-in-the-runtime.md)

---

## Context

Core already has most of what image plugins build on:

- `WP_Image_Editor`, with GD and Imagick backends;
- resumable `wp_create_image_subsizes()`;
- `image_editor_output_format` (WebP since 6.1, AVIF since 6.5) and `big_image_size_threshold`;
- srcset and lazy loading through `wp_filter_content_tags()`.

Smush, EWWW and Imagify add compression and format conversion on top, mostly in PHP or through remote services.
Making sub-sizes during the upload request is slow.

### Why this matters

Image work is the heaviest thing a WordPress site does on a request. Done in PHP, it holds a worker and makes the
uploader wait. Done outside core's editor API, it breaks the plugins and themes that pick sizes and formats through
core's filters.

---

## Decision

Add a Rust image-editor backend, with WebP and AVIF encoders, beside GD and Imagick. Move sub-size generation to the
background job runner (ADR-008).

### Rules

1. The Rust backend is one more `WP_Image_Editor` implementation, chosen through `wp_image_editors` like the others.
2. Sub-sizes are made by jobs on the runner, not in the upload request. Core's resumable sub-size bookkeeping tracks
   progress.
3. Added: re-compressing the existing library, stripping metadata, and a quality setting per format.
4. Kept: `wp_image_editors`, `intermediate_image_sizes_advanced`, `image_editor_output_format`,
   `wp_generate_attachment_metadata` and `wp_calculate_image_srcset`.
5. Recorded compatibility tests compare sub-sizes with WordPress's real output.

### What this looks like

✅ **Correct**

```rust
// The upload queues one job per attachment; the runner makes each size through the editor contract.
impl Job for GenerateSubSizes {
    fn run(&self, ctx: &JobContext) -> Result<(), JobError> {
        let sizes = ctx.hooks.apply_filters("intermediate_image_sizes_advanced", ...)?;
        for size in missing_sizes(&ctx.meta(self.attachment_id)?, &sizes) {
            let editor = ctx.editors.pick(self.attachment_id)?; // honours wp_image_editors
            ctx.meta.record_size(self.attachment_id, editor.resize_and_save(&size)?)?;
        }
        Ok(())
    }
}
```

❌ **Not permitted**

```rust
// Sizes made inline during upload, with a fixed encoder that bypasses the editor filters.
let webp = webp::Encoder::from_image(&img)?.encode(80.0);
std::fs::write(path.with_extension("webp"), &*webp)?;
```

### Patterns

- **Strategy:** image encoders and editor backends.
- **Command:** each image job on the queue.

---

## Consequences

### Positive

1. Uploads return at once.
2. WebP and AVIF output without a PHP plugin or a remote service.
3. Themes and plugins keep choosing sizes and formats through core's filters.

### Negative

1. Sub-sizes appear shortly after an upload, not during it.
   **Mitigation:** core's resumable bookkeeping records which sizes exist; srcset uses what is ready.
2. Waits for step 3 of ADR-003: attachments and the job runner.
   **Mitigation:** none; follows from ADR-003.

### Trade-offs

**We prioritise:**
- fast uploads;
- core's editor API as the single way images are made.

**We accept:**
- a short delay before sub-sizes exist;
- one more editor backend to maintain beside GD and Imagick.

---

## Fleet impact

- **speedyspec-server (repository):** gains the attachments port, the Rust editor backend and the image jobs. Not
  started.
- **speedyspec-php-runtime:** runs the jobs on its job runner (ADR-008). Not built.
- **Admin API:** uploads enqueue jobs instead of making sizes. Unverified.
- **Blog, front-end API:** serve the sizes once made; srcset follows core.
- **Plugins:** core's image filters keep working; Smush, EWWW and Imagify APIs are not supported.
- **speedyspec-server (binary), speedyspec-cli, admin site:** unaffected.

---

## Enforcement

### Automated

None. This rule is enforced by review only. Recorded compatibility tests for image sub-sizes are planned but do not
exist yet.

### Review checklist

- [ ] No image work happens inside the upload request.
- [ ] The Rust backend is reached through `wp_image_editors`, not called directly.
- [ ] The kept filters in rule 4 fire with core's names and arguments.
- [ ] Each encoder is a strategy behind the editor contract.

---

## Exceptions

None.

---

## Revision history

- **2026-11-19:** decided, in the native plugin features design.
- **2026-11-21:** recorded as an ADR, and rewritten in this structure.

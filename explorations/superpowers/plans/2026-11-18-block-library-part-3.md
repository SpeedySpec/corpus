# Block library part 3: media and media picking Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Give blocks access to the host's files through a media library contract with memory, filesystem and WordPress implementations, add the media area (image, gallery, video, audio, embed, icon, icon list, map) as a definitions package and a default outputs package, and let a person pick and upload media from the reference editor's inspector.

**Architecture:** `block-domain` gains `MediaLibraryInterface` and its value objects, plus a way for a behaviour to say a block cannot be drawn (`UnavailableBlockException`), which the view stage turns into an issue: visitors get nothing, admin mode gets the existing placeholder. `block-app` gains the upload check (type by content, size limit). Blocks store media ids; their behaviours look the item up when the block is resolved. `block-presentation` gains three media routes behind a new ability. The reference editor draws `x-control: media` and `x-control: media-list` as a picker over those routes.

**Tech Stack:** PHP 8.4, Pest, Composer path repositories, GD (demo only); plain ES modules and `node --test`.

**Spec:** `docs/superpowers/specs/2026-11-18-block-library-design.md` (in `wordpress-develop`): "Media", "The blocks" (Media), "Front-end assets", "Editor features" item 2, "Hosts".

**Repository:** `wordpress-develop/packages/speedyspec-api-block`; branch `feat/block-library-part-3` from `origin/main` once PR #10 (part 2) is merged, otherwise from `feat/block-library-part-2`. Paths are relative to its `packages/` directory.

## Global Constraints

- Everything in parts 1 and 2's Global Constraints holds: `declare(strict_types=1)`; 4 spaces; braces of classes, methods and control structures on their own line; docblocks per `wordpress-develop/.claude/rules/coding-style.md`; lines at most 120 characters; one commit per task with explicit paths, ending `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- `block-media-domain` requires only `speedyspec/block-domain`. `block-media` requires `speedyspec/block-domain` and `speedyspec/block-media-domain`. Neither depends on `block-app` or WordPress.
- Only `block-infra-wp` touches WordPress, and only through its `Ports/` interfaces, so its tests run without WordPress.
- Block names are `core/image`, `core/gallery`, `core/video`, `core/audio`, `core/embed`, `core/icon`, `core/icon-list`, `core/icon-list-item`, `core/map`, with category `media`; the package title is "Media". Their settings are fixed by this plan.
- Blocks store media ids as strings, never URLs. A media setting is `{"type": "string", "default": "", "x-control": "media", "x-media-kind": "<kind>"}`; a list of them is `{"type": "array", "items": {"type": "string"}, "default": [], "x-control": "media-list", "x-media-kind": "image"}`.
- Accepted uploads, checked by their first bytes and never by name or declared type: JPEG, PNG, GIF, WebP, AVIF, MP4, WebM, MP3, Ogg, WAV, PDF. Everything else, SVG and HTML included, is refused. The size limit is the host's, 20 MB (20 971 520 bytes) by default.
- Every `iframe` an output writes has `sandbox="allow-scripts allow-same-origin allow-presentation allow-popups"`, `loading="lazy"`, `referrerpolicy="strict-origin-when-cross-origin"` and a `title`, and is followed by a plain link to the same content. Only providers in `EmbedProviders` are ever embedded.
- URLs are printed only through `$input->escaper->url()` inside attributes; media text (alt, captions) through `text()` or, for captions, `richText()`.

## Review Focus

1. **An upload that lies about what it is** (HTML or SVG named `photo.jpg` and sent as `image/jpeg`, a PNG sent as `text/html`, a file whose first bytes are an image followed by markup). Expected: the type is decided by content alone; HTML and SVG are refused with 415 whatever they are called; an accepted file is stored and served with its detected type. Tasks 2 and 5.
2. **A media id that no longer works** (deleted, never existed, the id of a PDF in an image block, an empty id). Expected: a visitor gets nothing, admin mode gets the placeholder and an `unavailable-media` issue; a gallery draws the items it can and is only unavailable when none remain; nothing throws. Tasks 1 and 6.
3. **An embed or map URL that only looks allowed** (`https://youtube.com.evil.test/watch?v=x`, `https://evil.test/?u=youtube.com/watch`, `https://youtu.be@evil.test/x`, `javascript:`, `http://` YouTube). Expected: refused by the settings schema in the editor and again by the behaviour; the iframe source is always built from the provider's own embed address, never copied from the setting. Task 6.
4. **An upload's file name** (`../../etc/passwd.jpg`, a name with slashes, NUL bytes, 300 characters, only dots, the same name twice). Expected: the filesystem library stores every file under a name it generates inside its own directory; the given name is only kept, cleaned, as the title; two uploads never overwrite each other. Task 3.
5. **An upload at the edges** (an empty body, exactly the limit, one byte over, no `name`, a search with `page=0`, `perPage=1000` or a kind that does not exist). Expected: 400, 413 or 415 with a clear message and nothing stored; paging is clamped to 1–100 per page, never an error or an unbounded list. Tasks 4 and 5.

---

### Task 1: The media contract and unavailable blocks

**Files:**
- Create in `block-domain/src/`: `Contracts/MediaLibraryInterface.php`, `ValueObjects/MediaItem.php`, `ValueObjects/MediaSize.php`, `ValueObjects/MediaKind.php`, `ValueObjects/MediaQuery.php`, `ValueObjects/MediaPage.php`, `ValueObjects/MediaUpload.php`, `Exceptions/UnavailableBlockException.php`
- Modify: `block-domain/src/ValueObjects/IssueKind.php` (add `case UnavailableMedia = 'unavailable-media';`), `block-domain/src/ValueObjects/BuilderAbility.php` (add `case UploadMedia = 'upload-media';`), `block-app/src/Resolving/ViewStage.php`
- Test: `block-domain/tests/MediaTest.php`, `block-app/tests/Resolving/UnavailableBlockTest.php`

**Interfaces:**
- Produces:
  - `MediaKind` (string enum): `Image = 'image'`, `Video = 'video'`, `Audio = 'audio'`, `File = 'file'`; `static fromMimeType(string $mimeType): self` (`image/*` → Image, `video/*` → Video, `audio/*` → Audio, else File).
  - `MediaSize(string $name, string $url, int $width, int $height)`.
  - `MediaItem(string $id, string $url, MediaKind $kind, string $mimeType, string $title, string $alt = '', ?int $width = null, ?int $height = null, array $sizes = [])` with `@param list<MediaSize> $sizes`; `size(string $name): ?MediaSize`; `toArray(): array` (keys `id, url, kind, mimeType, title, alt, width, height, sizes`, sizes as `{name, url, width, height}`).
  - `MediaQuery(string $search = '', ?MediaKind $kind = null, int $page = 1, int $perPage = 20)`, clamping `page` to at least 1 and `perPage` to 1–100 in the constructor.
  - `MediaPage(array $items, int $total)` with `@param list<MediaItem> $items`.
  - `MediaUpload(string $name, string $mimeType, string $bytes)`; `kind(): MediaKind`. It carries an already checked upload; it does not check anything itself.
  - `MediaLibraryInterface`: `find(string $id): ?MediaItem`; `search(MediaQuery $query): MediaPage` (newest first; `search` matches the title, case-insensitive); `upload(MediaUpload $upload): MediaItem`.
  - `UnavailableBlockException(IssueKind $kind, string $message, ?string $path = null)` extends `RuntimeException`, with public readonly `kind` and `path`.
  - `ViewStage::run()` catches `UnavailableBlockException` from a behaviour before its generic `Throwable` catch, adds `new Issue($blockId, $e->kind, $e->getMessage(), $e->path)` and returns `[null, $issues, []]`, so the block is unresolved exactly like any other.

- [ ] **Step 1: Write the failing tests.** `MediaTest.php`: `MediaKind::fromMimeType('image/webp')` is Image, `'video/mp4'` Video, `'audio/ogg'` Audio, `'application/pdf'` File; `new MediaQuery(page: 0, perPage: 1000)` has page 1 and perPage 100, `perPage: 0` gives 1; `MediaItem::size('thumb')` finds a size and `size('none')` is null; `toArray()` has exactly the keys above. `UnavailableBlockTest.php`, using a behaviour that throws `new UnavailableBlockException(IssueKind::UnavailableMedia, 'The image is missing.', 'media')`: in front mode the document renders to `''` for that block and the result has one issue with kind `unavailable-media`, message `The image is missing.` and path `media`; in admin mode the placeholder output is drawn; a sibling block still renders.
- [ ] **Step 2: Run** `cd block-domain && vendor/bin/pest` and `cd block-app && vendor/bin/pest --filter=UnavailableBlock`. Expected: FAIL (classes missing).
- [ ] **Step 3: Implement** the value objects, contract, exception, enum cases and the `ViewStage` catch.
- [ ] **Step 4: Run** `block-domain` and `block-app` suites. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-domain): add the media library contract and unavailable blocks`.

---

### Task 2: Checking uploads

**Files:**
- Create: `block-app/src/Media/MediaTypeDetector.php`, `block-app/src/Media/MediaUploadPolicy.php`, `block-domain/src/Exceptions/MediaUploadRefusedException.php`
- Test: `block-app/tests/Media/MediaTypeDetectorTest.php`, `block-app/tests/Media/MediaUploadPolicyTest.php`

**Interfaces:**
- Consumes: `MediaUpload` (Task 1).
- Produces:
  - `MediaTypeDetector::detect(string $bytes): ?string`: the MIME type read from the first bytes, or null. Signatures: JPEG `FF D8 FF` → `image/jpeg`; PNG `89 50 4E 47 0D 0A 1A 0A` → `image/png`; GIF `GIF87a`/`GIF89a` → `image/gif`; WebP `RIFF....WEBP` → `image/webp`; AVIF `....ftypavif` or `ftypavis` at offset 4 → `image/avif`; MP4 `....ftyp` at offset 4 with any other brand → `video/mp4`; WebM `1A 45 DF A3` → `video/webm`; MP3 `ID3` or a frame sync `FF Fx`/`FF Ex` → `audio/mpeg`; Ogg `OggS` → `audio/ogg`; WAV `RIFF....WAVE` → `audio/wav`; PDF `%PDF-` → `application/pdf`. No `fileinfo`: the list is the policy, so it is written out.
  - `MediaUploadRefusedException(int $status, string $message)` extends `RuntimeException`, with public readonly `status` (400, 413 or 415).
  - `MediaUploadPolicy(MediaTypeDetector $detector, int $maxBytes = 20971520)`; `check(string $name, string $bytes): MediaUpload`. Refuses an empty body (400 "The file is empty."), an empty name (400 "Give the file a name."), more than `maxBytes` (413 "The file is larger than the limit of 20 MB." with the host's limit in MB), and an undetected type (415 "Only images, video, audio and PDF files can be uploaded."). The upload's type is the detected one; the declared `Content-Type` is not consulted.

- [ ] **Step 1: Write the failing tests.** Detector: one test per signature with a minimal byte string; `'<svg xmlns=...'`, `'<!DOCTYPE html>'`, `'GIF8'` (too short), `''` and `"\x00\x00\x00\x18ftyp"` with no brand all give null; `"\xFF\xD8\xFF" . '<script>'` is `image/jpeg` (the type, not a safety verdict: files are served with that type and `nosniff`). Policy: each refusal with its status and message; a PNG of exactly `maxBytes` passes and one byte more gives 413; HTML named `photo.jpg` gives 415; a PNG named `notes.txt` passes with `mimeType` `image/png`.
- [ ] **Step 2: Run** `cd block-app && vendor/bin/pest --filter=Media`. Expected: FAIL.
- [ ] **Step 3: Implement** both classes and the exception (in `block-domain`, because the API and hosts catch it). Bind `MediaTypeDetector` and `MediaUploadPolicy` in `AppServiceProvider` (no `MediaLibraryInterface` binding: that is the host's).
- [ ] **Step 4: Run** `block-app` suite. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-app): check uploads by their content and size`.

---

### Task 3: Memory and filesystem libraries, with a shared suite

**Files:**
- Create: `block-contract-tests/src/mediaLibraryContract.php` (and its `autoload.files` entry), `block-infra-memory/src/InMemoryMediaLibrary.php`, `block-infra-filesystem/src/FilesystemMediaLibrary.php`
- Modify: `block-infra-memory/src/MemoryServiceProvider.php`, `block-infra-filesystem/src/FilesystemServiceProvider.php`
- Test: `block-infra-memory/tests/MediaLibraryTest.php`, `block-infra-filesystem/tests/MediaLibraryTest.php`

**Interfaces:**
- Consumes: Task 1's contract and value objects.
- Produces:
  - `mediaLibraryContract(Closure $make): void`, a Pest `describe` block: an upload is found by its id with the same type, kind and title; `find('missing')` is null; search by part of a title, case-insensitive; search by kind; newest first; paging (`total` counts all matches, `perPage` 2 over 3 items gives 2 then 1); two uploads with the same name both exist with different ids and URLs.
  - `InMemoryMediaLibrary(string $baseUrl = 'memory://media/')`; ids `m1`, `m2`, …; `add(MediaItem $item): void` for seeding tests.
  - `FilesystemMediaLibrary(string $directory, string $baseUrl)`: stores each upload as `<id>.<ext>` (id from `bin2hex(random_bytes(8))`, extension from the detected type) and keeps an `index.json` beside the files, written to a temporary file then renamed. The given name, stripped of any path, control characters and anything outside letters, digits, space, `.`, `-`, `_`, and cut to 120 characters, is only the title (`upload` when nothing is left). Images get width and height from `getimagesize()` on the stored file; no sizes are generated (resizing is the host's, recorded). `path(string $id): ?string` gives the stored file's path for a host to serve.
  - `FilesystemServiceProvider(string $directory, ?string $mediaBaseUrl = null)` binds `MediaLibraryInterface` to `FilesystemMediaLibrary("$directory/media", $mediaBaseUrl)` only when a base URL is given.

- [ ] **Step 1: Write the failing tests.** Both packages run `mediaLibraryContract()`. The filesystem test adds: an upload named `../../etc/passwd.jpg` is stored inside the directory with a generated name and titled `passwd.jpg`; a name of only dots is titled `upload`; a 300-character name gives a 120-character title; a PNG upload has its real width and height; a second library over the same directory finds the first's uploads.
- [ ] **Step 2: Run** each package's `vendor/bin/pest`. Expected: FAIL.
- [ ] **Step 3: Implement** the suite, both libraries and the provider changes. Run `composer update speedyspec/*` where a package's lock needs the new domain classes.
- [ ] **Step 4: Run** `block-infra-memory`, `block-infra-filesystem` and `block-contract-tests` (if it has its own tests). Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-infra): keep media in memory and in a directory`.

---

### Task 4: WordPress attachments

**Files:**
- Create: `block-infra-wp/src/Ports/WpAttachmentsInterface.php`, `block-infra-wp/src/WordPress/WordPressAttachments.php`, `block-infra-wp/src/WpMediaLibrary.php`
- Modify: `block-infra-wp/src/WpServiceProvider.php`
- Test: `block-infra-wp/tests/MediaLibraryTest.php` (with an in-test fake of the port)

**Interfaces:**
- Consumes: Task 1; `mediaLibraryContract()` from Task 3.
- Produces:
  - `WpAttachmentsInterface`: `attachment(int $id): ?array` (keys `id, url, mimeType, title, alt, width, height, sizes` where sizes is `list<array{name, url, width, height}>`); `search(string $search, ?string $mimePrefix, int $page, int $perPage): array{ids: list<int>, total: int}` (newest first); `insert(string $name, string $mimeType, string $bytes): int`.
  - `WordPressAttachments` implements the port with `get_post()`, `wp_get_attachment_url()`, `wp_get_attachment_metadata()`, `wp_get_attachment_image_src()`, `get_post_meta($id, '_wp_attachment_image_alt', true)`, `WP_Query` (`post_type` attachment, `post_status` inherit, `post_mime_type`, `s`, `orderby` date) and, for insert, `wp_upload_bits()` then `wp_insert_attachment()` and `wp_generate_attachment_metadata()`. It is the only file that calls WordPress and is not unit-tested (as the other `WordPress/` classes).
  - `WpMediaLibrary(WpAttachmentsInterface $attachments)`: ids are the attachment ids as strings; a non-numeric id is not found; a kind maps to the MIME prefix `image/`, `video/`, `audio/`, and File to `application/`.
  - `WpServiceProvider` binds `MediaLibraryInterface` to `WpMediaLibrary`.

- [ ] **Step 1: Write the failing test.** Run `mediaLibraryContract()` over `WpMediaLibrary` with a fake port that keeps attachments in an array; add: `find('12abc')` and `find('-1')` are null without calling the port; sizes come through as `MediaSize`s.
- [ ] **Step 2: Run** `cd block-infra-wp && vendor/bin/pest`. Expected: FAIL.
- [ ] **Step 3: Implement** the port, the WordPress adapter, the library and the binding.
- [ ] **Step 4: Run** `block-infra-wp` suite. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-infra-wp): use WordPress attachments as the media library`.

---

### Task 5: Media routes in the builder API

**Files:**
- Modify: `block-presentation/src/ApiRequest.php`, `block-presentation/src/BuilderApi.php`, `block-builder/demo/server.php` (pass headers only; the rest of the demo is Task 8)
- Modify: `block-presentation/composer.json` (`speedyspec/block-infra-memory` as a dev dependency, for the tests)
- Test: `block-presentation/tests/MediaRoutesTest.php`

**Interfaces:**
- Consumes: Tasks 1 and 2.
- Produces:
  - `ApiRequest(string $method, string $path, array $query = [], string $body = '', array $headers = [])`; header names are lower-cased by the constructor; `header(string $name): ?string`.
  - `GET /media?search=&kind=&page=&perPage=` (needs Edit) → `{"items": [MediaItem::toArray()...], "total": n, "page": p, "perPage": n}`; an unknown `kind` is 400 "Unknown media kind.".
  - `GET /media/{id}` (needs Edit) → `{"item": {...}}`, or 404.
  - `POST /media?name=` (needs UploadMedia) with the file as the raw body → 201 `{"item": {...}}`; a `MediaUploadRefusedException` becomes its status with `{"error": message}`. *My call:* the spec has the type travel in `Content-Type`; the client still sends it, but the stored type is the detected one, since the spec also says the check is by content and a declared type that disagrees is exactly what Review Focus 1 is about.
  - When no `MediaLibraryInterface` is bound, each media route answers 501 "This site has no media library.".

- [ ] **Step 1: Write the failing tests** with `InMemoryMediaLibrary` and an access stub: listing and paging (`perPage=1000` answers with `perPage` 100); kind filter; bad kind 400; find and 404; upload 201 then found by `GET`; HTML upload with `Content-Type: image/jpeg` 415; over the limit (policy built with a small `maxBytes`) 413; empty body 400; Edit without UploadMedia gives 403 on `POST` but 200 on `GET`; no library bound gives 501; header names are case-insensitive.
- [ ] **Step 2: Run** `cd block-presentation && vendor/bin/pest --filter=MediaRoutes`. Expected: FAIL.
- [ ] **Step 3: Implement** the request headers and routes; the demo passes `getallheaders()` (or `$_SERVER` `HTTP_*` and `CONTENT_TYPE` where that is missing).
- [ ] **Step 4: Run** `block-presentation` suite. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-presentation): list, find and upload media through the builder API`.

---

### Task 6: Media block definitions

**Files:**
- Create: `block-media-domain/` (`composer.json`, `phpunit.xml`, `src/MediaTypesPackage.php`, `src/EmbedProviders.php`, `src/Embed.php`, `src/ImageBehaviour.php`, `src/GalleryBehaviour.php`, `src/PlayerBehaviour.php`, `src/EmbedBehaviour.php`, `src/MapBehaviour.php`, `tests/Pest.php`, `tests/PackageContractTest.php`, `tests/TypesTest.php`, `tests/BehavioursTest.php`, `tests/EmbedProvidersTest.php`)

**Interfaces:**
- Consumes: Task 1; `InMemoryMediaLibrary` (Task 3) as a dev dependency for tests.
- Produces:
  - `MediaTypesPackage(MediaLibraryInterface $library)` and `static from(BlockServiceContainer $container): self`; name `core/media-types`, title "Media". Every block has a title, a description and an example, as the text blocks do. Settings (every block also takes the shared `core/style` settings through its outer element):
    - `core/image`: `media` (media setting, kind image), `alt` (`["string", "null"]`, default null: null uses the item's alt, `""` marks it decorative), `caption` (rich), `link` (string, default `""`), `size` (string, pattern `^[a-z0-9_-]{0,32}$`, default `""` for the full file).
    - `core/gallery`: `media` (media list), `columns` (integer 1–6 or responsive map, default 3, `x-control: responsive`), `caption` (rich).
    - `core/video`: `media` (kind video), `poster` (kind image), `controls` (boolean, default true), `autoplay` (boolean, default false; autoplay is always muted), `loop` (boolean, default false), `caption` (rich).
    - `core/audio`: as video without `poster`.
    - `core/embed`: `url` (string, `pattern` built from `EmbedProviders::PATTERN`), `title` (string, default "Embedded content"), `caption` (rich).
    - `core/icon`: `name` (pattern `^[a-z0-9-]{1,40}$`, default `star`), `label` (string, default `""` meaning decorative).
    - `core/icon-list`: `icon` (as `core/icon`'s name, default `check`), children `core/icon-list-item`; provides context `core/icon` from its `icon`.
    - `core/icon-list-item`: `icon` (string, default `""` meaning the list's), `text` (rich); parents `core/icon-list`; uses `core/icon`.
    - `core/map`: `address` (string, at most 200 characters), `latitude` (`["number", "null"]`, −90 to 90), `longitude` (`["number", "null"]`, −180 to 180), `zoom` (integer 1–19, default 15), `title` (string, default "Map").
  - `Embed(string $provider, string $src, string $link)`.
  - `EmbedProviders`: `PATTERN` (the anchored regex the schema uses) and `static embedFor(string $url): ?Embed`. Providers, *my call*: YouTube (`https://www.youtube.com/watch?v=ID`, `https://youtube.com/watch?v=ID`, `https://youtu.be/ID`, `https://www.youtube.com/shorts/ID`, ID `[A-Za-z0-9_-]{11}`) → `https://www.youtube-nocookie.com/embed/ID`; Vimeo (`https://vimeo.com/NUM`) → `https://player.vimeo.com/video/NUM`. The URL is parsed with `parse_url`, the host compared exactly, and anything with a user, password or port refused. `static mapFor(float $lat, float $lon, int $zoom): Embed` builds the OpenStreetMap embed (`https://www.openstreetmap.org/export/embed.html?bbox=…&layer=mapnik&marker=lat,lon`, bbox half-width `360 / 2 ** zoom`) and link (`https://www.openstreetmap.org/?mlat=…&mlon=…#map=zoom/lat/lon`).
  - Views (what outputs read):
    - image `{url, width, height, alt, srcset, caption, link}`; `srcset` lists the item's sizes and the full file as `url Nw`, by width; the chosen `size` gives `url`, `width`, `height`, falling back to the full file;
    - gallery `{items: list<{url, width, height, alt}>, columns, caption}`;
    - player `{kind, url, mimeType, poster, controls, autoplay, muted, loop, caption}` with `muted` true whenever `autoplay` is;
    - embed `{provider, src, link, title, caption}`;
    - map `{src, link, title, address}` (`src` empty when there are no coordinates; with only an address, `link` is `https://www.openstreetmap.org/search?query=` + the encoded address);
    - icon and icon list blocks have no behaviour.
  - Unavailable: an empty or unknown media id, or one of the wrong kind, throws `UnavailableBlockException(IssueKind::UnavailableMedia, …, 'media')` (messages "Choose an image.", "The chosen image no longer exists.", "The chosen file is not an image."; likewise for video and audio). A gallery skips missing and wrong-kind items and is unavailable only when none remain. A missing poster is dropped, not fatal. An embed URL `embedFor()` refuses, and a map with neither coordinates nor address, are unavailable with path `url` or `address`.

- [ ] **Step 1: Write the failing tests.** `PackageContractTest` in definitions mode. `TypesTest`: names, category, parents and children, defaults, and that the schema refuses each Review Focus 3 URL. `EmbedProvidersTest`: each accepted form gives the provider's embed address; each Review Focus 3 URL, `http://youtu.be/…`, a 10-character id and `https://www.youtube.com:8443/watch?v=…` give null; `mapFor(51.5, -0.12, 15)` gives the expected bbox and link. `BehavioursTest` over `InMemoryMediaLibrary`: an image with sizes gives `srcset` ordered by width and honours `size`; `alt` null uses the item's, `""` stays empty; each unavailable case gives its issue through a resolved document (Review Focus 2); a gallery with one missing of three draws two; autoplay forces muted; a map with coordinates has an OpenStreetMap `src` and with only an address has an empty `src` and a search link.
- [ ] **Step 2: Run** `cd block-media-domain && vendor/bin/pest`. Expected: FAIL.
- [ ] **Step 3: Implement** the package, providers and behaviours.
- [ ] **Step 4: Run** `block-media-domain` suite. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-media): add the media block definitions`.

---

### Task 7: Media block outputs

**Files:**
- Create: `block-media/` (`composer.json`, `phpunit.xml`, `src/MediaPackage.php`, `src/MediaOutput.php`, `src/Icons.php`, `tests/Pest.php`, `tests/PackageContractTest.php`, `tests/OutputTest.php`)

**Interfaces:**
- Consumes: Task 6's views and `EmbedProviders`.
- Produces:
  - `MediaPackage(StyleCollectorInterface $styles)`, `static from(BlockServiceContainer $container): self`; name `core/media`, title "Media, drawn"; registers `MediaOutput` for the nine blocks.
  - Markup, the block's attribute bag always on the outermost element:
    - image: `<figure{attrs}><img src srcset sizes="100vw" width height alt loading="lazy" decoding="async"></figure>`, wrapped in `<a href>` when `link` passes `url()`, with `<figcaption>` (marked `caption`) when there is a caption or while editing;
    - gallery: `<figure{attrs} class="b-…">` holding `<ul>` of `<li><img …></li>`, with the column count by breakpoint sent to the style collector as a grid, as `core/grid` does;
    - video and audio: `<figure{attrs}><video|audio src controls? autoplay? muted? loop? playsinline? poster? preload="metadata">` with a plain download link inside the element as its fallback;
    - embed and map: `<figure{attrs}><iframe …>` with the attributes from Global Constraints, then `<a href=link>` with the title; a map without `src` draws only the link;
    - icon: `<span{attrs} class="b-icon">` holding the SVG from `Icons`, `role="img" aria-label` when it has a label, `aria-hidden="true"` otherwise; an unknown name draws nothing in front mode and `<span{attrs}>` with the name as text in admin mode;
    - icon list: `<ul{attrs} class="b-icon-list">`; item: `<li>` with the icon (its own or the list's) and the text in a `<span>` marked `text`.
  - `Icons::svg(string $name): ?string` and `Icons::NAMES`: an original set of 16 line icons on a 24-unit grid using `currentColor` (`check`, `x`, `star`, `heart`, `arrow-right`, `arrow-left`, `plus`, `minus`, `info`, `alert`, `phone`, `mail`, `map-pin`, `clock`, `calendar`, `user`). They are written for this package, not copied from another icon set.

- [ ] **Step 1: Write the failing tests.** `PackageContractTest` in outputs mode with `MediaTypesPackage` given an `InMemoryMediaLibrary`. `OutputTest`: each block's markup for a seeded item; the attribute bag on the outer element; a `javascript:` image link is dropped; every iframe carries every attribute in Global Constraints and its `src` is the provider's; a caption is marked while editing and absent for visitors when empty; an unknown icon draws nothing for visitors and its name for editors; an icon list item without an icon uses the list's.
- [ ] **Step 2: Run** `cd block-media && vendor/bin/pest`. Expected: FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** `block-media` suite. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-media): add the media block outputs`.

---

### Task 8: Picking media in the reference editor, the demo and documentation

**Files:**
- Create: `block-builder/src/media.js`, `block-builder/tests/media.test.js`
- Modify: `block-builder/src/api.js`, `block-builder/src/index.js`, `block-builder/src/builder-element.js`, `block-builder/src/styles.js`, `block-builder/tests/api.test.js`, `block-builder/demo/server.php`, `block-builder/demo/composer.json`, `block-builder/README.md`, repository `README.md`, `wordpress-develop/docs/superpowers/specs/2026-11-18-block-library-design.md` (a part 3 note)

**Interfaces:**
- Consumes: Task 5's routes; Task 6's `x-control` and `x-media-kind` settings.
- Produces:
  - `api.js`: `media({ search, kind, page, perPage })`, `mediaItem(id)`, `upload(file)` (sends the `File` as the raw body with `?name=` and the file's own `Content-Type`; a refusal comes back as `ApiError` with the server's message).
  - `media.js` (no DOM): `mediaQuery(kind, search, page)` → the object `media()` takes, with `kind` omitted when empty; `withMedia(list, id)` (appends, ignoring an id already there), `withoutMediaAt(list, index)`, `movedMedia(list, from, to)`; `accepts(kind)` → the `accept` attribute for the file input (`image/*`, `video/*`, `audio/*`, or the full accepted list for File).
  - Inspector: `x-control: media` draws the chosen item's thumbnail (images) or title, with Choose and Remove; `x-control: media-list` draws the chosen thumbnails in order, each with remove and move buttons, and Add. Choose and Add open a picker panel in the inspector: a search box, a grid of items of the setting's kind (24 per page, with More), and Upload, which uploads, then selects the new item. The picker remembers nothing between openings. Choosing sends an ordinary `updateSettings`.
  - Demo: loads `MediaTypesPackage::from($container)` and `MediaPackage::from($container)` after the text packages; binds `FilesystemServiceProvider($dir, '/media-files/')`; serves `/media-files/<id>.<ext>` from the library's directory with the stored type, `X-Content-Type-Options: nosniff` and `Content-Disposition: inline` (and 404 for anything else); the access stub allows UploadMedia; on first start it draws three sample PNGs with GD (flat colour fields with a shape, so they are original) and uploads them, and seeds the page with an image, a gallery, an embed of a YouTube URL, an icon list and a map with coordinates.

- [ ] **Step 1: Write the failing tests.** `media.test.js` for each helper (an id already in the list is not added twice; moving past either end leaves the list as it was; `mediaQuery('', 'cat', 2)` has no `kind`). `api.test.js`: `upload()` sends the raw body, the encoded name and the file's type, and `media()` builds its query string.
- [ ] **Step 2: Run** `cd block-builder && npm test`. Expected: FAIL.
- [ ] **Step 3: Implement** the helpers, client methods, inspector controls, picker and demo; write the README sections ("Media" under the block areas, with the library contract and its three implementations; picking media under the builder) and the design note.
- [ ] **Step 4: Run** every PHP suite and `npm test`; start the demo (`.claude/launch.json` `block-builder-demo`) and check by hand: choose an image from the picker, upload a PNG, try to upload an `.html` file renamed `.jpg` (refused with the server's message), add and reorder gallery images, delete a media file from the demo directory and see the placeholder and issue, and confirm visitors see nothing for it. Expected: PASS.
- [ ] **Step 5: Commit** `feat(block-media): pick and upload media in the reference editor and show the media blocks in the demo`.

---

## Out of scope for this plan

Generating image sizes in the filesystem library (the host's; WordPress already does), editing or deleting media from the picker, alt text editing on the library item itself, drag-and-drop upload onto the page, choosing an icon from a visual list (a text field with the names in its description for now), oEmbed discovery and any provider beyond YouTube, Vimeo and OpenStreetMap, the other areas (parts 4 to 6), and the remaining editor features (part 7).

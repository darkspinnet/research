# Static asset recovery

DarkSpinner no longer needs to carry client-owned web images in its executable.
The first-run content warm-up reconstructs known resources under
`darkspin/cache/www/static`, and the server exposes that directory through its
existing static API routes.

## Recovered from the vanilla client

The recovery recipe contains package identities, sizes, and SHA-256 checksums,
not image payloads. It currently prepares 119 files:

- 100 creature thumbnails from `Data/PreBaked.package`.
- 17 account avatars from `Data/UI.package`.
- The green and red status lights from `Data/Web.package`. Their decoded pixels
  match the prior server images, although the PNG encoding differs.

Every prepared member is selected by DBPF type, group, and instance. Existing
cache files are checksum-validated and skipped; missing or damaged files are
atomically regenerated.

## Unresolved server assets

The 39 files under `notes/assets/www/static` have no exact or pixel-identical
resource in the vanilla client packages inspected. They are retained here as
research inputs and are not embedded or copied into the runtime cache. Until a
source or replacement is selected, requests for these paths return `404`.

The unresolved set consists of:

- Eight Game-style CSS files.
- The `PirulenRg-Regular.ttf` fallback font. `Web.package` and `UI.package`
  contain a different 56,896-byte build of the same font family, but it is not
  the same file as the prior 54,152-byte server fallback.
- Twenty button, checkbox, text-field, title-bar, yellow-light, and
  registration-background PNG files.
- Six JavaScript files, including the currently unreferenced 813,732-byte
  AngularJS distribution.
- The registration and announcement HTML/CSS documents.

The closest yellow status-light resource is `Web.package` type `0x2F7D0004`,
group `0x00000000`, instance `0x00000000CC6ECDAA`, but its pixels are not
identical. Several checkbox and title-bar images have same-sized client
candidates with visibly different pixels; they must not be treated as recovered
without an explicit replacement decision.

## Follow-up

Decide whether the registration surface is still required now that DarkSpinner
creates local profiles directly. If it is required, source or redesign the
small HTML/CSS/JS surface and its control textures. If it is not required,
remove those routes and retain only the package-derived thumbnail and avatar
cache.

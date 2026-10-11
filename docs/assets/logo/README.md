# Wirestead logo assets

These assets use the square bracket identity from
[wirestead/wirestead](https://github.com/wirestead/wirestead/tree/main/assets).
The visual wordmark is lowercase `wirestead`; use `Wirestead` in prose.

## Files

- `light/wirestead-logo-horizontal.png` and
  `light/wirestead-logo-horizontal-readme-light.png`: light-background lockups.
- `dark/wirestead-logo-horizontal.png` and
  `dark/wirestead-logo-horizontal-readme-dark.png`: dark-background lockups.
- `light/wirestead-logo.png` and `dark/wirestead-logo.png`: square avatar badges.
- `wirestead-favicon.svg` and `wirestead-favicon-64.png`: documentation favicons.
- [guide/Wirestead-brand-guide.md](guide/Wirestead-brand-guide.md): current identity
  and usage rules. Asset paths in the guide are relative to the core repository.
- `guide/wirestead-brand-guide.png` and `feature/wirestead-feature.png`: historical
  artwork, retained for reference; do not use as current brand assets.

## Updating

Copy matching production exports from the core repository's `assets/` directory.
The horizontal files map to `wirestead-logo-horizontal.png` (light) and
`wirestead-logo-dark.png` (dark); square badges map to
`wirestead-icon-light-512.png` and `wirestead-icon-dark-512.png`.
Keep the existing documentation filenames so Doxygen's light/dark logo switch
continues to work. Copy the favicon pair and brand guide from the same revision.

## Usage

Prefer horizontal lockups for documentation headers and square badges for avatars.
Choose the variant for the background, preserve aspect ratios, and keep clear space
around the mark. Use GitHub's `#gh-light-mode-only` and `#gh-dark-mode-only` image
fragments for theme-specific Markdown logos.

# Contributing

Thanks for considering a contribution. This repository documents a
production-validated email security framework -- changes should reflect real
deployment experience, not abstract best practice.

## Ways to Contribute

- **Factual corrections** -- wrong RFC reference, deprecated cmdlet, broken
  link. These are always welcome via PR.
- **Real-world tuning** -- "this default broke X in production, here is what
  worked." Add to the relevant doc with a brief context note.
- **New chapter** -- a security control not yet covered (e.g. additional
  provider integration). Open an issue first to discuss scope.
- **Translations** -- not currently in scope.

## Before You Open a PR

1. Read [`SECURITY.md`](SECURITY.md) -- particularly the redaction policy.
   Real domain names, tenant identifiers, and operational identifiers must
   not appear in committed docs.
2. Run markdown lint locally if you have it installed:
   ```bash
   npx markdownlint-cli2 '**/*.md'
   ```
3. If you add external links, verify they resolve.
4. Keep the writing style consistent: terse, technical, no emoji.

## What Gets Merged

- Changes that are correct, cite a source where appropriate, and survive the
  CI checks (markdownlint, link check, OSINT redaction)
- Additions that fit the existing structure (`/dkim/`, `/dmarc/`, etc.)
- Removals or simplifications -- this framework is a living doc, not an
  archive

## What Does Not Get Merged

- Promotional content for security vendors
- Untested guidance (if you have not deployed it, mark it explicitly as
  unverified)
- Changes that reintroduce real operational identifiers (the CI
  `osint-check` job will block these)
- Cosmetic-only changes that do not improve clarity

## Reporting Security Issues

See [`SECURITY.md`](SECURITY.md) for the responsible disclosure process.
Documentation defects that could lead readers to a less-secure
configuration are in scope for security reporting.

## Local Setup

The repo is documentation only -- no build step. To preview a doc as it
will render on GitHub:

```bash
# Install once
npm install -g markdown-preview

# Preview one file
markdown-preview path/to/file.md
```

CI runs on every PR:
- `markdownlint-cli2` against `.markdownlint-cli2.yaml`
- `lychee` link check against all markdown files
- `trufflehog` secret scan
- Custom OSINT redaction check

A passing CI run is required for merge.

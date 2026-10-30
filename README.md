# Feynos Notes

OneNote-style notes app for **Android**, **Fedora (.rpm)** and **Windows**, built entirely from
[`.github/workflows/feynos-notes.yml`](.github/workflows/feynos-notes.yml). Every push to `main`
builds all three and publishes them under **Releases**.

- Notebooks → colored sections → pages and subpages
- Rich text: headings, colors, highlight, lists, to-do checkboxes, tags, tables, links, pictures, hand-drawn sketches
- Search across every page, recycle bin, light/dark theme
- **Backups to Google Drive & Proton Drive**
  - Desktop: add the folders your Drive / Proton Drive apps sync; backups run automatically and on close, keeping the last N
  - Android: *Back up now* opens the share sheet → pick Drive or Proton Drive
  - Restore from any backup file (merge or replace)

Optional Android signing secrets: `ANDROID_KEYSTORE_B64`, `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS`, `ANDROID_KEY_PASSWORD`.

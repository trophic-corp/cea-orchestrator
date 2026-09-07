# Starlight documentation site

Not yet scaffolded — no site framework has been installed here. `docs-writer` is the owning
agent; the first `/ship` or `/extend` pipeline that needs a published page should have
`docs-writer` run `npm create astro@latest -- --template starlight` (or equivalent) here
rather than this being pre-built with no content to justify its structure.

Once scaffolded, every `/ship`/`/extend` pipeline adds or updates a page here in the same PR
as the code — see `docs/pipeline/README.md`.

# Wirestead Docs

Documentation repository for the `wirestead` C++20 core library.

Core library:
https://github.com/wirestead/wirestead

This repository contains:

- user guides
- contributor guides
- architecture notes
- API stability policy
- transport feature matrix
- Doxygen API reference configuration

Generated documentation is not committed. Doxygen output is produced under
`build/doxygen/` by local scripts and GitHub Actions.

The published site (API reference + coverage report), built from the latest
published `wirestead` release, is deployed to
https://wirestead.github.io/wirestead-docs/ by `.github/workflows/pages.yml`.

## Documentation

- [Documentation index](docs/index.md)
- [User guide](docs/user/index.md)
- [Contributor guide](docs/contributor/index.md)
- [Doxygen configuration](doxygen/)

## Local validation

Generate the Doxygen API reference before publishing documentation changes that
touch API pages, snippets, or Doxygen configuration:

```bash
./scripts/generate_docs.sh
```

The script writes HTML output to `build/doxygen/html/`. The GitHub Actions
Doxygen workflow runs `doxygen doxygen/Doxyfile` and uploads the generated HTML
as the `doxygen-html` workflow artifact.

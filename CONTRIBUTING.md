# Contributing to LGD-JSON

Thanks for your interest in the Linked Geometry Document (LGD) format. LGD-JSON is a
pre-1.0 specification under active development, and feedback on the schema, the
documentation, and the examples is welcome.

This document explains how to report issues, propose changes to the format, and
submit pull requests.

## Reporting issues

Use [GitHub Issues](https://github.com/pizzanapoli/lgd-json-specification/issues) for bug
reports, questions, and suggestions. Please label your issue so it can be triaged:

- **bug** — a schema that accepts invalid data or rejects valid data, an incorrect
  example, or an error in the specification text.
- **enhancement** — a proposed addition or change to the format.
- **question** — a request for clarification about how to model something in LGD.

Before opening a new issue, search existing issues to avoid duplicates. For a bug,
include the smallest LGD document that reproduces the problem, the schema file you
validated against, and the validator you used.

## Proposing changes to the format

The format is the contract that all LGD documents and tools depend on, so changes
to it are discussed before they are implemented.

1. **Open an issue first.** Describe the problem the change solves and, where
   relevant, how other formats (GeoJSON, CityJSON, IFC, GeoSPARQL) handle the same
   case.
2. **Wait for discussion** before opening a pull request. This avoids work on a
   change that may need a different shape.
3. **Every schema change must include**, in the same pull request:
   - an **example file** under `examples/` that exercises the change (new or
     updated), validating against the relevant schema;
   - a **`CHANGELOG.md` entry** under the appropriate version section describing the
     change;
   - an update to **`README.md`** if the change affects documented behaviour.

## Versioning policy

The project follows [Semantic Versioning](https://semver.org/). While the format is
pre-1.0:

- **Minor bumps** (`0.x.0`) may include **breaking changes** to the format.
- **Patch bumps** (`0.0.x`) are **additive or fixes only** and do not break existing
  valid documents.

After 1.0, breaking changes will be reserved for major version bumps.

## Style and validation

- **Schema files** (`schema/`) and **example files** (`examples/`) use **2-space
  indentation** and are written in JSON Schema Draft 2020-12.
- **Examples must validate** against the relevant schema before you submit them. Use
  any Draft 2020-12 validator, for example:

  ```sh
  # using ajv-cli (npm i -g ajv-cli ajv-formats)
  ajv validate --spec=draft2020 -c ajv-formats \
    -s schema/linkedgraph.schema.json \
    -d examples/urban-graph.json
  ```

- Keep cross-references between schema files relative (e.g. `objects.schema.json`)
  so the schemas resolve correctly both locally and when served from a release tag.
- Match the surrounding style of the file you are editing — naming, key ordering,
  and the level of inline documentation.

## Pull request checklist

Before requesting review, confirm:

- [ ] The change was discussed in an issue (for format changes).
- [ ] Affected example files validate against their schema.
- [ ] `CHANGELOG.md` has an entry for the change.
- [ ] `README.md` is updated if documented behaviour changed.
- [ ] Indentation and style match the existing files.

## License

By contributing, you agree that your contributions will be licensed under the
[CC-BY 4.0](LICENSE) license that covers this repository.

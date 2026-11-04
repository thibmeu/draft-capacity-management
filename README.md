<!-- regenerate: off -->

# APPA: Anonymous Presentation for Proportionate Access

Working draft of a MoLE-based approach to limiting clients behind shared
intermediaries without requiring stable client identifiers at origins.
This is an unpublished editor's copy being developed toward an individual -00.

- [Editor's copy](https://thibmeu.github.io/draft-brian/draft-meunier-webbotauth-appa.html)
- [Contributing](CONTRIBUTING.md)
- [Draft source](draft-meunier-webbotauth-appa.md)

## Build locally

Install Git, GNU Make, Python 3 with pip and venv, Ruby with Bundler, and Node.js
with npm. Then run:

```sh
make deps
make
```

The Makefile fetches [i-d-template](https://github.com/martinthomson/i-d-template)
into `lib/` and installs its Python and Ruby tools there. The diagram dependency
is installed into `node_modules/`. The first build needs network access for tools
and bibliography entries.

The build produces `draft-meunier-webbotauth-appa.html` and
`draft-meunier-webbotauth-appa.txt`. Generated files and local review references
under `.review/` are ignored by Git.

Building locally does not publish the draft. The inherited GitHub workflows can
publish an editor's copy on pushes and submit a draft on `draft-*` tags or a
manual publish-workflow run.

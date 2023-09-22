# design-docs

Workspace for all Storj design documents

## Conventions

- All the documents are in the root of this repository.
- Documents are in [Github flavored markdown](https://github.github.com/gfm/).
- The name of the documents starts with the date that the writer starts to write it, with the format
  `YYYYMMDD-`, followed with the title of the document transformed to lowercase and hyphen-separated
  (replace white spaces with `-`) and the `.md` file extension.
- Each document has a frontmatter with the following required fields:
  - `tags`: A list of tags which classifies the document.

  The title of the document must be separated with one blank line of the frontmatter separator.
- Additional files linked by the document are accepted, however, we discourage them when there are
  valid alternatives for not requiring them. For example, [mermaid diagrams](https://mermaid.js.org/)
  instead of diagrams that cannot be expressed in the same markdown document and not rendered by
  Github.

  The additional files must be store in a directory located in the root of this repository with the
  same name of the document without the `.md` extension.

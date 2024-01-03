# design-docs

Workspace for all Storj design documents

## Conventions

- All the documents are in the root of this repository (see [exceptions](#exception))
- Documents are in [Github flavored markdown](https://github.github.com/gfm/).
- The name of the documents starts with the date that the writer starts to write it, with the format
  `YYYYMMDD-`, followed with the title of the document transformed to lowercase and hyphen-separated
  (replace white spaces with `-`) and the `.md` file extension.
- Each document has a front matter with the following required fields:
  - `tags`: A list of tags which classifies the document.

  The title of the document must be separated with one blank line of the front matter separator.
- Additional files linked by the document are accepted, however, we discourage them when there are
  valid alternatives for not requiring them. For example, [mermaid diagrams](https://mermaid.js.org/)
  instead of diagrams that cannot be expressed in the same markdown document and not rendered by
  Github.

  The additional files must be store in the _assets_ directory and their name must start with the
  same name of the document without the `.md` extension. When there are more than one additional
  file, only one file may match the document file name excluding the extension, the file name of the
  rest must follow the document file name with some words related to what they describe.

  The additional file name must follow the same conventions of the document file names mentioned
  above.
- The documents must use the last version of the template document
- [00000000-template.md](00000000-template.md). The matches the structure that we outlined in our
  Clonfluence page
  [proposal design document process](https://storjlabs.atlassian.net/wiki/spaces/delivery/pages/2548137985/Design+Doc+Process+Proposal+2023)
  (non-public document).


### Exceptions

We have imported documents from they were stored before.

We endeavored to apply these conventions to them except changing their format to the new template,
however, there may be some other conventions that weren't applied to some of them due to they
weren't straightforward to adapt.

All of the documents were imported without setting any `tag`. Pull requests are welcome to add
`tags` to them.

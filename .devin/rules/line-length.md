---
description: "Enforce the 79-character line limit in all files"
trigger: always_on
---

# Line Length: 79 Characters Max

- **Every line must be 79 characters or fewer**, in every file type —
  Python, TypeScript/TSX, Markdown, etc. This is not Python-only.
- **Always verify after editing or creating a file.** Check line lengths
  before considering a task complete, e.g.:
  `awk 'length > 79 {print FILENAME ":" NR ": " length " chars"}' <file>`
- When fixing violations, match the file's existing style: split imports
  across multiple lines, break JSX props / function arguments onto their
  own lines, and wrap long strings or expressions.
- Fix violations you introduce; also fix pre-existing long lines in
  files you are already modifying.

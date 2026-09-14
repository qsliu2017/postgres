# Book Authoring Rules

- For a code snippet with a description, put the description itself in a collapsible `<details>` summary and the snippet in its body. The summary is part of the prose: do not leave the same introductory description immediately before the `<details>` block or replace it with a generic label such as "Example" or "Definition".

  ````md
  <details>
  <summary>PostgreSQL performs this operation in <code>function_name()</code>.</summary>

  ```c
  /* path/to/source.c:function_name */
  /* code snippet */
  ```

  </details>
  ````

- Markdown is not rendered inside raw HTML summaries. Use `<code>...</code>`, not Markdown backticks, for inline code inside `<summary>`.

- Start each PostgreSQL source-code snippet with a location comment in this format:

  ```c
  /* <path>:<optional function name> */
  ```

- Do not put source line numbers in snippet location comments. Line numbers change across PostgreSQL versions.

- Include the enclosing function signature and braces when a source snippet excerpts a function body.

- Prefer active voice: write “`function()` performs the operation,” not “the operation is performed in `function()`.”

- Format call paths as top-down call trees containing only the important, critical paths. Start with the entry point. Keep a linear dispatch chain on one line, separated by `->`. Indent branches beneath their caller, and keep sibling or alternative branches at the same indentation. A call may include a brief comment or condition. Use function names only; do not append source locations or line numbers.

  ````md
  ```text
  entry -> dispatcher -> caller
    -> parallel caller
    -> conditional caller, when a condition is true
      -> sub caller
  ```
  ````

- Structure a code walkthrough as: section heading, optional SQL example, visible call-path graph, then collapsible source-code descriptions. Do not include debugger setup such as attaching to a backend or setting breakpoints. Follow the graph's order in the walkthrough and merge methods from the same graph line into one source block. Put a description that applies to one graph line in that block's `<summary>`; put descriptions that span multiple graph lines in visible prose outside the blocks.

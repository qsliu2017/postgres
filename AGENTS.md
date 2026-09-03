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

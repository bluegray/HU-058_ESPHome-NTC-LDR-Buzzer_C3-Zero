# Instructions to AI Agents

## Permissions

- Ask permission before accessing the file system anywhere for anything.
- Never assume permission from previous interactions, always ask again.
- Make sure you have explicit permission to read or modify files before doing so.
  If a prompt is vague and you are not sure, ask again.

## README Guidelines

Write READMEs that are professional, concise, and plain-spoken.

- Open with one or two sentences on what the project does and who it's for.
- Then cover, in order and only if relevant: quick start (clone to running), usage examples, configuration (table of env vars), development (test, lint, build).
- Use short sentences, common words, and no marketing language, filler, or unexplained acronyms.
- Put every command in a fenced code block with a language tag. Keep commands copy-paste ready, with no `$` prompts and `<placeholders>` for values.
- Only document what exists. Verify commands, paths, and versions against the project, and list minimum prerequisite versions.
- Keep it readable in under two minutes. Move long guides and API details to `docs/` and link to them.
- Update the README whenever setup or commands change.

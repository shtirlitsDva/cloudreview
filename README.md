# cloudreview

A hosted review surface for agent-authored documents.

An AI agent (Claude Code, or any other) publishes a Markdown document — a
questionnaire, a spec, an analysis, a code review — to a web page. A human opens
that page in a browser, reads it as rendered HTML, and annotates it
paragraph-by-paragraph: typed comments, dictated comments, pasted images. When
they click **Finish**, the agent pulls the annotations back and continues work.

The problem it solves: reviewing a long agent-authored text inside a terminal
chat is slow and lossy. One question at a time, each blocking on a reply.
cloudreview turns that serial interrogation into a single asynchronous pass.

## Status

Early. The design is being charted with the
[wayfinder](https://github.com/mattpocock) mapping process — see the
`wayfinder:map` issue in this repo for the destination, the decisions taken so
far, and the open questions.

Nothing here is implemented yet.

## Security

This is a public repository. No credentials, connection strings, keys, or
tokens belong in it — ever. See `.gitignore` for the ignore rules and the
security decisions recorded on the map for how secrets are handled at runtime.

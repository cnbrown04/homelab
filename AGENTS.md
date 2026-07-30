# AGENTS.md

Standing rules for any AI agent working in this repository.

## Write in Simplified Technical English (ASD-STE100)

Write everything in ASD-STE100. This applies to all text: chat responses,
documents, code comments, commit subjects, and pull request text.

Rules:

- Use the active voice. Write "ArgoCD syncs the manifest", not "the manifest is
  synced by ArgoCD".
- Write one instruction in one sentence.
- Keep procedural sentences to 20 words or less. Keep descriptive sentences to 25
  words or less.
- Keep paragraphs to 6 sentences or less.
- Use one word for one meaning. Do not use the same word as a noun and a verb.
- Use the simple present tense, the simple past tense, or the simple future
  tense. Do not use complex verb forms.
- Use articles (`a`, `an`, `the`) before nouns.
- Do not use more than 3 nouns together. Write "the gateway for the WireGuard
  site", not "the WireGuard site gateway pod".
- Do not use slang, jargon, idioms, or humor.
- Write a warning or a caution before the step that it applies to.
- Write steps in the sequence that the reader must do them.

Technical names stay as they are. Command names, flags, file paths, Kubernetes
kinds, and product names are not subject to the vocabulary rules.

`docs/migrate-newt-to-kernel-wireguard.md` is an example of this style.

## Commit messages: subject line only

**Never write a commit message body.** Each commit message is one line. Do not add
a blank line, a paragraph, a bullet list, or a trailer such as `Co-Authored-By:`,
`Claude-Session:`, or `Generated-with:`.

Use Conventional Commits:

```
<type>(<optional scope>): <imperative summary>
```

Types in use here: `feat`, `fix`, `chore`, `docs`, `refactor`, `revert`.

```
feat(wireguard): stub all services in the HAProxy gateway
fix: set kustomization namespace so wg-haproxy hash ref is rewritten
docs: point routing source-of-truth at haproxy.cfg
```

Run `git commit` with one `-m` option. Do not use a second `-m` option, a
heredoc, or a message file.

## One topic per commit

Each commit changes one thing. Make more than one commit if the summary needs the
word "and". Make more than one commit if the change touches unrelated subsystems.
Small commits are better than one large commit.

Put long explanations in `CONTEXT.md` or in a file in `docs/`. Do not put them in
the git log.

## History

A one-time rewrite in July 2026 removed the body from each commit in the history.
Do not add bodies again.

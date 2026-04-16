# LLM Wiki — Usage Notes

Open terminal in project root, run `claude`. It reads `CLAUDE.md` automatically.

## Commands

**Ingest a source:** `Ingest Raw/Rust/whatever-file.md`

**Ask a question:** just ask — it searches the index and reads relevant pages. Say "file it" if the answer is worth keeping as a wiki page.

**Health check:** `Lint the wiki`

**Batch ingest:** `Ingest everything in Raw/Rust/` — useful but less precise. Prefer one-at-a-time for important sources.

## Tips

Commit after each ingest session so you can roll back. High-connectivity sources (ones that mention many tools/concepts) make the best early ingests — they seed the graph with stubs that future sources link into. Keep Obsidian open alongside Claude Code to watch the graph grow in real time. If a page drifts in a direction you don't like, tell Claude Code to revise it — or edit it yourself in Obsidian and it'll respect your changes on the next pass. Update `CLAUDE.md` whenever you want to change conventions — the schema evolves with the wiki.
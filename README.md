# penyero

A small POSIX broker that mediates AI agents' access to production, with human approval and output
review.

The agent asks, you approve in your own pane, and it gets the result without ever holding the
credentials:

```
┌─ vim ───────────────────────────────────────┬─ ai-agent ─────────────────────────────┐
│ src/links.rs                                │ ● Archiving idle links. How many       │
│ 17  /// Links unopened for this many days.  │   would that touch in prod?            │
│ 18  const IDLE_DAYS: i32 = 90;              │                                        │
│ 19                                          │   $ id=$(penyero request \             │
│ 20  pub fn archive_idle(db: &mut Client)    │       prod-db-ro \                     │
│ 21      -> Result<u64, Error> {             │       -r "links idle 90+ days" \       │
│ 22      db.execute(ARCHIVE, &[&IDLE_DAYS])█ │       < idle.sql)                      │
│ 23  }                                       │   $ penyero wait "$id" &&              │
│ ~                                           │       penyero retrieve "$id"           │
│                                             │   └ waiting for approval...            │
├─ penyero serve trusted shortener ───────────┴────────────────────────────────────────┤
│ [3mxp9dq2vt] trusted/shortener · prod-db-ro                                          │
│     reason: links idle 90+ days                                                      │
│     SELECT count(*) FROM links                                                       │
│      WHERE last_opened < now() - interval '90 days'                                  │
│ run? [y]es [Y]es+release [n]o(+msg) [s]kip [e]dit [v]iew [r]unning [?] █             │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

**Status: definition stage.** There is no code yet. The design is being worked out in these
documents:

- [SPEC.md](SPEC.md): what penyero is and does, and its security model.
- [DESIGN.md](DESIGN.md): how it is built.
- [SETUP.md](SETUP.md): a reference deployment around it, on
  [cayo](https://github.com/nandilugio/cayo), isolated workspaces for AI agents.

Licensed under the [MIT License](LICENSE).

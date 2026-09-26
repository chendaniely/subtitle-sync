# subtitle-sync

A command-line tool that spot-checks sidecar subtitle files against a video's audio and fixes
whole-file timing offsets. The design is in `docs/superpowers/specs/`.

## This repository is public

The tool runs on one person's private media library, but nothing about that library belongs
in this repository.

**Never commit or publish**, whether in files, commit messages, or GitHub issue or PR text:

- media file names, show or movie titles from the library, its folder layout, or counts of
  its contents
- absolute paths to the library or its network share, hostnames, IP addresses
- anything extracted from the library: subtitle text, audio, transcripts, `results.jsonl`,
  `report.html`, calibration measurements
- the owner's name or personal notes

**Where those things go instead.** All of these are git-ignored:

- `config.local.toml`: library paths and machine-specific settings
- `docs/private/`: library-specific notes, test cases, and `deny-list.txt`
- `.cache/` and the configured runs folder: every output

**Test fixtures are synthetic.** Only made-up subtitle lines and generated audio go in
`tests/fixtures/`, never copyrighted text or clips. Tests that need real media read it from
the library at run time and write nothing into the repository.

**Examples are generic.** Write "a title containing dots, such as `a.k.a.`", not a real
episode name, and "a German-language film", not its title.

**Before every commit and push**, check the staged changes and the commit message against
`docs/private/deny-list.txt` (its header explains how each entry is matched), and read the
diff once for anything the list can't catch.

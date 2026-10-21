# Beckhoff Package Hosting

The TwinCAT Packaging Guide, a Zensical site: `zensical.toml` plus the Markdown pages under `docs\`. The
README has the layout and the build commands. The writing rules for the pages: plain English, short
sentences, every claim grounded in a real package or a test that was run, hostnames and feed names as
placeholders.

## Git is read-only for Claude in this repository

Claude may read the repository (`git status`, `git log`, `git diff`, `git show`, `gh pr view` and the like)
and may edit files under `docs\` and the site configuration. Claude never runs a command that changes git
state or anything on GitHub: no `git add`, `commit`, `push`, `pull`, `merge`, `rebase`, `reset`, `restore`,
`checkout`, `switch`, `stash`, `tag`, branch creation or deletion, remote changes, and no `gh` command that
creates, edits, merges or closes anything. `.claude\settings.json` denies those commands; do not work around
the denial through another shell, another tool, a script, or by asking a subagent.

When the work reaches a point where one of those commands is the next step, stop and hand it to John:

1. Say in one line what the command will do.
2. Put the exact command in a fenced `bash` code block, one command per block, run from the repository
   root, with no leading prompt character.
3. Wait. John runs it himself and reports back. Do not re-check whether it ran unless asked.

This applies to the first commit of the repository as much as to every later one. It also applies to the
app's own commit, push and pull request buttons: do not offer them in a wrap-up for this repository; offer
the commands in the message instead.

Everything that only reads or only builds is fine without asking: `python -m zensical build --clean --strict`,
`python -m zensical serve`, reading and searching files.

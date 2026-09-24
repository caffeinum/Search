# This fork

`caffeinum/Search` is a fork of [driceroland/Search](https://github.com/driceroland/Search). It carries a few changes on top of upstream `main`: the headless proposal in `docs/headless.md`, and bench tabs that WebKit paints as seen (`Bench.paintUnseen`).

Remotes: `origin` is this fork, `upstream` is driceroland (fetch only; its push URL is disabled on purpose).

## Keeping it in sync

```bash
git fetch upstream && git rebase upstream/main   # or `git merge upstream/main` once our commits are shared
git push origin main                             # add --force-with-lease after a rebase of pushed commits
```

## Building without Xcode

With only the Command Line Tools, the macOS 27 SDK's `@State` needs a SwiftUI macro plugin they don't ship. Build against the 26.5 SDK instead:

```bash
SDKROOT=/Library/Developer/CommandLineTools/SDKs/MacOSX26.5.sdk ./build.sh release
```

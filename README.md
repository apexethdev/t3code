# T3 Code: stable + terminal dock

This is a personal fork of [T3 Code](https://github.com/pingdotgg/t3code). It tracks the
**stable** releases and adds one feature: a setting to dock the terminal at the bottom of the
right panel, instead of under the chat.

For anything else, including what T3 Code is, how to install it and how to use it, see the
[upstream repo](https://github.com/pingdotgg/t3code) and [t3.codes](https://t3.codes).

## Why this fork exists

The dock was proposed upstream in
[pingdotgg/t3code#19868](https://github.com/pingdotgg/t3code/pull/19868), which was closed
without merging. This fork keeps the patch alive on top of each stable release, so it can be used
as a daily desktop app.

## Branches

- **`stable-dock`** is the one that matters. It's the latest stable tag (`vX.Y.Z`) plus the
  dock patch: 4 small commits, mostly in `apps/web`. Its
  [`STABLE-DOCK.md`](https://github.com/apexethdev/t3code/blob/stable-dock/STABLE-DOCK.md) has
  the full rebase, build and install steps.
- **`main`** is an untouched mirror of upstream `main` (the nightly track), except for this
  README. Don't build from it.

## Keeping up with new releases

This fork does not update itself, and a self-built app has auto-update turned off. To pick up
new upstream features:

1. **Watch for new stable releases.** Stable tags are `vX.Y.Z`, roughly every 1–2 weeks; ignore
   `-nightly` and `-preview` tags. To check:
   `gh release list -R pingdotgg/t3code --exclude-pre-releases -L 1`
2. **Rebase `stable-dock` onto the new tag.** Expect conflicts in busy files like `ChatView.tsx`
   and `Sidebar.tsx`. Then run the dock tests listed in `STABLE-DOCK.md`.
3. **Rebuild and reinstall the macOS app**, following `STABLE-DOCK.md`. The build needs the
   public cloud config copied from the official app, otherwise sign-in and T3 Connect won't
   work. Each build is unsigned, so expect a one-time macOS keychain prompt after every rebuild.

If you don't want the upkeep, use the official app instead: you lose the dock but get automatic
updates.

## License

MIT, same as upstream. See [LICENSE](LICENSE).

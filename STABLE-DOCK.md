# stable-dock: personal T3 Code build

This branch is the latest **stable** T3 Code release tag plus a small patch that adds a
"Dock terminal in right panel" setting. Upstream closed the PR (pingdotgg/t3code#19868), so the
owner builds and runs it as their macOS desktop app in place of the official stable app.

Read this before rebasing, rebuilding, or reinstalling. Do not open PRs upstream from this branch.

## Branch layout

- `stable-dock` = `<latest stable tag>` + the dock patch + this file. Never base it on `main`;
  `main` is the nightly track.
- The dock patch: `feat(web): option to dock the terminal under the right panel` plus its
  follow-up `fix(web)`/`feat(web)` commits, including a "Terminal" header for the docked drawer and
  the setting defaulting to on. They touch mostly `apps/web`, plus one setting in
  `packages/contracts`.
- Remotes: `origin` is the owner's public fork. Upstream is `https://github.com/pingdotgg/t3code`.
  Add it as `upstream` if it's missing.

## When a new stable release lands

Check with `gh release list -R pingdotgg/t3code --exclude-pre-releases -L 3`. Stable tags look
like `vX.Y.Z`; ignore `-nightly` and `-preview` tags.

### 1. Rebase (any machine)

```sh
git fetch upstream --tags
git rebase --onto vNEW vOLD stable-dock
```

Conflicts are common in `apps/web/src/components/ChatView.tsx`, `Sidebar.tsx`,
`settings/SettingsPanels.tsx` and `apps/web/src/lib/revealThreadTerminal.ts`, which upstream
changes constantly. Keep upstream's behaviour and re-apply the dock logic on top.

Then run the patch's tests:

```sh
vp test run apps/web/src/lib/terminalDock.test.ts \
  apps/web/src/hooks/useThreadTerminalVisible.test.ts \
  apps/web/src/lib/revealThreadTerminal.test.ts \
  packages/contracts/src/settings.test.ts
```

Pushing the rebased branch to the fork is a force push. Ask the owner first.

### 2. Build the DMG (macOS only)

Requirements:

- The Node version from `engines` in `package.json`. Use nvm: `nvm install 24`. Don't change the
  nvm default, which stays on Node 22.
- pnpm through corepack, matching `packageManager`.
- Current stable Rust from rustup, for `native/resource-monitor`.
- Xcode command line tools.

**Cloud config is required.** Release CI compiles four _public_ values into the app: the Clerk
publishable key, the Clerk JWT template, the Clerk CLI OAuth client ID and the relay URL. Without
them, the build has no sign-in button and T3 Connect fails with "Your cloud sign-in changed". They
are public repo variables shipped inside every official app. Take them from the currently
installed **official** app, or from a backup copy of it. Never commit them, and never put them in
a tracked `.env`.

```sh
OFFICIAL="/path/to/official/T3 Code (Beta).app"   # the official app, not this build
X=$(mktemp -d) && npx -y @electron/asar extract "$OFFICIAL/Contents/Resources/app.asar" "$X"
CLI=$(grep -rl 'buildTimeClerkPublishableKey' "$X/apps/server/dist" | head -1)
export T3CODE_CLERK_PUBLISHABLE_KEY=$(grep -o 'buildTimeClerkPublishableKey = readBuildTimeValue("[^"]*")' "$CLI" | cut -d'"' -f2)
export T3CODE_CLERK_CLI_OAUTH_CLIENT_ID=$(grep -o 'buildTimeClerkCliOAuthClientId = readBuildTimeValue("[^"]*")' "$CLI" | cut -d'"' -f2)
export T3CODE_RELAY_URL=$(grep -o 'buildTimeRelayUrl = normalizeSecureRelayUrl("[^"]*")' "$CLI" | cut -d'"' -f2)
export T3CODE_CLERK_JWT_TEMPLATE=$(grep -rho 'VITE_CLERK_JWT_TEMPLATE:`[^`]*`' "$X/apps/server/dist/client/assets" | head -1 | cut -d'`' -f2)
rm -rf "$X"
# Every value must be non-empty; print lengths only, not values
for v in T3CODE_CLERK_PUBLISHABLE_KEY T3CODE_CLERK_CLI_OAUTH_CLIENT_ID T3CODE_RELAY_URL T3CODE_CLERK_JWT_TEMPLATE; do
  eval "echo $v: \${#$v}"; done
```

If any value comes out empty, the bundle layout changed. Inspect the extracted bundle by hand
before building.

Build:

```sh
unset T3CODE_DESKTOP_UPDATE_REPOSITORY GITHUB_REPOSITORY T3CODE_DESKTOP_SIGNED
nvm exec 24 corepack pnpm install
nvm exec 24 corepack pnpm run dist:desktop:dmg:arm64 --build-version X.Y.Z   # match the stable tag
```

The output is `release/T3-Code-X.Y.Z-arm64.dmg`. It's unsigned and has no `app-update.yml`, so it
never auto-updates back to the official app. That's intended.

Check the DMG before installing it. Mount it with `hdiutil attach -nobrowse -readonly`, then
`grep -a -c` its `app.asar` for each of:

- `terminalDock`: the dock patch is in.
- The relay host: the cloud config is in.

Also confirm there is no `app-update.yml` in `Contents/Resources`.

### 3. Install (needs the owner)

1. Ask the owner to quit T3 Code. Quitting ends any agent sessions running in it.
2. If you don't have a backup of the official app for this version yet, `ditto` it into
   `.backups/` in this checkout, e.g. `.backups/t3code-official-X.Y.Z.app`. Keep data
   tarballs there too. `.backups/` is ignored locally through `.git/info/exclude`; add that line
   on a fresh clone.
3. Replace `/Applications/T3 Code (Beta).app` with the app from the DMG. Then run
   `xattr -dr com.apple.quarantine` on it and launch it.

The build keeps the official bundle ID and data folder, so threads and settings carry over.

On first launch, macOS asks for the "t3code Safe Storage" keychain item. That's expected for an
unsigned build; the owner should choose Always Allow. The prompt returns after every rebuild.

## Known limitations

- Passkey sign-in may not work, because it needs T3's signing entitlements. Email or GitHub
  sign-in works.
- There are no update notifications. Watch for new stable tags with the command at the top.

## Rollback

Quit the app and copy the backed-up official `.app` back into `/Applications`. The data folder is
shared, so there's nothing else to restore.

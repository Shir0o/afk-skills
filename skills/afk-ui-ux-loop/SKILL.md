---
name: afk-ui-ux-loop
description: "Autonomous UI/UX & visual E2E audit loop for single repos or multi-repo rotations: supports Flutter and Web projects, captures multi-viewport & interactive click-through screenshots, performs multimodal visual critique, ships one surgical UI/UX improvement via PR, watches CI, squash-merges, and advances Release Please workflows when present."
argument: "[optional: --repo <path|name>, --root <dir>, --owner <user|org|any>, --exclude <csv>, --limit <N> (default: 5), --all, --cutoff <HH:MM>, --concurrency <N> (default: 2), --state-file <path>, --code-cmd <cmd>, --emulator, --admin-merge]"
---

# AFK UI/UX & Visual E2E Audit Loop

Autonomous UI/UX visual inspection, interactive click-through testing, surgical improvement, and PR/release workflow for Flutter and Web projects.

Standard unit tests can pass even when a screen is broken in practice—clipped text, unreadable dark-mode contrast, undersized touch targets, or unresponsive navigation. `afk-ui-ux-loop` combines **programmatic accessibility assertions**, **interactive E2E click-through captures**, and **multimodal visual inspection of rendered screenshots** to ship one surgical improvement per audited repository.

---

## Configuration & Caller Overrides

**Any options passed via CLI arguments or a calling automation prompt override these defaults.**

- **Target Scope (`--repo`, `--root`)**:
  - Defaults to **single-repo mode** against the current git repository when invoked inside a project without `--root`.
  - Pass `--repo <path|name>` to audit a specific repository, or `--root <dir>` (e.g. `~/projects`, `~/code`, or `~`) for multi-repo discovery (depth 1 under `<dir>`).
- **Owner Filter (`--owner`)**:
  - Optional in multi-repo mode. When scanning a broad root such as `$HOME` without `--owner`, auto-detect the authenticated GitHub login via `gh api user --jq .login` so third-party clones are skipped. Pass `--owner any` to disable remote owner filtering.
- **Exclusions (`--exclude`)**:
  - Always skip dotfiles/hidden directories, OS/cache directories (`Library`, `Pictures`, `Music`, `Movies`, `node_modules`), non-UI/tool repos, and any repo whose `AGENTS.md` or `PROJECT.md` opts out of screenshots or UI audits. Merge any caller-supplied `--exclude` entries.
- **Rotation & State (`--limit`, `--all`, `--state-file`)**:
  - In multi-repo mode, persist `{"last_audited_at": {"<repo-name>": "<ISO-8601 UTC timestamp>"}}` in `--state-file` (default: `${XDG_CONFIG_HOME:-~/.config}/afk-ui-ux-loop/state.json`), selecting the `N` oldest repositories (default: `5`, or all with `--all`).
- **Cutoff & Concurrency (`--cutoff`, `--concurrency`)**:
  - Optional local-time stop cutoff (`--cutoff HH:MM`); do not spawn new repository workers past it. Cap concurrent subagents at `--concurrency` (default: `2`) to conserve host CPU and memory.
- **Optional Code Delegation (`--code-cmd`)**:
  - Works standalone using the agent's own file-editing tools. When `--code-cmd` (or a calling prompt) supplies an external coding CLI (e.g. `opencode run -m <model>`), offload code reading, test harness authoring, and widget/CSS edits to it while the multimodal subagent handles screenshot capture, visual inspection, and git/PR/CI orchestration.
- **Design Guidance Skills**:
  - Consult installed UI/UX design skills if present (e.g. `better-interface`, `better-accessibility`, `better-ui`, `better-typography`, `better-colors`, `impeccable`, or any skill paths provided by the caller); otherwise evaluate against WCAG 2.2 AA and standard layout, hierarchy, and responsiveness criteria.
- **Merge & Release (`--admin-merge`)**:
  - Squash-merge with `gh pr merge <pr> --squash --delete-branch` (adding `--admin` when `--admin-merge` is set or when the caller / branch protection requires admin bypass and the user holds admin rights). Advance Release Please / release deployment workflows only when `.github/workflows/release-please.yml` exists or when explicitly requested by the caller.

---

## Operating Principles (Hard Rules)

1. **Visual + Interactive Click-Through First.**
   - Every audit MUST render and visually inspect PNG screenshots via the file-reading/vision tool (`read_file`) across viewports and interactive states—never rely solely on code-level checks:
     - **Flutter**: Light (`390x844`), Dark (`390x844`), Small Phone + Large Font (`320x568` @ `1.6x` text scale), plus interactive click-through frames (`step2_tab.png` / modal / action results).
     - **Web**: Mobile (`375x667`) and Desktop (`1280x800`), plus click-through navigation/theme/modal states.

2. **Zero-Polling Reactive Wakeup.**
   - Whenever the coordinator or a worker launches a background subagent or long-running command (`flutter test`, external `--code-cmd`, `gh pr checks --watch`, `gh run watch`), immediately end the turn and wait for the completion wakeup notification. Never poll task status in a loop.

3. **Bounded Concurrency & Optional Time Cutoff.**
   - Run at most `--concurrency` (default `2`) repository workers concurrently.
   - If `--cutoff HH:MM` is configured, check local time before dispatching each new repository and stop spawning once past the cutoff.

4. **Optional Hybrid Architecture.**
   - In multi-repo runs (or when delegating per-repo work), spawn lightweight multimodal subagents (`invoke_subagent`) to capture screenshots, visually critique PNGs, and drive git/PR/CI steps.
   - When `--code-cmd` is configured, invoke it from the repository root for heavy code reading, `test/ui_ux_audit_test.dart` authoring, and surgical widget/CSS edits; otherwise apply edits directly with native tools.

5. **Strict Ephemeral Resource & Emulator Teardown.**
   - Delete generated golden/screenshot PNGs (`rm -rf test/goldens artifacts/screenshots`) after visual inspection, and ensure `/test/goldens/` is listed in `.gitignore`.
   - If an Android Emulator or iOS Simulator was booted or created during the run, shut it down (`adb emu kill`, `avdmanager delete avd -n afk_temp_avd`, `xcrun simctl shutdown all`, `xcrun simctl erase all`).
   - Stop any background web preview server started during Web checks.

6. **One Surgical Improvement -> CI -> Merge -> Optional Release.**
   - Ship **one** high-impact UI/UX improvement per run on a branch (`fix/ui-ux-<slug>` or `feat/ui-ux-<slug>`) with a conventional commit (`fix(ui): ...` or `feat(ui): ...`).
   - Run lightweight local static checks only (offload full project test suites to CI), watch CI (`gh pr checks <pr> --watch`, up to 3 auto-fix attempts), squash-merge, and advance Release Please / tag-triggered release workflows when configured.

---

## Phase 0 — Target Resolution, Discovery & Rotation (Orchestrator)

1. **Resolve target repositories**:
   - **Single-repo mode** (default inside a git repo without `--root`, or when `--repo <path|name>` is passed): target that repository directly.
   - **Multi-repo mode** (when `--root <dir>` is passed, or when invoked outside a git repo): scan `<dir>` (depth 1):
     ```bash
     ROOT_DIR="${ROOT_DIR:-$HOME}"
     OWNER="${OWNER:-$(gh api user --jq .login 2>/dev/null || true)}"
     for dir in "$ROOT_DIR"/*/; do
       [ -d "$dir/.git" ] || continue
       dir="${dir%/}"
       repo_name="${dir##*/}"
       case "$repo_name" in
         .*|Library|Pictures|Music|Movies|node_modules) continue ;;
       esac
       # Also skip any caller-supplied --exclude names and non-UI / opt-out repos
       remote=$(git -C "$dir" remote get-url origin 2>/dev/null || true)
       if [ -n "$OWNER" ] && [ "$OWNER" != "any" ]; then
         case "$remote" in
           *"$OWNER/"*|*"$OWNER:"*) ;;
           *) continue ;;
         esac
       fi
       if [ -f "$dir/pubspec.yaml" ] || [ -f "$dir/package.json" ]; then
         echo "$repo_name|$dir|$remote"
       fi
     done
     ```
   - Classify each discovered repo as **Flutter (`pubspec.yaml`)** or **Web / Node / Static (`package.json`)**.

2. **Load state & select batch (multi-repo mode)**:
   - Read `--state-file` (default `${XDG_CONFIG_HOME:-~/.config}/afk-ui-ux-loop/state.json`; create parent dirs/file if missing).
   - Sort candidate UI repositories by oldest `last_audited_at` timestamp (never-audited repositories first).
   - Select the oldest `--limit` repositories (default `5`, or all if `--all` is passed).

3. **Dispatch workers**:
   - Check `--cutoff` (if set) before dispatching each batch.
   - Launch up to `--concurrency` (default `2`) `invoke_subagent` workers concurrently (`TypeName: "self"`, lightweight/fast multimodal model when available, e.g. `Model: "flash"`).
   - Enforce the **Zero-Polling Rule**: end the turn after launching subagents or background commands and await wakeup notifications.

---

## Phase 1 — Interactive Visual & Usability Audit (Per-Repo Worker)

### A. Flutter Projects (`pubspec.yaml`)

1. **Sync Default Branch**:
   ```bash
   DEFAULT_BRANCH=$(git rev-parse --abbrev-ref origin/HEAD 2>/dev/null || echo "origin/main")
   DEFAULT_BRANCH="${DEFAULT_BRANCH#origin/}"
   git checkout "$DEFAULT_BRANCH" && git pull --ff-only origin "$DEFAULT_BRANCH"
   ```
   - If release-pipeline scaffolding was explicitly requested by the caller and `.github/workflows/release-please.yml` / release workflows are missing, prepare them according to the repository's conventions.

2. **Create or Update Visual & Interactive Harness (`test/ui_ux_audit_test.dart`)**:
   - Pump viewport/theme/font-scale variants AND interactive click-through journeys (delegate authoring to `--code-cmd` if configured, or write directly):
   ```dart
   import 'package:flutter/material.dart';
   import 'package:flutter_test/flutter_test.dart';

   Future<void> verifyScreenUsabilityAndVisuals(
     WidgetTester tester,
     Widget appOrScreen,
     String screenName,
   ) async {
     final SemanticsHandle handle = tester.ensureSemantics();

     final variants = {
       'phone_light': (size: const Size(390, 844), scale: 1.0, dark: false),
       'phone_dark': (size: const Size(390, 844), scale: 1.0, dark: true),
       'small_large_font': (size: const Size(320, 568), scale: 1.6, dark: false),
     };

     for (final entry in variants.entries) {
       await tester.binding.setSurfaceSize(entry.value.size);
       await tester.pumpWidget(
         MediaQuery(
           data: MediaQueryData(
             size: entry.value.size,
             textScaler: TextScaler.linear(entry.value.scale),
             platformBrightness: entry.value.dark ? Brightness.dark : Brightness.light,
           ),
           child: appOrScreen,
         ),
       );
       await tester.pumpAndSettle();

       // 1. Programmatic usability & WCAG accessibility assertions
       await expectLater(tester, meetsGuideline(androidTapTargetGuideline));
       await expectLater(tester, meetsGuideline(textContrastGuideline));

       // 2. Save rendered screenshot ONLY when --update-goldens is passed
       // so Linux CI never fails on host font antialiasing diffs
       if (autoUpdateGoldenFiles) {
         await expectLater(
           find.byType(MaterialApp).first,
           matchesGoldenFile('goldens/${screenName}_${entry.key}.png'),
         );
       }
     }

     // 3. Interactive E2E Click-Through: tap primary navigation tabs / actions
     await tester.binding.setSurfaceSize(const Size(390, 844));
     final navFinder = find.byType(NavigationDestination).evaluate().isNotEmpty
         ? find.byType(NavigationDestination)
         : find.byType(BottomNavigationBarItem);
     if (autoUpdateGoldenFiles && navFinder.evaluate().length > 1) {
       await tester.tap(navFinder.at(1));
       await tester.pumpAndSettle();
       await expectLater(
         find.byType(MaterialApp).first,
         matchesGoldenFile('goldens/${screenName}_step2_tab.png'),
       );
     }

     handle.dispose();
   }
   ```

3. **Capture & Visually Inspect Screenshots**:
   - Run targeted capture: `flutter test --update-goldens test/ui_ux_audit_test.dart`.
   - Optional Native Emulator/Simulator E2E Check (when `--emulator` is requested or platform views/WebViews/shaders require full device rendering):
     - Check available AVDs (`${ANDROID_HOME:-${ANDROID_SDK_ROOT:-~/Library/Android/sdk}}/emulator/emulator -list-avds`) or iOS Simulators (`xcrun simctl list devices available`).
     - If needed, create/boot a headless AVD (`-no-window -no-audio -no-boot-anim`), run `flutter drive` / `integration_test` + `adb exec-out screencap -p > artifacts/screenshots/emu_screen.png`, and **always kill & delete temporary AVDs/simulators in cleanup**.
   - Inspect every generated PNG (`phone_light.png`, `phone_dark.png`, `small_large_font.png`, `step2_tab.png`) with `read_file`.
   - Consult installed UI/UX skills if present (e.g. `better-interface`, `better-accessibility`, `better-ui`, `better-typography`, `better-colors`, `impeccable`, or caller-specified skill paths); otherwise critique directly against WCAG 2.2 AA and usability standards:
     - Clipped text or `RenderFlex` overflows at `320px` / `1.6x` text scale
     - `< 48x48dp` touch targets or missing semantic tooltips/labels
     - Low-contrast text in Light or Dark mode (`< 4.5:1`)
     - Cramped padding, misaligned edges, unclear hierarchy, or unresponsive interactive states

### B. Web / Node / Static Sites (`package.json`)

1. Start the local dev/preview server in the background on an unused port (e.g. `npm run dev -- --port 5180`).
2. Navigate and interact using `chrome-devtools-mcp` (`new_page`, `navigate_page`, `resize_page`, `take_screenshot`, `take_snapshot`, `click`, `close_page`) — or project-local Playwright if already installed in `node_modules` — at mobile (`375x667`) and desktop (`1280x800`). Avoid running uninstalled global `npx playwright`, which blocks on interactive install prompts.
3. Click through primary navigation links, theme toggles, and interactive controls, capturing screenshots at each step.
4. Inspect the screenshots with `read_file` and check the accessibility tree snapshot (`take_snapshot`).
5. Stop the background preview server and delete temporary screenshot files after inspection.

---

## Phase 2 — Surgical Fix & Local Verification

1. Implement **ONE clear, high-impact UI/UX improvement** in the project's native design idiom (via `--code-cmd` if configured, or directly).
2. Re-run visual capture (`flutter test --update-goldens test/ui_ux_audit_test.dart` or `chrome-devtools-mcp` screenshots) and inspect updated PNGs via `read_file` across Light, Dark, Small + `1.6x` Font Scale, and interactive click-through states.
3. **Clean up ephemeral artifacts**:
   - Delete generated screenshots (`rm -rf test/goldens artifacts/screenshots`) and ensure `/test/goldens/` is in `.gitignore`.
   - Shut down any emulator, simulator, or dev server started during the run.
4. Run fast static gates (offload full project test suites to CI):
   - **Flutter**: `flutter test test/ui_ux_audit_test.dart && flutter analyze && dart format --output=none --set-exit-if-changed .`
   - **Web / Node**: `npm run lint` / `npm run build` (matching the repo's scripts).

---

## Phase 3 — PR, CI Watch, Merge & Optional Release Pipeline

1. **Create Branch & Open PR**:
   - Checkout `fix/ui-ux-<short-slug>` or `feat/ui-ux-<short-slug>`.
   - Commit with a conventional commit message (`fix(ui): ...` or `feat(ui): ...`).
   - Push and open a PR via `gh pr create` (following `skill://pr` if installed, or a concise Summary / Visual Evidence / Verification body).

2. **Watch CI & Squash-Merge PR**:
   - Run `gh pr checks <pr> --watch` (auto-fixing up to 3 attempts if any check fails).
   - Squash-merge once green:
     ```bash
     # Add --admin when --admin-merge is set or requested by the caller
     gh pr merge <pr> --squash --delete-branch
     ```

3. **Optional Release Please & Release Workflow Verification**:
   - Only if `.github/workflows/release-please.yml` exists (or release automation was requested by the caller):
     - Watch the `Release Please` workflow on the default branch (`gh run list --workflow=release-please.yml --limit 1` -> `gh run watch <id>`).
     - Find the open Release Please PR (`gh pr list --search "chore(main): release"`).
     - Watch its CI checks (`gh pr checks <release-pr> --watch`).
     - Squash-merge it using the same merge flags (`--squash --delete-branch`, plus `--admin` if configured).
     - Verify that any tag-triggered release workflow (e.g. `release.yml` / `release-android.yml` for mobile/store tracks) triggers cleanly on the new `v*` tag.

4. **Sync Local Checkout & Update State**:
   - `git checkout "$DEFAULT_BRANCH" && git pull --ff-only origin "$DEFAULT_BRANCH"`
   - When running in multi-repo mode or when `--state-file` is provided, update `last_audited_at["<repo_name>"]` with the current ISO-8601 UTC timestamp in the state file.

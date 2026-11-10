---
name: afk-ui-ux-loop
description: "Autonomous, budget-aware UI/UX & visual E2E click-through audit loop across ~/ repositories: dynamically discovers repos, selects a rotating subset to stay within a $20/mo budget, runs interactive visual click-through & multi-viewport screenshot inspections (headless matrix, Playwright/DevTools MCP, or on-demand Android/iOS emulator with guaranteed teardown), ships one surgical UI/UX improvement via PR, watches CI, admin-merges both the feature PR and Release Please PR, and publishes to the Google Play internal testing track."
argument: "[optional: --repo <name>, --limit <N> (default: 2), --all, --emulator]"
---

# AFK UI/UX & Visual E2E Audit Loop

Autonomous, budget-conscious UI/UX visual inspection, interactive click-through testing, surgical improvement, and Google Play release pipeline for coding projects under `~/`.

Standard unit tests pass even when a screen is completely unusable—text clipped off-screen, dark mode contrast unreadable, touch targets overlapping, or interactive flows broken. `afk-ui-ux-loop` eliminates manual user testing by combining **programmatic accessibility assertions**, **interactive E2E click-through navigation**, and **multimodal LLM vision critiques of real rendered screenshots**—shipping one surgical improvement per selected repository all the way to the **Google Play Console internal testing track**.

---

## Operating Principles (Hard Rules)

1. **Daily Full-Fleet Execution Powered by `opencode-go/deepseek-v4.1-flash` (OpenCode Go Subscription).**
   - Because heavy code reading, test harness generation, widget/CSS patching, and git/PR operations are side-loaded to your OpenCode Go subscription model **`opencode-go/deepseek-v4.1-flash`** (`opencode run -m opencode-go/deepseek-v4.1-flash`), **every discovered UI repository under `~/` is processed on every run** without burning Antigravity quota.
   - The Antigravity `flash` (3.8 Flash low-effort) subagent acts as the **orchestrator and multimedia/vision specialist**: it triggers screenshot/golden captures, uses `read_file` to visually inspect the rendered PNG screenshots, and delegates the code/test authoring to `opencode run -m opencode-go/deepseek-v4.1-flash "<surgical instructions>"`.
   - Maintain `~/.gemini/config/afk-ui-ux-state.json` to track `last_audited_at` per repository so interrupted runs resume cleanly and all repositories under `~/` are covered.

2. **Dynamic Future-Proof Repository Discovery (`~/`).**
   - Never rely solely on a static hardcoded list. Dynamically discover all git repositories under `~/` (maxdepth 2, excluding `Library`, `Pictures`, `Music`, `Movies`, `.Trash`, `node_modules`, `.gemini`, `.claude`, `.agents`, `.github-repo`, `afk-skills`, `helpful-scripts`, and non-UI backend-only scripts).
   - Cross-reference with `~/.github-repo/apply-standards.sh` (`REPOS` array). If a newly created UI repository under `~/` is not yet registered in `~/.github-repo/apply-standards.sh`, onboard it or include it automatically in the rotation.
   - Classify each UI repository by stack:
     - **Flutter (`pubspec.yaml`)**
     - **Web / Node / Capacitor / Eleventy (`package.json` / `.eleventy.js`)**
     - **Godot (`project.godot`)**

3. **Hybrid Architecture: Antigravity Flash (Orchestrator + Multimodal Vision) + OpenCode Go `opencode-go/deepseek-v4.1-flash` (Code & Test Lifting).**
   - **Antigravity Subagent (`Model: "flash"`)**: Acts strictly as the orchestrator and multimedia/vision engine:
     1. Runs `flutter test --update-goldens` (or Playwright / Android Emulator / iOS Simulator screenshot capture).
     2. Uses `read_file` on the generated PNGs (`phone_light.png`, `phone_dark.png`, `small_large_font.png`, `step2_tab.png`) to perform multimodal visual and usability critique.
     3. Runs `gh pr checks --watch`, `gh pr merge --squash --admin --delete-branch`, and verifies the Release Please + Google Play `internal` track release.
   - **OpenCode Go (`opencode-go/deepseek-v4.1-flash`)**: Handles all code reading, test harness authoring (`test/ui_ux_audit_test.dart`), and surgical widget/CSS code edits:
     - Check available OpenCode Go DeepSeek Flash model if needed (`opencode models | grep 'opencode-go/deepseek.*flash'`), defaulting to **`opencode-go/deepseek-v4.1-flash`**.
     - Invoke from the repository root:
       ```bash
       opencode run -m opencode-go/deepseek-v4.1-flash "<detailed surgical instructions>"
       ```
   - Run at most **1–2 subagents concurrently** to conserve host CPU and memory (`AGENTS.md` §6).

4. **Visual + Interactive Click-Through First (Never Programmatic-Only).**
   - Every audit MUST produce and visually inspect (`read_file` on PNGs) rendered screenshots across states and interactive flows—not just pass code-level assertions:
     1. **Multi-Viewport & State Matrix Screenshots**: Light Mode (`390x844`), Dark Mode (`390x844`), and Small Phone + Large Font (`320x568` @ `1.6x` text scale).
     2. **Interactive Click-Through Journey (E2E Flow)**: Programmatically tap through primary navigation tabs, open dialogs/modals/forms, toggle states, and capture `step_1_home.png`, `step_2_tab_or_modal.png`, `step_3_action_result.png` to verify buttons actually respond and secondary screens look great.
     3. **On-Demand Emulator / Simulator / Browser (with Mandatory Teardown)**:
        - For **Flutter**: Use headless `flutter test --update-goldens test/ui_ux_audit_test.dart` for multi-screen interactive `tester.tap(...)` + `pumpAndSettle()` screenshot sequences AND/OR boot an on-demand Android AVD (`~/Library/Android/sdk/emulator/emulator -avd <name> -no-window -no-audio -no-boot-anim`) or iOS Simulator (`xcrun simctl`) when native platform views/WebViews/shaders require full device rendering.
        - For **Web**: Boot the local dev/preview server in the background, navigate and click through flows using `chrome-devtools-mcp` (`new_page`, `resize_page`, `click`, `take_screenshot`, `take_snapshot`) or `npx playwright`, capturing mobile (`375x667`) and desktop (`1280x800`) screenshots.

5. **Strict Ephemeral Resource & Emulator Teardown.**
   - **Always clean up** before finishing a repository:
     - Delete generated golden/screenshot PNGs (`rm -rf test/goldens artifacts/screenshots`) after visually inspecting them via `read_file`, and ensure `/test/goldens/` is in `.gitignore`.
     - If an Android Emulator or iOS Simulator was booted or created during the run, shut it down (`adb emu kill`, `avdmanager delete avd -n afk_temp_avd`, `xcrun simctl shutdown all`, `xcrun simctl erase all`) and delete any downloaded temporary system images if disk space is constrained.
     - Kill any background dev servers started during Web E2E checks.

6. **End-to-End Release to Google Play Internal Track.**
   - Every improvement lands on a branch (`fix/ui-ux-<slug>` or `feat/ui-ux-<slug>`) with a conventional commit (`fix(ui): ...` or `feat(ui): ...`).
   - Watch CI (`gh pr checks <pr> --watch`), auto-fixing up to 3 attempts if needed, then squash-merge with admin bypass (`gh pr merge <pr> --squash --admin --delete-branch`).
   - For Android/Flutter apps: ensure `.github/workflows/release-please.yml` and `.github/workflows/release.yml` (Fastlane `supply` to Google Play `internal` track) exist (modeled after `~/tempo/.github/workflows/`). Wait for the `Release Please` PR, watch its CI checks, and squash-merge it with `--squash --admin --delete-branch` so the `v*` tag triggers the Google Play `internal` testing track deployment.

---

## Phase 0 — Dynamic Repo Discovery & Budget Rotation (Orchestrator)

1. **Discover all candidate UI repositories under `~/`**:
   ```bash
   for dir in ~/*/; do
     [ -d "$dir/.git" ] || continue
     repo_name=$(basename "$dir")
     # Skip non-app / infra / tool repos
     case "$repo_name" in
       afk-skills|helpful-scripts|claude-code-playground|.*) continue ;;
     esac
     remote=$(git -C "$dir" remote get-url origin 2>/dev/null || true)
     [[ "$remote" == *"Shir0o/"* ]] || continue
     if [ -f "$dir/pubspec.yaml" ] || [ -f "$dir/package.json" ] || [ -f "$dir/project.godot" ]; then
       echo "$repo_name|$dir|$remote"
     fi
   done
   ```

2. **Load `~/.gemini/config/afk-ui-ux-state.json`** (create if missing):
   - Sort discovered UI repositories by oldest `last_audited_at` timestamp (repositories not yet audited today come first).
   - Process **every discovered UI repository under `~/`** on each run (unless `--repo <name>` or `--limit <N>` is explicitly passed).

3. **Dispatch Subagents (`Model: "flash"` Orchestrator/Multimedia + `opencode-go/deepseek-v4.1-flash` Code Engine)**:
   - Launch `invoke_subagent` (`TypeName: "self"`, `Model: "flash"`) in batches of at most 2 concurrent subagents across all discovered repositories.
   - Each subagent offloads code reading, `test/ui_ux_audit_test.dart` scaffolding, and surgical widget/CSS edits to **`opencode run -m opencode-go/deepseek-v4.1-flash`** while using its native multimodal `read_file` tool to inspect rendered PNG screenshots.

---

## Phase 1 — Interactive Visual & Usability Audit (Subagent per Repo)

### A. Flutter Projects (`pubspec.yaml`)

1. **Sync & Verify Release Pipeline**:
   - `git checkout main && git pull --ff-only origin main`.
   - Check `.github/workflows/release-please.yml` and `.github/workflows/release.yml`. If missing on an Android-capable Flutter app, copy and adapt the standard Fastlane `internal` track workflow from `~/tempo/.github/workflows/`.

2. **Write Interactive Multi-Screen & Multi-Viewport Harness (`test/ui_ux_audit_test.dart`)**:
   - Render the app and capture both **viewport variants** AND **interactive click-through steps**:
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
       // so Ubuntu CI never fails on macOS vs Linux font antialiasing diffs
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
     // Tap secondary tabs or primary interactive controls and capture screenshots
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

3. **Run Targeted Visual Capture & Inspect with Multimodal Vision**:
   - Run: `flutter test --update-goldens test/ui_ux_audit_test.dart`
   - Optional Native Emulator E2E Check (when `--emulator` is requested or the app uses platform plugins/WebViews that cannot render in headless widget tests):
     - Check available AVDs: `~/Library/Android/sdk/emulator/emulator -list-avds` or iOS Simulators (`xcrun simctl list devices available`).
     - If needed, create a temporary AVD via `avdmanager`, boot with `-no-window -no-audio`, run `flutter drive` / `integration_test` + `adb exec-out screencap -p > /tmp/emu_screen.png`, and **always kill & delete the AVD in a `finally` cleanup step**.
   - Call `read_file` on every generated PNG (`phone_light.png`, `phone_dark.png`, `small_large_font.png`, `step2_tab.png`).
   - Consult installed UI/UX skills (`/Users/twang/.gemini/config/skills/better-interface/SKILL.md`, `better-accessibility`, `better-ui`, `better-typography`, `better-colors`, `impeccable`) to critique:
     - Clipped text or `RenderFlex` overflows at `320px` / `1.6x` font scale
     - `< 48x48dp` touch targets or missing semantic tooltips/labels
     - Unreadable low-contrast text in Light or Dark mode (`< 4.5:1`)
     - Cramped padding, misaligned card edges, broken visual hierarchy, or unresponsive interactive states

### B. Web / Node / Eleventy Projects (`package.json`)

1. Start the local dev/preview server in the background.
2. Use `chrome-devtools-mcp` (`new_page`, `navigate_page`, `resize_page`, `take_screenshot`, `take_snapshot`, `click`) or `npx playwright` at mobile (`375x667`) and desktop (`1280x800`).
3. Click through primary navigation links, theme toggles, and interactive controls, capturing screenshots at each step.
4. Inspect the screenshots with `read_file` and check the accessibility tree snapshot (`take_snapshot`).
5. Stop the background server and delete temporary screenshot files after inspection.

---

## Phase 2 — Surgical Fix & Local Verification

1. Implement **ONE clear, high-impact UI/UX + usability improvement** in the project's native design idiom.
2. Re-run `flutter test --update-goldens test/ui_ux_audit_test.dart` and visually inspect the updated PNGs via `read_file` to confirm the fix looks great across Light, Dark, Small+1.6x Font Scale, and interactive click-through states.
3. **Clean up ephemeral artifacts**:
   - Delete `test/goldens/` (`rm -rf test/goldens`) and ensure `/test/goldens/` is in `.gitignore`.
   - Shut down any emulator/simulator or dev server started during the run.
4. Run fast static gates:
   - Flutter: `flutter test test/ui_ux_audit_test.dart && flutter analyze && dart format --output=none --set-exit-if-changed .`
   - Web/Node: `npm run lint` / `npm run build`

---

## Phase 3 — PR, CI Watch, Admin Merge & Google Play Internal Track Release

1. **Create Branch & Open PR**:
   - Checkout `fix/ui-ux-<short-slug>` or `feat/ui-ux-<short-slug>`.
   - Commit with conventional commit message (`fix(ui): ...` or `feat(ui): ...`).
   - Push and create PR via `gh pr create` formatted per `skill://pr`.
2. **Watch CI & Squash-Merge Feature/Fix PR**:
   - `gh pr checks <pr> --watch` (auto-fix up to 3 attempts if any check fails).
   - `gh pr merge <pr> --squash --admin --delete-branch`.
3. **Merge Release Please PR & Trigger Google Play Internal Release**:
   - If `.github/workflows/release-please.yml` exists:
     - Watch the `Release Please` workflow run on `main` (`gh run list --workflow=release-please.yml --limit 1` -> `gh run watch <id>`).
     - Find the open Release Please PR (`gh pr list --search "chore(main): release"`).
     - Watch its CI checks (`gh pr checks <release-pr> --watch`).
     - Squash-merge with admin bypass: `gh pr merge <release-pr> --squash --admin --delete-branch`.
     - Verify that `release.yml` / `release-android.yml` triggers on the new `v*` tag to build and upload the signed AAB to the Google Play Console `internal` track.
4. **Sync Local Checkout & Update Rotation State**:
   - `git checkout main && git pull --ff-only origin main`.
   - Update `last_audited_at` for this repository in `~/.gemini/config/afk-ui-ux-state.json`.

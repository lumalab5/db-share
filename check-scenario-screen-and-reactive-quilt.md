# Desktop chip scrolling + marquee conversation title

## Context

Two defects on Windows desktop, both rooted in the app never having been adapted from its mobile-first assumptions:

1. **Scenario screen chips are unreachable by mouse.** The category and CEFR rows at
   [scenarios_screen.dart:95-146](../../../mnt/hgfs/Share/vlearn/vLearn2/flutter_app/lib/features/scenarios/scenarios_screen.dart#L95-L146)
   are plain horizontal `ListView`s. The app sets no `scrollBehavior` on `MaterialApp.router`
   ([main.dart:186](../../../mnt/hgfs/Share/vlearn/vLearn2/flutter_app/lib/main.dart#L186)), so Flutter's default
   `MaterialScrollBehavior` omits `PointerDeviceKind.mouse` from `dragDevices` → click-drag does nothing.
   A plain vertical mouse wheel doesn't help either: `ScrollableState` reads `scrollDelta.dx` for a
   horizontal axis, and a wheel only produces `dy`. Result: overflowing categories are unreachable
   except by shift+wheel. The same latent bug sits in the home and news strips.

2. **Conversation title bar can't handle a long title.** It currently shows only `'Turn N'` /
   `'Past conversation'` ([conversation_screen.dart:266-273](../../../mnt/hgfs/Share/vlearn/vLearn2/flutter_app/lib/features/conversation/conversation_screen.dart#L266-L273)),
   which is uninformative — the session already carries `scenarioTitle`, unused here. Once the real
   scenario title goes in, it will overflow: localized titles run 30-50+ chars while ~200px of the
   bar is taken by the timer chip, tutor toggle and End button.

**Outcome:** chips scroll by mouse drag *and* wheel on desktop; the conversation bar shows the
scenario title and marquee-scrolls it when it doesn't fit.

**Scope confirmed with the user:** also fix the home/news strips. Tutor mode's hand-rolled header
([tutor_mode_view.dart:375-407](../../../mnt/hgfs/Share/vlearn/vLearn2/flutter_app/lib/features/conversation/widgets/tutor_mode_view.dart#L375-L407))
is **out of scope** — it keeps its centered `Turn N`.

---

## Part 1 — Mouse-friendly horizontal scrolling

### 1a. New `lib/core/theme/app_scroll_behavior.dart`

```dart
/// Adds mouse + trackpad to the drag devices Flutter enables by default, so
/// horizontal strips can be click-dragged on Windows/desktop.
class AppScrollBehavior extends MaterialScrollBehavior {
  const AppScrollBehavior();
  @override
  Set<PointerDeviceKind> get dragDevices => const {
        PointerDeviceKind.touch,
        PointerDeviceKind.mouse,
        PointerDeviceKind.trackpad,
        PointerDeviceKind.stylus,
        PointerDeviceKind.invertedStylus,
      };
}
```

Wire it as `scrollBehavior: const AppScrollBehavior(),` on the `MaterialApp.router` at
[main.dart:186-208](../../../mnt/hgfs/Share/vlearn/vLearn2/flutter_app/lib/main.dart#L186-L208). Leave the minimal
error-screen `MaterialApp` at [main.dart:132](../../../mnt/hgfs/Share/vlearn/vLearn2/flutter_app/lib/main.dart#L132) alone —
it has no scrollables.

Watch-out to verify by hand: this also lets the mouse drag *vertical* lists, which can contend with
drag-to-select in the two `SelectableText` widgets ([license_screen.dart:405](../../../mnt/hgfs/Share/vlearn/vLearn2/flutter_app/lib/features/license/license_screen.dart#L405),
[settings_screen.dart:740](../../../mnt/hgfs/Share/vlearn/vLearn2/flutter_app/lib/features/settings/settings_screen.dart#L740)).
Expected to be fine (axis-based arena resolution), but check selection still works on those screens.

### 1b. New `lib/shared/widgets/horizontal_wheel_scroll.dart`

Wheel support is the half `dragDevices` cannot give. Widget owns a `ScrollController` and hands it
to a builder:

```dart
class HorizontalWheelScroll extends StatefulWidget {
  const HorizontalWheelScroll({super.key, required this.builder});
  final Widget Function(BuildContext context, ScrollController controller) builder;
}
```

State: create/dispose `_controller`; wrap `widget.builder(context, _controller)` in a `Listener`
with `onPointerSignal`. **Critical detail** — do not scroll directly from the `Listener`, register
with the pointer-signal resolver, or the strip *and* the page behind it will both scroll:

```dart
void _onPointerSignal(PointerSignalEvent event) {
  if (event is! PointerScrollEvent || !_controller.hasClients) return;
  final pos = _controller.position;
  final target = (pos.pixels + event.scrollDelta.dy)
      .clamp(pos.minScrollExtent, pos.maxScrollExtent);
  if (target == pos.pixels) return; // already at the end — let the page scroll
  GestureBinding.instance.pointerSignalResolver
      .register(event, (_) => _controller.jumpTo(target));
}
```

The inner horizontal `Scrollable` sees the event first, computes `dx == 0`, and declines to
register; our ancestor `Listener` then registers and wins over any outer vertical scrollable. Not
registering when already clamped is deliberate: wheeling past the strip's end falls through to the
page, which is what desktop users expect.

### 1c. Apply at the five strip sites

Pattern (same at every site — wrap, take the controller, pass it to the list):

```dart
SizedBox(
  height: 44,
  child: HorizontalWheelScroll(
    builder: (context, controller) => ListView(
      controller: controller,
      padding: const EdgeInsets.symmetric(horizontal: 16),
      scrollDirection: Axis.horizontal,
      children: [ /* unchanged */ ],
    ),
  ),
)
```

Sites:
- [scenarios_screen.dart:95](../../../mnt/hgfs/Share/vlearn/vLearn2/flutter_app/lib/features/scenarios/scenarios_screen.dart#L95) — categories
- [scenarios_screen.dart:123](../../../mnt/hgfs/Share/vlearn/vLearn2/flutter_app/lib/features/scenarios/scenarios_screen.dart#L123) — CEFR levels
- [home_screen.dart:307](../../../mnt/hgfs/Share/vlearn/vLearn2/flutter_app/lib/features/home/home_screen.dart#L307) — `_ScenarioStrip`
- [home_screen.dart:390](../../../mnt/hgfs/Share/vlearn/vLearn2/flutter_app/lib/features/home/home_screen.dart#L390) — `_ScenarioStripSkeleton`
- [news_strip.dart:27](../../../mnt/hgfs/Share/vlearn/vLearn2/flutter_app/lib/features/news/widgets/news_strip.dart#L27)

No visual change on mobile; `_Chip` at
[scenarios_screen.dart:220](../../../mnt/hgfs/Share/vlearn/vLearn2/flutter_app/lib/features/scenarios/scenarios_screen.dart#L220)
is untouched.

---

## Part 2 — Scenario title with marquee

### 2a. New `lib/shared/widgets/marquee_text.dart`

A self-contained widget (no new pubspec dependency — the project has none for this):

```dart
class MarqueeText extends StatefulWidget {
  const MarqueeText(this.text, {super.key, this.style,
    this.velocity = 30,                                  // logical px / second
    this.pause = const Duration(milliseconds: 1200)});
}
```

Behavior:
- `LayoutBuilder` for the available width; measure the string with a `TextPainter` using the
  resolved style.
- **Fits, or animations disabled** (`MediaQuery.disableAnimationsOf(context)`, for reduce-motion):
  render a plain `Text(maxLines: 1, overflow: TextOverflow.ellipsis)` and start no ticker.
- **Overflows:** `ClipRect` → `SingleChildScrollView(scrollDirection: horizontal,
  physics: NeverScrollableScrollPhysics(), controller: _c)` → the `Text`, driven by an async loop:
  pause → `animateTo(maxScrollExtent)` at `distance / velocity` seconds, `Curves.linear` → pause →
  `animateTo(0, Curves.easeOut)` → repeat. Guard every await with `mounted && _c.hasClients`, and
  restart the loop from `didUpdateWidget` when `text` or `style` changes.

Lives in `lib/shared/widgets/` alongside the other cross-screen primitives (`refreshing_dot.dart`,
`score_ring.dart`).

### 2b. Use it in the conversation AppBar

At [conversation_screen.dart:265-273](../../../mnt/hgfs/Share/vlearn/vLearn2/flutter_app/lib/features/conversation/conversation_screen.dart#L265-L273),
replace the title with the scenario title, falling back the same way the history screen already does
at [conversation_history_screen.dart:151](../../../mnt/hgfs/Share/vlearn/vLearn2/flutter_app/lib/features/conversation/conversation_history_screen.dart#L151):

```dart
title: data.maybeWhen(
  data: (d) => MarqueeText(d.session.scenarioTitle ?? 'Free conversation'),
  orElse: () => const Text('Conversation'),
),
```

`ConversationSession.scenarioTitle` already exists and is parsed
([models.dart:254, 271, 282](../../../mnt/hgfs/Share/vlearn/vLearn2/flutter_app/lib/core/models/models.dart#L254)) — no
model or API change needed.

**One judgment call to flag:** the title is the only place `Turn N` and the read-only
"Past conversation" marker are currently shown, and comments at
[conversation_screen.dart:259-262](../../../mnt/hgfs/Share/vlearn/vLearn2/flutter_app/lib/features/conversation/conversation_screen.dart#L259-L262)
say the read-only layout exists for exactly that status banner. To avoid losing both, prepend to
`actions` a compact `EditorialMono` 13px label (matching the countdown chip's style) reading
`Turn N` when active or `Past` when read-only — shown only when the bar is at least ~400px wide, so
a narrow phone gives the title all the room. Easy to delete if you'd rather have the title alone.

---

## Verification

1. `flutter analyze` — must stay clean.
2. `flutter test` — the existing suites under `test/{features,datapack,guard,storage}` should be
   unaffected; add none unless a strip test already exists.
3. **Windows run** via the repo's `run_debug_windows.bat`, in a *narrow* window so the chips overflow:
   - Scenarios: click-drag the category row with the mouse → scrolls. Hover it and turn the wheel →
     scrolls horizontally. Wheel past the last chip → nothing jumps.
   - Home + News: same two gestures on the card strips; wheeling the strip must **not** also scroll
     the page underneath (that's the resolver check).
   - Settings / License: drag-select text in the `SelectableText` fields still works.
4. Start a scenario conversation with a long title (any 30+ char seed title in
   `assets/seed/scenarios.json`, or switch locale to a language with longer strings): the AppBar
   title should pause, slide left, pause, ease back. Resize the window wider until it fits — the
   animation must stop and the text sit static.
5. Open a finished session from history → title is the scenario title, `Past` marker visible.
6. Android emulator smoke test: chip rows and strips still flick-scroll by touch exactly as before.

## Commit

Per project rules: one commit for the whole task plus a story file under
`story_claude/2026_08_27/<hhmmss>_Desktop_chip_scrolling_and_marquee_title.md`.

## Step 0 — copy this plan into the repo

Before implementing, save this document to
`docs/application/04_desktop_chip_scroll_and_marquee_title.md`, next to the existing app-level
design docs (`01_auth.md`, `02_network.md`, `03_protocol.md`), so it is version-controlled with the
code. Adjust the file's own relative links to the repo root as part of the copy.


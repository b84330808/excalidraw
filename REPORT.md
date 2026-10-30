# excalidraw: overlap box selection picks rotated shapes it doesn't touch

## Branch

- Fork branch: `b84330808/excalidraw:claude/optimistic-turing-5a8hkn`
- Commit: `309c7f015540ec7a78a885cf11b542b80d89b25a` (author `b84330808 <b84330808@gmail.com>`, trailer `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`)
- Based on upstream `excalidraw/excalidraw` `master` at `b99a49d` (latest at the time of writing, re-checked after committing)
- Files: `packages/element/src/bounds.ts` (+1 −6), `packages/excalidraw/tests/selection.test.tsx` (+25)

## Policy check

- `CONTRIBUTING.md` points to `dev-docs/docs/introduction/contributing.mdx`. It has no AI or agent policy and no attestation requirement. It asks for a semantic PR title and recommends tests for bug fixes. It also recommends manually testing the change on the per-PR staging deploy; that's a recommendation, not an attestation, but worth doing once the PR is open.
- `AGENTS.md`, `CLAUDE.md`, `.github/copilot-instructions.md` are coding guidelines only.
- There is no PR template in the repo.
- `.github/workflows/semantic-pr-title.yml` requires a scope. Allowed scopes are `app`, `editor`, `packages/excalidraw`, `packages/utils`, `docker` and `repo`. The title below uses `editor`.
- Our open PR is excalidraw/excalidraw#12368 (`fix(editor): ignore clipped frame children when zooming to fit`). This change doesn't overlap it: it is box selection, not zoom-to-fit, and it touches neither frames nor `zoomToFit*`.

## PR title

```
fix(editor): don't select rotated shapes the selection box misses in overlap mode
```

## PR body

```markdown
With the box selection mode set to "overlap", dragging a selection box near a rotated shape can select it even though the box doesn't touch it.

`elementsOverlappingBBox` first checks whether the midpoints of the shape's sides are inside the box. It already rotates those midpoints by the element's angle when building them, and then rotates them again inside the `.some()`, so it actually tests points rotated by twice the angle. For a 200×20 rectangle rotated by 45°, the right side's midpoint is tested as if rotated by 90°, about 25px off the outline, and a box around that spot selects the rectangle. This removes the second rotation. Boxes that cross the outline without containing a midpoint are still caught by the edge intersection check that follows. `elementsOverlappingBBox` is also exported from `@excalidraw/excalidraw` and `@excalidraw/utils`, so API callers got the same false positives.

The existing rotated rectangle test uses a square, whose midpoints rotated by 90° land on each other, which is why it didn't catch this. The new test box-selects a rotated 200×20 rectangle off its outline (selected on master, not with this change) and around its rotated midpoint (selected either way).

I used Claude Code to write this change.
```

## The bug

`packages/element/src/bounds.ts`, `elementsOverlappingBBox`, step 3 (the "overlap" preliminary check for non-linear elements). The four side midpoints are built with `pointRotateRads(..., center, element.angle)`, then the `.some()` callback did `pointInsideBounds(pointRotateRads(point, center, element.angle), selectionBounds)`, rotating them a second time. The fix is `.some((point) => pointInsideBounds(point, selectionBounds))`.

This only produces false positives. A box that really touches the outline is still found by the `intersectElementWithLineSegment` fallback. The user-facing trigger is Main menu → box selection mode "overlap", with a rotated non-square shape (rectangle, diamond, ellipse, image, text, embeddable…) and a box dragged inside its axis-aligned bounds but off the shape.

The code dates back to the original overlap box selection (it was moved from `selection.ts` into `bounds.ts` by excalidraw/excalidraw#11275 unchanged).

Direct check of `elementsOverlappingBBox` on a 200×20 rectangle at (100,190) rotated 45° (selected count):

| box | master | this branch |
| --- | --- | --- |
| `[190, 260, 210, 310]` (off the outline) | 1 | 0 |
| `[265, 265, 275, 275]` (around the rotated right midpoint) | 1 | 1 |

## Test

`packages/excalidraw/tests/selection.test.tsx`, in the existing `describe("box-selection overlap mode")` block, next to the rotated-rectangle and rotated-ellipse cases:

```ts
it("should not select a rotated rectangle when the selection box misses its outline", () => {
  // 200x20, centered at 200,200, lying along the diagonal
  const rect = API.createElement({
    type: "rectangle",
    x: 100,
    y: 190,
    width: 200,
    height: 20,
    angle: Math.PI / 4,
  });

  API.setElements([rect]);

  // ~25px off the outline, around where the right side midpoint would be
  // if rotated by 90° instead of 45°
  boxSelect(190, 260, 210, 310);

  assertSelectedElements([]);

  // around the right side midpoint
  boxSelect(265, 265, 275, 275);

  assertSelectedElements([rect.id]);
});
```

### Fail before (fix reverted, test kept)

```
$ vitest run packages/excalidraw/tests/selection.test.tsx -t 'should not select a rotated rectangle when the selection box misses its outline'
 RUN  v3.0.6 /home/user/excalidraw
 ❯ packages/excalidraw/tests/selection.test.tsx (69 tests | 1 failed | 68 skipped) 516ms
   × box-selection overlap mode > should not select a rotated rectangle when the selection box misses its outline 513ms
     → expected 1 to be +0 // Object.is equality
⎯⎯⎯⎯⎯⎯⎯ Failed Tests 1 ⎯⎯⎯⎯⎯⎯⎯
 FAIL  packages/excalidraw/tests/selection.test.tsx > box-selection overlap mode > should not select a rotated rectangle when the selection box misses its outline
AssertionError: expected 1 to be +0 // Object.is equality
- Expected
+ Received
- 0
+ 1
 ❯ assertSelectedElements packages/excalidraw/tests/test-utils.ts:229:37
    227|     .flat()
    228|     .map((item) => (typeof item === "string" ? item : item.id));
    229|   expect(selectedElementIds.length).toBe(ids.length);
       |                                     ^
    230|   expect(selectedElementIds).toEqual(expect.arrayContaining(ids));
    231| };
 ❯ packages/excalidraw/tests/selection.test.tsx:763:5
⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[1/1]⎯
 Test Files  1 failed (1)
      Tests  1 failed | 68 skipped (69)
error Command failed with exit code 1.
```

(Line 763 is the first `assertSelectedElements([])`, after the off-outline box.)

### Pass after

```
$ vitest run packages/excalidraw/tests/selection.test.tsx -t 'should not select a rotated rectangle when the selection box misses its outline'
 RUN  v3.0.6 /home/user/excalidraw
 ✓ packages/excalidraw/tests/selection.test.tsx (69 tests | 68 skipped) 377ms
   ✓ box-selection overlap mode > should not select a rotated rectangle when the selection box misses its outline 375ms
 Test Files  1 passed (1)
      Tests  1 passed | 68 skipped (69)
Done in 10.14s.
```

### Other checks (with the fix)

| Check | Result |
| --- | --- |
| Related suites: `packages/element`, `packages/math`, `packages/utils`, `selection.test.tsx`, `lasso.test.tsx`, `move.test.tsx` | 46 files passed, 1014 tests passed (37 skipped, 1 todo) |
| `yarn test:update` (full suite) | 149 files passed, 2633 tests passed (47 skipped, 1 todo); no snapshot changes |
| `yarn test:typecheck` | pass |
| `yarn test:code` (eslint, `--max-warnings=0`) | pass |
| `yarn test:other` (prettier) | pass |

## Duplicate search (excalidraw/excalidraw, all states)

All searches were run without a state filter, so they cover open, closed and merged items.

| Search | Type | Results |
| --- | --- | --- |
| `elementsOverlappingBBox` | PRs | 1: excalidraw/excalidraw#11470 (closed, `NonDeleted` type narrowing), unrelated |
| `box selection overlap rotated` | PRs | excalidraw/excalidraw#9771, excalidraw/excalidraw#9488, excalidraw/excalidraw#6256, excalidraw/excalidraw#3512, excalidraw/excalidraw#1899, all unrelated |
| `rotated element box select` | PRs | 19 results (excalidraw/excalidraw#10947, excalidraw/excalidraw#9771, excalidraw/excalidraw#9488, excalidraw/excalidraw#8884, excalidraw/excalidraw#8613, excalidraw/excalidraw#7855…), none about this |
| `box selection mode` | PRs | excalidraw/excalidraw#12074 (open, separate lasso selection mode), excalidraw/excalidraw#12041, excalidraw/excalidraw#12012, excalidraw/excalidraw#11661, excalidraw/excalidraw#11605, excalidraw/excalidraw#11267… none touch the overlap pre-check |
| `overlap selection` | PRs | excalidraw/excalidraw#12353, excalidraw/excalidraw#12237, excalidraw/excalidraw#12168, excalidraw/excalidraw#12146, excalidraw/excalidraw#12108, excalidraw/excalidraw#12074, excalidraw/excalidraw#11992, excalidraw/excalidraw#11991, excalidraw/excalidraw#11862 (closed, lasso respects `boxSelectionMode`), excalidraw/excalidraw#11861, excalidraw/excalidraw#11858 (open, lasso selection mode), excalidraw/excalidraw#11250, excalidraw/excalidraw#11053 (closed, introduced overlap box selection), excalidraw/excalidraw#8021… none fix this |
| `"double rotation" OR "rotated twice" OR "rotates twice"` | PRs | excalidraw/excalidraw#5723 only, unrelated |
| `"double rotation" OR "rotated twice" OR "rotates twice"` | Issues | excalidraw/excalidraw#12034 (laser-pointer `runLength`), excalidraw/excalidraw#12073, excalidraw/excalidraw#11909, excalidraw/excalidraw#5166…, unrelated |
| `box selection overlap mode selects rotated element not touched` | Issues | 40 results (excalidraw/excalidraw#11842, excalidraw/excalidraw#8713, excalidraw/excalidraw#3889, excalidraw/excalidraw#6862, excalidraw/excalidraw#9517…), none about this |
| `overlap box selection wrong element selected` | Issues | 64 results (excalidraw/excalidraw#11856 lasso test, excalidraw/excalidraw#11842, excalidraw/excalidraw#11709 flipped-element bounds, excalidraw/excalidraw#10882…), none about this |

One caveat: this session could search upstream but not read PR diffs. The lasso PRs (excalidraw/excalidraw#11858, excalidraw/excalidraw#11861, excalidraw/excalidraw#12074) are about lasso mode per their titles and bodies. Before opening the PR, the local session could confirm with `gh pr diff` that none of them edits the `.some(...)` in `elementsOverlappingBBox`.

## Other bugs found and dropped as duplicates

| Bug | Already covered by |
| --- | --- |
| YouTube `embed/videoseries?list=` and `watch?app=desktop&v=` links lose the playlist/video id (`RE_YOUTUBE` alternation order) | excalidraw/excalidraw#12226 (open) |
| Unlisted Vimeo links `vimeo.com/<id>/<hash>` | excalidraw/excalidraw#12198 (open) |
| `restoreElements` with `deleteInvisibleElements` crashes (whole file "invalid") on a linear or freedraw element without `points`: `isInvisiblySmallElement` runs on the raw element | issue excalidraw/excalidraw#11925; PRs excalidraw/excalidraw#11926, excalidraw/excalidraw#11992, excalidraw/excalidraw#12000 (open) |
| Character-width cache keyed by `charCodeAt(0)`, so astral characters share their high surrogate's slot | excalidraw/excalidraw#12371 (open) |
| Uppercase `.PNG` file handle saved as JSON | excalidraw/excalidraw#12289 (open) |
| laser-pointer `runLength` double-counts the last segment | issue excalidraw/excalidraw#12034 (open) |

## Unclaimed leads for later (no PR or issue found, not worked on)

- `wrapText` treats the no-break spaces U+00A0, U+2007 and U+202F as break opportunities (`COMMON.WHITESPACE = /\s/u` in `packages/element/src/textWrapping.ts`). The editor's textarea doesn't break there, so text re-wraps differently after editing ends. A fix needs `WHITESPACE`, the `/\s/.test(token)` in `wrapLine` and `satisfiesWordInvariant` to share one breaking-whitespace class. `wrapText("a b c", font, 40)` gives `"a b\nc"`, expected `"a\nb c"`.
- `segmentsIntersectAt` (`packages/math/src/segment.ts`) excludes both ends of the second segment (`u === 0` → null and `u < 1`), so the result depends on argument order. This is frame-related through `isElementIntersectingFrame`, so it's close to our frame PR's area.
- `ellipseDistanceFromPoint` returns NaN for zero-height or zero-width ellipses. This is only reachable through the API or imported data.
- `validateLibraryUrl` (`packages/excalidraw/data/library.ts`) puts allowlisted hostnames into a regex without escaping `.` and never checks the protocol, so `raw-githubusercontent.com` passes. It's security-adjacent: report it privately rather than in a public PR.
- Chart paste reads `1,5` as 15 (commas are always thousands separators), including semicolon-separated data from decimal-comma locales. It's a heuristic change, not a clear bug.

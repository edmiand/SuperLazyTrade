# ship — bump version, commit, and push

Bump the major version number in `SuperLazyTrade.pine`, then commit and push.

## Steps

1. Read line 4 of `SuperLazyTrade.pine` to get the current `VERSION` string (e.g. `"V49B"`).

2. Extract the numeric part and increment it by 1 (e.g. `49` → `50`). The new version is just the number with no letter suffix (e.g. `"V50"`).

3. Replace the VERSION line:
   - Old: `string VERSION = "V49B"` (whatever the current value is)
   - New: `string VERSION = "V50"` (new incremented value, no letter)

4. Also update the `indicator(...)` call on line 5 if it hard-codes a version string — but usually it references `VERSION` so no change needed there.

5. Run `git diff SuperLazyTrade.pine` to confirm only the VERSION line changed.

6. Stage, commit, and push:
   ```
   git add SuperLazyTrade.pine
   git commit -m "Bump version to V<N>"
   git push
   ```

7. Report the old version → new version to the user.

## Notes

- The user may optionally provide a commit message suffix as an argument (e.g. `/ship add session filter`). If provided, use: `"Bump version to V<N>: <suffix>"`.
- Never change the `@version=6` Pine Script directive on line 1 — that is the Pine Script language version, not the indicator version.
- If the diff shows unrelated changes beyond the VERSION line, stage only `SuperLazyTrade.pine` and proceed — those changes are intentional.

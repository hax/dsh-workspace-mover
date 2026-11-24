# Fork notes

This is a local fork of [PianoPrince/dsh-workspace-mover](https://github.com/PianoPrince/dsh-workspace-mover)
(for 2.4.4, HEAD `99644fe`), carried so it can run on **DeepSeek Harness 0.2.2-rc.3**
with the defects found during review fixed. It is not published.

## Why this fork exists

The upstream release targets 0.2.1-rc.5. On 0.2.2-rc.3 its opt-in tests against an
installed runtime fail, and a review of the move pipeline found three data-integrity
gaps plus one usability trap. Each fix below has a regression test that fails without it.

## Changes

| Commit | Change | Test that pins it |
| --- | --- | --- |
| `test:` | Restore `test:installed-host` on 0.2.2: minified client bundle anchor, `provider`/`model` selection, `tools.resolve`, and 0.2.2 fork-lineage shape | `npm run test:installed-host` (18/18) |
| `fix:` P3 | An attach-time rollback no longer loses workspace membership: drop the registry's poisoned header/path cache before re-attaching, detach strays, and report instead of claiming a clean rollback | `attach 失败 → 自动回滚到源目录并重新挂账` |
| `fix:` P10 | Refuse moves while `workspace/session-activity` reports background jobs or delegated subagents; `turn` keeps the existing stop-and-move path, and `schedule` is allowed with a warning (it addresses the session by id, so the timer follows the move — refusing would strand scheduled sessions with no way to pause them) | `后台作业仍在跑 → 拒绝迁移，且零副作用`、`定时任务不算「占用中」…` |
| `fix:` P7 | Irreversible endpoints (`trash.purge`, `backups.deleteOne`, non-dryRun `data.cleanup`) need a one-shot token bound to action + target; the ignored `{ authority: 'loopback' }` argument is gone | `不可逆端点需要一次性确认令牌…` |
| `fix:` P6 | A move of an archived session reports `archived` / `archived-hidden` and the client says it stays hidden until restored | `归档会话移动后仍然是归档态…` |

## Deliberately not changed

- **Session locks / multi-process moves.** The plugin moves the whole session
  directory (so `session.lock` travels and its inode survives) but never probes the
  host's `flock` lease, so a second DSH process writing the same session would race.
  Fixing this needs a supported lease API; until then, do not move sessions while
  another DSH process is running against the same `DSH_HOME`.
- **Encoders.** `projectKey` / `encodeSegment` are still reimplemented (copied) from
  the kernel. They match 0.2.2 byte for byte, including the 251-character truncation;
  a kernel change would drift silently, which is what `mover_doctor`'s layout probe is for.
- **Archive/unarchive.** The delete path still falls back to writing the archive set
  directly when the official method is absent, which skips the host's
  `workspace/session-unarchiving` event.

## Maintenance

```sh
# upstream sync (expect conflicts only in the four patched regions)
git fetch origin && git rebase origin/main

# verification
npm test                                  # 352 pass / 3 skip
DSH_TEST_NODE_MODULES=<dsh>/node_modules npm run test:installed-host   # 18/18
npm run check:syntax
```

Install into a profile with `link:` while developing:

```sh
dsh plugin --profile web add "link:$PWD"
```

`link:` installs do not resolve peer dependencies from the target, which is fine here:
the host half imports only `node:` builtins and local modules, and the client half is
loaded through the host's own module loader.

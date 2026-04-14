---
name: qqmusic-playlist
description: >
  Automate QQ Music playlist management via reverse-engineered internal Web APIs.
  Use when the user wants to list playlists, add/remove songs, search tracks,
  create/delete playlists, manage favorites (red heart), or bulk-import songs
  from text into QQ Music.
---

# QQ Music Playlist Automation

Manage QQ Music playlists through internal Web APIs reverse-engineered from y.qq.com.

All API calls require three auth params: `qm_keyst` (login credential), `uin` (QQ number), `g_tk` (CSRF token derived from qm_keyst). Obtain them via the auth flow below.

## Authentication

Try in priority order, stop on first success:

**Strategy A — Auto-extract (when chrome-devtools-mcp is available)**

Check if MCP tools `list_pages`, `select_page`, `evaluate_script` exist. If so:

1. `list_pages` → find page with URL containing `y.qq.com`
2. If none → `new_page` to open `https://y.qq.com`
3. `select_page` to select it
4. `evaluate_script`:

```javascript
(() => {
  const m = document.cookie.match(/qm_keyst=([^;]+)/);
  if (!m) return { error: "Not logged in: qm_keyst cookie not found. Please log in at y.qq.com first." };
  const qm_keyst = m[1];
  const calcGtk = k => { let h = 5381; for (let i = 0; i < k.length; i++) h += (h << 5) + k.charCodeAt(i); return h & 0x7fffffff; };
  const u = document.cookie.match(/uin=o?(\d+)/);
  if (!u) return { error: "Not logged in: uin cookie not found." };
  return { qm_keyst, uin: u[1], gtk: calcGtk(qm_keyst), status: "ok" };
})()
```

**Strategy B — File / env vars**

Check in order:
- Env vars `QM_KEYST` and `QQ_UIN`
- `.env` file in project root (`QM_KEYST=xxx`, `QQ_UIN=xxx`)

Compute g_tk:
```python
def calc_gtk(key: str) -> int:
    h = 5381
    for c in key:
        h += (h << 5) + ord(c)
    return h & 0x7FFFFFFF
```

**Strategy C — Ask user (last resort)**

Tell the user: open y.qq.com → F12 → Application → Cookies → copy `qm_keyst` and `uin`.

## How to Call APIs

**With browser (Strategy A):** Execute `fetch` via `evaluate_script` in page context.

**Without browser (Strategy B/C):** Use Python `urllib` or `curl` with headers:
```
Cookie: uin=<uin>; qm_keyst=<qm_keyst>; qqmusic_key=<qm_keyst>
Referer: https://y.qq.com/
Content-Type: application/json; charset=utf-8
```

## Operations

| # | Operation | Module / Method | Type |
|---|-----------|----------------|------|
| 1 | List user playlists | GET `c.y.qq.com` | Read |
| 2 | Get playlist songs | GET `c.y.qq.com` | Read |
| 3 | Get playlist `dirid` | GET `c.y.qq.com` (`onlysong=0`) | Read |
| 4 | Add songs to playlist | `PlaylistDetailWrite` / `AddSonglist` | Write |
| 5 | Remove songs from playlist | `PlaylistDetailWrite` / `DelSonglist` | Write |
| 6 | Delete playlist | `PlaylistBaseWrite` / `DelPlaylist` | Write |
| 7 | Create playlist | `PlaylistBaseWrite` / `AddPlaylist` | Write |
| 8 | Search songs | `music.search.SearchCgiService` / `DoSearchForQQMusicDesktop` | Read |
| 9 | SmartBox quick search | GET `c.y.qq.com` | Read |
| 10 | Check if songs are favorited | `SongFavRead` / `IsSongFanByMid` or `IsSongFanById` | Read |
| 11 | Add/remove favorites (red heart) | `PlaylistDetailWrite` / `AddSonglist` or `DelSonglist` with `dirId: 201` | Write |
| 12 | Bulk text import (search + add) | Composite workflow | Write |

For full API specs, params, and code examples for each operation, read [reference.md](reference.md).

## Key Facts

| Topic | Detail |
|-------|--------|
| `tid` vs `dirid` | `tid` = public playlist ID (visible in URLs). `dirid` = internal directory ID required for write ops. Get it via `onlysong=0` endpoint. |
| Favorites playlist | Fixed `dirId: 201`. Adding/removing favorites = adding/removing songs from this playlist. No separate favorites API needed. |
| `songType` | Almost always `13`. No need to determine dynamically. |
| Read vs Write endpoints | Reads: `c.y.qq.com` (legacy). Writes: `POST https://u.y.qq.com/cgi-bin/musicu.fcg` (unified). |
| `claude-in-chrome` blocked | This MCP tool blocks y.qq.com. Must use `chrome-devtools-mcp` instead. |
| `g_tk` refresh | Recalculate when cookie changes. Long-running ops may need refresh. |
| `credentials` | All `fetch` calls must include `credentials: 'include'` to send cookies. |
| Batch search | A single `musicu.fcg` call supports multiple `req_N` keys (up to 10 searches per call). |

## Rate Limits

| Operation | Concurrency | Interval |
|-----------|-------------|----------|
| Read playlist info | 8 parallel | 200ms |
| Write (add/remove songs) | Sequential | 500ms per batch (50 songs/batch) |
| Search | Sequential | 200ms per query |
| Delete playlist | Sequential | 300ms per call |

## Output Format

After each operation, report:
- **Operation**: what was done
- **Summary**: success / existed / failed counts
- **Key data**: `dirId`, song ID range, batch count
- **Errors**: specific error details if any

## Quality Bar

- Read current state before writes (list → add → list to verify).
- On mid-batch errors, log succeeded vs failed batches. Do not retry the entire operation.
- `g_tk` errors (code `-1` or `1000`) → recalculate gtk, retry once.
- Network errors → surface to user, let them decide.

## Boundaries

- Search returns first match; ambiguous song names may mismatch. Confirm with user when in doubt.
- Only operates on the **currently logged-in user's** playlists. Cannot modify others' playlists.
- Unofficial API — may break if Tencent changes backend.

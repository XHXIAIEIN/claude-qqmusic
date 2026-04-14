# API Reference

## POST Endpoint Template (musicu.fcg)

All write operations go through this endpoint:

```
POST https://u.y.qq.com/cgi-bin/musicu.fcg
Content-Type: application/json
```

Request body template:

```json
{
  "comm": {
    "g_tk": <gtk>,
    "uin": "<uin>",
    "format": "json",
    "ct": 20,
    "cv": 4747474,
    "platform": "yqq.json",
    "authst": "<qm_keyst>"
  },
  "req_0": {
    "module": "<module>",
    "method": "<method>",
    "param": { ... }
  }
}
```

Multiple operations in a single call: use `req_0`, `req_1`, `req_2`... keys (up to ~10).

All `fetch` calls must use `credentials: 'include'`.

---

## Operation 1: List User Playlists

**GET** `https://c.y.qq.com/rsc/fcgi-bin/fcg_user_created_diss?hostuin=${uin}&sin=0&size=100&format=json`

Response: `data.data.disslist[]` — each item has `{ tid, diss_name, song_cnt, dirid }`.

- `tid`: public playlist ID (for read ops)
- `diss_name`: playlist name
- `song_cnt`: song count

---

## Operation 2: Get Playlist Songs

**GET** `https://c.y.qq.com/qzone/fcg-bin/fcg_ucc_getcdinfo_byids_cp.fcg?type=1&json=1&utf8=1&onlysong=1&disstid=${tid}&format=json`

Response: `data.songlist[]` — each item has `{ songid, songmid, songname, singer[], albumname }`.

---

## Operation 3: Get Playlist dirid

**GET** — same endpoint as Operation 2 but with `onlysong=0`:

`https://c.y.qq.com/qzone/fcg-bin/fcg_ucc_getcdinfo_byids_cp.fcg?type=1&json=1&utf8=1&onlysong=0&disstid=${tid}&format=json`

Response: `data.cdlist[0].dirid` — the internal directory ID needed for all write operations.

Key: `onlysong=0` returns full playlist metadata including `dirid`. `onlysong=1` returns only songs.

---

## Operation 4: Add Songs to Playlist

**POST** musicu.fcg

```javascript
{
  module: "music.musicasset.PlaylistDetailWrite",
  method: "AddSonglist",
  param: {
    dirId: <dirId>,
    v_songInfo: [{ songType: 13, songId: <id> }, ...]  // max 50 per batch
  }
}
```

Response: `req_0.data.result.songlist[]` — each item has `existed: 0` (new) or `1` (duplicate).

Batch strategy: 50 songs per batch, 500ms delay between batches.

---

## Operation 5: Remove Songs from Playlist

**POST** musicu.fcg

```javascript
{
  module: "music.musicasset.PlaylistDetailWrite",
  method: "DelSonglist",
  param: {
    dirId: <dirId>,
    v_songInfo: [{ songType: 13, songId: <id> }, ...]
  }
}
```

---

## Operation 6: Delete Playlist

**POST** musicu.fcg

```javascript
{
  module: "music.musicasset.PlaylistBaseWrite",
  method: "DelPlaylist",
  param: { dirId: <dirId> }
}
```

Batch delete: sequential, 300ms delay between calls.

---

## Operation 7: Create Playlist

**POST** musicu.fcg

```javascript
{
  module: "music.musicasset.PlaylistBaseWrite",
  method: "AddPlaylist",
  param: { dirName: "<name>", dirDesc: "", dirPicUrl: "", taglist: [] }
}
```

Response: `req_0.data.result` contains new playlist info. `code=1000` means not logged in.

---

## Operation 8: Search Songs

**POST** musicu.fcg

```javascript
{
  module: "music.search.SearchCgiService",
  method: "DoSearchForQQMusicDesktop",
  param: {
    query: "<query>",
    num_per_page: 10,
    page_num: 1,
    search_type: 0  // 0=song, 2=album, 3=playlist, 7=lyrics, 9=MV, 12=artist
  }
}
```

Response: `req_0.data.body.song.list[]` — each item has `{ id, mid, name, singer[{name}], album{name}, type }`.

`id` = songId, `mid` = songMid, `type` = songType.

---

## Operation 9: SmartBox Quick Search

**GET** `https://c.y.qq.com/splcloud/fcgi-bin/smartbox_new.fcg?key=${encodeURIComponent(keyword)}&format=json`

Response:
- `data.data.song.itemlist[]` — song suggestions
- `data.data.singer.itemlist[]` — artist suggestions
- `data.data.album.itemlist[]` — album suggestions

Lightweight, suitable for autocomplete.

---

## Operation 10: Check Favorites (IsSongFan)

**POST** musicu.fcg — by songMid:

```javascript
{
  module: "music.musicasset.SongFavRead",
  method: "IsSongFanByMid",
  param: { v_songMid: ["004Z8Ihr0JIu5s", "001yS0W31QLFns"] }
}
```

Response: `req_0.data.m_fan` → `{ "004Z8Ihr0JIu5s": true, "001yS0W31QLFns": false }`.

By songId:

```javascript
{
  module: "music.musicasset.SongFavRead",
  method: "IsSongFanById",
  param: { v_songId: [102065756, 233013360] }
}
```

Response: `req_0.data.m_fan` → `{ "102065756": true, "233013360": false }`.

---

## Operation 11: Add/Remove Favorites (Red Heart)

The favorites playlist ("我喜欢") has a fixed `dirId: 201`. Use Operation 4 (AddSonglist) and Operation 5 (DelSonglist) with `dirId: 201`.

**Add favorite:**
```javascript
{ module: "music.musicasset.PlaylistDetailWrite", method: "AddSonglist", param: { dirId: 201, v_songInfo: [{ songType: 13, songId: <id> }] } }
```

**Remove favorite:**
```javascript
{ module: "music.musicasset.PlaylistDetailWrite", method: "DelSonglist", param: { dirId: 201, v_songInfo: [{ songType: 13, songId: <id> }] } }
```

---

## Operation 12: Bulk Text Import (Search + Add)

Composite workflow — given a list of song names, search each and add matches to a playlist:

1. Parse input: each line is `"artist - title"` or just `"title"`
2. Search each via Operation 8 (`num_per_page: 1` for best match)
3. Deduplicate by songId
4. Batch add via Operation 4 (50 per batch, 500ms intervals)
5. Report: matched count, not-found list, added count, already-existed count

Optimization: batch searches — pack up to 10 searches into a single musicu.fcg call using `req_0`...`req_9`.

---

## Full Workflow Example

Complete flow in a single `evaluate_script` call:

```javascript
(async () => {
  // 1. Auth
  const m = document.cookie.match(/qm_keyst=([^;]+)/);
  if (!m) return { error: "Not logged in" };
  const qm_keyst = m[1];
  const calcGtk = k => { let h = 5381; for (let i = 0; i < k.length; i++) h += (h << 5) + k.charCodeAt(i); return h & 0x7fffffff; };
  const gtk = calcGtk(qm_keyst);
  const uin = document.cookie.match(/uin=o?(\d+)/)[1];

  // 2. Get dirid
  const tid = 123456789; // replace with target playlist tid
  const info = await (await fetch(
    `https://c.y.qq.com/qzone/fcg-bin/fcg_ucc_getcdinfo_byids_cp.fcg?type=1&json=1&utf8=1&onlysong=0&disstid=${tid}&format=json`,
    { credentials: 'include' }
  )).json();
  const dirId = info.cdlist[0].dirid;

  // 3. Batch add (50 per batch, 500ms interval)
  const songIds = [/* song ID array */];
  const results = [];
  for (let i = 0; i < songIds.length; i += 50) {
    const batch = songIds.slice(i, i + 50);
    const res = await (await fetch("https://u.y.qq.com/cgi-bin/musicu.fcg", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      credentials: "include",
      body: JSON.stringify({
        comm: { g_tk: gtk, uin, format: "json", ct: 20, cv: 4747474, platform: "yqq.json", authst: qm_keyst },
        req_0: {
          module: "music.musicasset.PlaylistDetailWrite",
          method: "AddSonglist",
          param: { dirId, v_songInfo: batch.map(id => ({ songType: 13, songId: id })) }
        }
      })
    })).json();
    results.push({ batch: i / 50 + 1, result: res.req_0 });
    if (i + 50 < songIds.length) await new Promise(r => setTimeout(r, 500));
  }
  return { dirId, batches: results.length, results };
})()
```

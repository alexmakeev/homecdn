# HomeCDN — Technology Research

## Summary: Recommended Minimum Viable Stack

| Layer | Tool | Why |
|-------|------|-----|
| **Video platform** | **Jellyfin** | Only self-hosted platform with apps on ALL targets: web, Android, Android TV, Fire TV, LG webOS, Samsung Tizen (sideload). Auto-indexes directories. GPL v2. 47k stars. |
| **SSO broker** | **Keycloak** + [keycloak-russian-providers](https://github.com/playa-ru/keycloak-russian-providers) | Only solution with ready-made VK + Yandex plugins. ~400MB RAM. |
| **SSO plugin** | [jellyfin-plugin-sso](https://github.com/9p4/jellyfin-plugin-sso) | OIDC/SAML support for Jellyfin. Documented Keycloak integration. |
| **Admin curation UI** | **Custom build** (FastAPI + React/Vue) | No existing tool covers multi-platform search + preview + checkbox select + download queue. |
| **Download engine** | **yt-dlp** | Supports YouTube, RuTube, VK Video, 1800+ sites. Full metadata download. |
| **Auto-download** | **Pinchflat** (optional) | Channel subscriptions for automated download alongside manual curation. |
| **Deployment** | **Docker Compose** | All components containerized. |

### Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                    HomeCDN Server (Intel N100, 8GB RAM)               │
│                                                                      │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │  Jellyfin   │  │  Admin       │  │  Keycloak    │               │
│  │  (video     │  │  Curation UI │  │  (SSO:       │               │
│  │  platform)  │  │  (search,    │  │  VK, Yandex, │               │
│  │             │  │  preview,    │  │  Google)     │               │
│  │  Web + Apps │  │  download    │  │             │               │
│  │  video.home │  │  queue)      │  │  ~400MB RAM  │               │
│  └──────┬──────┘  └──────┬───────┘  └──────────────┘               │
│         │                │                                          │
│         │                ▼                                          │
│         │   ┌──────────────────────┐                                │
│         │   │  yt-dlp (download    │                                │
│         │   │  engine + metadata)  │                                │
│         │   └──────────┬───────────┘                                │
│         │              │                                            │
│         ▼              ▼                                            │
│  ┌─────────────────────────────────┐                                │
│  │  Shared Storage (HDD RAID)      │                                │
│  │  /media/videos/                 │                                │
│  │  Jellyfin auto-indexes (inotify)│                                │
│  └─────────────────────────────────┘                                │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 1. Video Platform Comparison

| Feature | **Jellyfin** | **MediaCMS** | **PeerTube** | **Emby** |
|---------|-------------|-------------|-------------|---------|
| License | GPL v2 | AGPL v3 | AGPL v3 | Proprietary |
| GitHub stars | ~47k | ~4.5k | ~14.1k | closed |
| Web UI | Media library (Netflix-like) | YouTube-like (tags, comments) | YouTube-like (federated) | Netflix-like |
| User accounts | Built-in | Built-in + RBAC | Built-in | Built-in |
| SSO/OIDC | Plugin (jellyfin-plugin-sso) | No (Django AllAuth incomplete) | Plugin (OIDC, OAuth2) | LDAP only (paid) |
| **Android app** | Google Play + F-Droid | No (Flutter prototype) | Google Play + F-Droid | Google Play (paid tier) |
| **Android TV** | Google Play | No | No | Google Play (paid tier) |
| **Fire TV** | Amazon Appstore | No | No | Amazon Appstore |
| **Samsung Tizen** | Sideload (one-click tool) | No | No | Samsung Store |
| **LG webOS** | LG Content Store (official) | No | No | LG Content Store |
| Auto-index dir | Yes (inotify) | No | No (upload only) | Yes (inotify) |
| Upload API | No (file-based) | Yes (REST) | Yes (REST) | No (file-based) |
| Docker | Official | Official | Official | Official |
| HW transcoding | FFmpeg QSV/VA-API, N100 confirmed | FFmpeg | FFmpeg | FFmpeg (paid for HW) |
| Cost | Free | Free | Free | $119 lifetime |

**Winner: Jellyfin** — only platform with full app coverage across all target devices.

### Jellyfin Client Apps Detail

| Platform | App | Status |
|----------|-----|--------|
| Android phone | Jellyfin (official) | Google Play, F-Droid. 1.9M downloads. |
| Android phone | Findroid (3rd party) | Google Play, F-Droid. Native Kotlin, mpv player, offline downloads. |
| Android TV | Jellyfin for Android TV | Google Play. AV1 support. |
| Fire TV | Jellyfin for Fire TV | Amazon Appstore. Same as Android TV. |
| Samsung Tizen | jellyfin-tizen | **Not in store** (failed review). Sideload via [one-click installer](https://github.com/PatrickSt1991/Samsung-Jellyfin-Installer). |
| LG webOS | jellyfin-webos | **Official LG Content Store** (2020+ TVs). |
| iOS | Swiftfin | App Store. Native Swift. Video only. |
| Roku | Jellyfin Roku | Roku Store. v3.0.15 (Dec 2025). |

---

## 2. Admin Curation Interface

### Existing Tools Analysis

| Tool | Search by keywords | Multi-platform | Preview | Checkbox select | Download + metadata | Docker |
|------|-------------------|----------------|---------|----------------|-------------------|--------|
| **Tube Archivist** | Only downloaded (Elasticsearch) | YouTube only | Yes (player) | Yes (multi-select + queue) | Yes (full) | Yes (3 containers) |
| **Pinchflat** | No (channel subscriptions only) | YouTube only | No | No | Yes (NFO) | Yes |
| **MeTube** | No (paste URL only) | Any via yt-dlp | No | No | Yes | Yes |
| **yt-dlp CLI** | `ytsearch:` only | YouTube search only | No | No | Yes (full) | No |
| **Invidious** | YouTube search API | YouTube only | Yes (player) | No | No | Yes |
| **SearXNG** | Multi-engine | No Rutube/VK | Links only | No | No | Yes |

**Conclusion: No existing tool covers the full workflow.** Closest is Tube Archivist (search + preview + select + queue) but it's YouTube-only by design (maintainer confirmed: no multi-platform planned).

### Recommended: Custom Admin UI

```
┌──────────────────────────────────────────────┐
│             Admin Curation UI                 │
│                                               │
│  Search bar: "смешарики"                      │
│  [YouTube ✓] [RuTube ✓] [VK Video ✓]         │
│                                               │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ │
│  │ thumb  │ │ thumb  │ │ thumb  │ │ thumb  │ │
│  │ title  │ │ title  │ │ title  │ │ title  │ │
│  │ 12:34  │ │ 5:20   │ │ 8:15   │ │ 22:00  │ │
│  │ YT     │ │ RT     │ │ VK     │ │ YT     │ │
│  │ [☑]    │ │ [☐]    │ │ [☑]    │ │ [☐]    │ │
│  └────────┘ └────────┘ └────────┘ └────────┘ │
│                                               │
│  [▶ Preview]         [⬇ Download Selected (2)]│
└──────────────────────────────────────────────┘
```

**Backend (Python/FastAPI):**
- Search aggregator: parallel queries to YouTube (yt-dlp `ytsearch:` or Invidious API), Rutube API, VK API
- Normalize results: `{id, platform, title, thumbnail, duration, url, description}`
- Download queue (Redis/SQLite + worker)
- yt-dlp subprocess: `--write-info-json --write-thumbnail --write-subs --embed-metadata`
- Output to Jellyfin media directory → auto-indexed
- Auth: JWT + roles (super-admin, curator)

**Frontend (React/Vue):**
- Search bar + platform filters
- Video card grid with thumbnails
- Checkbox selection + "Download selected" button
- Popup preview (iframe embed from source platform)
- Queue page with progress

### Platform Search APIs

| Platform | API | Auth | Notes |
|----------|-----|------|-------|
| YouTube | yt-dlp `ytsearch:N query` / Invidious API | None / self-hosted | Up to 1000 results |
| RuTube | `rutube.ru/api/search/` | Semi-public | Params: query, page, limit, duration, category |
| VK Video | VK API `video.search` | Access token required | Params: q, sort, hd, adult, filters |

---

## 3. SSO / Authentication

### VK and Yandex: OAuth2-only (not OIDC)

Both VK and Yandex implement OAuth 2.0 but **NOT** full OpenID Connect:
- No `id_token` returned
- No `.well-known/openid-configuration` endpoint
- Require an identity broker to convert OAuth2 → OIDC for downstream apps

### SSO Solutions Comparison

| | Authelia | Authentik | Keycloak |
|---|---------|-----------|----------|
| RAM | ~30 MB | ~2 GB | ~400 MB |
| VK/Yandex | **No** (no brokering) | Manual generic OAuth | **Yes** (dedicated plugin) |
| OIDC provider | Yes | Yes | Yes |
| Complexity | Low | Medium | High |
| Fit for N100 | Excellent | Marginal | Good |

**Winner: Keycloak** — only solution with [ready-made VK + Yandex plugins](https://github.com/playa-ru/keycloak-russian-providers).

### SSO Flow

```
User clicks "Login with VK"
    → Keycloak redirects to VK OAuth
    → VK returns access_token
    → Keycloak converts to OIDC id_token
    → jellyfin-plugin-sso receives OIDC token
    → Jellyfin creates/maps user account
```

### RAM Budget (Intel N100 / 8GB)

| Component | RAM (idle) | RAM (load) |
|-----------|-----------|------------|
| OS + system | 500 MB | 500 MB |
| Jellyfin | 200 MB | 500-1500 MB |
| Keycloak | 400 MB | 400 MB |
| Admin UI (FastAPI) | 50 MB | 100 MB |
| PostgreSQL | 100 MB | 200 MB |
| yt-dlp workers | 0 | 200 MB |
| nginx | 20 MB | 20 MB |
| **Total** | **~1.3 GB** | **~3 GB** |

Comfortable headroom on 8GB.

---

## 4. Content Pipeline (End-to-End)

```
1. Curator opens Admin UI → searches "смешарики"
2. Backend queries YouTube + RuTube + VK in parallel
3. Results displayed as cards with thumbnails
4. Curator previews (popup player from source platform)
5. Curator selects videos with checkboxes → "Download"
6. Backend queues yt-dlp jobs
7. yt-dlp downloads: video + metadata JSON + thumbnail + subtitles
8. Files saved to /media/videos/{platform}/{title}/
9. Jellyfin detects new files (inotify) → indexes automatically
10. Users see new content in Jellyfin apps (web, phone, TV)
```

---

## 5. References

### Platforms
- [Jellyfin](https://jellyfin.org/) — [GitHub](https://github.com/jellyfin/jellyfin) — [Docs](https://jellyfin.org/docs/)
- [MediaCMS](https://mediacms.io/) — [GitHub](https://github.com/mediacms-io/mediacms)
- [PeerTube](https://joinpeertube.org/) — [GitHub](https://github.com/Chocobozzz/PeerTube)

### SSO
- [Keycloak](https://www.keycloak.org/) — [Russian providers plugin](https://github.com/playa-ru/keycloak-russian-providers)
- [jellyfin-plugin-sso](https://github.com/9p4/jellyfin-plugin-sso)
- [Authelia](https://www.authelia.com/) — [Jellyfin integration](https://www.authelia.com/integration/openid-connect/clients/jellyfin/)

### Curation / Download
- [yt-dlp](https://github.com/yt-dlp/yt-dlp) — [Supported sites](https://github.com/yt-dlp/yt-dlp/blob/master/supportedsites.md)
- [Tube Archivist](https://github.com/tubearchivist/tubearchivist) — YouTube-only archiver
- [MeTube](https://github.com/alexta69/metube) — yt-dlp web UI
- [Pinchflat](https://github.com/kieraneglin/pinchflat) — YouTube channel auto-downloader
- [Invidious API](https://docs.invidious.io/api/) — YouTube search backend

### Jellyfin Apps
- [Android (Google Play)](https://play.google.com/store/apps/details?id=org.jellyfin.mobile)
- [Findroid (Google Play)](https://play.google.com/store/apps/details?id=dev.jdtech.jellyfin)
- [Android TV (Google Play)](https://play.google.com/store/apps/details?id=org.jellyfin.androidtv)
- [Fire TV (Amazon)](https://www.amazon.com/Jellyfin-for-Fire-TV/dp/B07TX7Z725)
- [Samsung Tizen installer](https://github.com/PatrickSt1991/Samsung-Jellyfin-Installer)
- [LG webOS (LG Store)](https://us.lgappstv.com/main/tvapp/detail?appId=1030579)
- [Swiftfin iOS (App Store)](https://apps.apple.com/us/app/swiftfin/id1604098728)

### API Sources
- [RuTube Node.js client](https://www.npmjs.com/package/rutube) — [PHP client](https://rutube.github.io/php-api-client/)
- [VK API video.search](https://dev.vk.com/en/method/video.search)

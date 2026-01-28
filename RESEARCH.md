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
| **Microblog** | **GoToSocial** | Lightest ActivityPub server (~300 MB RAM). Native OIDC (Keycloak). Allowlist federation = isolated. Admin-only registration. Mastodon API = Tusky/Phanpy apps. |
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

---

## 5. Tube Archivist Fork Analysis

### Architecture
- **Stack:** Python/Django + React SPA + Elasticsearch 8 + Redis/Celery
- **Search:** Local Elasticsearch index only (already-downloaded content). NO external search.
- **Download:** yt-dlp as Python library, Celery workers from Redis queue
- **Jellyfin integration:** Separate C# plugin reads API. File-based: shared filesystem.
- **Indices (7):** ta_config, ta_channel, ta_video, ta_download, ta_playlist, ta_subtitle, ta_comment

### YouTube Coupling
- `youtube_id` hardcoded as primary key across ALL 7 ES indices
- `urlparser.py` hardcodes youtube.com URL patterns and ID lengths (11=video, 24=channel)
- `remote_query.py` constructs `youtube.com/channel/` URLs directly
- `yt_dlp_base.py` handles YouTube-specific auth (cookies, PO tokens)

### Maintainer Stance
> "Downloading, indexing and playing videos from YouTube, there are currently no plans to expand this to any additional platforms. A scope too broad will result in development effort spreading too thin."

### Fork Feasibility Verdict

| Factor | Fork Tube Archivist | Build with FastAPI |
|--------|---|---|
| Time to MVP | 3-6 months (untangle YouTube coupling) | 2-3 months (clean design) |
| Technical debt | High (YouTube assumptions everywhere) | None |
| Upstream sync | Impossible after changes | N/A |
| Reusable code | Minimal (ES patterns, UX concepts) | N/A |

**Recommendation: Build from scratch with FastAPI.** The amount of reusable code from Tube Archivist is minimal compared to the effort of refactoring `youtube_id` out of 7 indices and hundreds of references.

---

## 6. Torrent Content Pipeline

### Torrent Search: Jackett vs Prowlarr

| Feature | **Jackett** | **Prowlarr** |
|---------|------------|-------------|
| Russian trackers | RuTracker, Kinozal, RuTor, NNM-Club, LostFilm, NewStudio | RuTracker only |
| Auto-sync with *arr | Manual Torznab URLs | Automatic |
| Docker | `linuxserver/jackett` | `linuxserver/prowlarr` |

**Winner: Jackett** — supports all 4 major Russian trackers natively.

**Bonus: [TorAPI](https://github.com/Lifailon/TorAPI)** — MIT-licensed REST API for RuTracker, Kinozal, RuTor, NNM-Club. Docker: `lifailon/torapi`. Swagger docs. Alternative if Jackett parsing fails.

### Metadata Catalogs

| Source | Free | Russian content | Posters | API |
|--------|------|----------------|---------|-----|
| **TMDB** | Yes | Localized (`ru`) | Yes | REST, ~50 req/s |
| **Kinopoisk Unofficial** | Yes (token) | Primary source | Yes | REST |
| **TheTVDB** | No ($12/yr) | Limited | Yes | REST v4 |

**Combo:** TMDB (primary) + Kinopoisk Unofficial (for Russian-specific content).

### Download Client

**qBittorrent** — best API, *arr integration, RSS filters. ~80 MB RAM. Docker: `linuxserver/qbittorrent`.

### Complete *arr Stack

```
Curator browses TMDB         Torrent search           Download          Media
┌──────────────┐    ┌──────────────────┐    ┌──────────────┐    ┌───────────┐
│  Jellyseerr  │───>│ Radarr (movies)  │───>│ qBittorrent  │───>│ Jellyfin  │
│  (request)   │    │ Sonarr (TV)      │<──>│              │    │           │
└──────────────┘    │  └── Jackett ──┘ │    └──────────────┘    └───────────┘
                    └──────────────────┘
```

**Jellyseerr** — Netflix-like request UI built for Jellyfin. Curators browse TMDB, request content, Radarr/Sonarr find matching torrents via Jackett, qBittorrent downloads, Jellyfin indexes.

### Additional RAM for Torrent Stack

| Service | RAM |
|---------|-----|
| Jellyseerr | 100 MB |
| Radarr + Sonarr | 300 MB |
| Jackett | 100 MB |
| qBittorrent | 80 MB |
| FlareSolverr (Cloudflare bypass) | 150 MB |
| **Total** | **~730 MB** |

### Known Issues with Russian Content
- Radarr/Sonarr parser struggles with Russian torrent naming (`Русское / English (Year)`)
- Jackett has "strip Russian letters" option — helps but not perfect
- For unmatched content, manual torrent add via Jackett web UI → qBittorrent

### Two Parallel Pipelines

```
Pipeline 1 (Streaming platforms): Custom Admin UI → yt-dlp → Jellyfin
Pipeline 2 (Torrents):           Jellyseerr → Radarr/Sonarr → Jackett → qBittorrent → Jellyfin
Both output to same Jellyfin media directories.
```

---

## 7. Microblogging / Social Platform for Kids

**Goal:** Add a microblogging feature to HomeCDN where ~100 users (kids in a community) can create profiles, post texts/photos/videos, follow each other, and see a feed. Must run on Intel N100 / 8GB RAM alongside Jellyfin + Keycloak.

### 7.1 Platform Comparison Table

| Feature | **GoToSocial** | **Akkoma** | **Mastodon** | **Sharkey** | **Pixelfed** |
|---------|---------------|-----------|-------------|------------|-------------|
| **Type** | Microblog (ActivityPub) | Microblog (ActivityPub) | Microblog (ActivityPub) | Microblog (ActivityPub) | Photo sharing (ActivityPub) |
| **Language** | Go | Elixir/Erlang | Ruby/Node.js | Node.js/TypeScript | PHP (Laravel) |
| **RAM idle** | **250-350 MB** | **700 MB - 1.3 GB** (with PostgreSQL) | **2-4 GB** (min 2 GB) | **~2 GB** (min) | **2-4 GB** |
| **RAM under load (~100 users)** | **500 MB - 1 GB** | **1-2 GB** | **3-6 GB** | **2-4 GB** | **2-4 GB** |
| **Docker** | Official image | Community images | Official | docker-compose | Official |
| **OIDC/SSO** | **Native OIDC** (documented Keycloak) | OAuth consumer (Keycloak strategy exists, **buggy** — issue #646) | **Native OIDC** (env vars, Keycloak documented) | **No OIDC** (open issue #9132) | **No OIDC** (unmerged PR #3436) |
| **Disable federation** | **Allowlist mode** (empty list = no federation) | **SimplePolicy accept list** (empty = no federation) | **LIMITED_FEDERATION_MODE** (dedicated mode) | Config-based | Config-based |
| **Admin-only registration** | **Default: closed.** Admin creates via CLI. Optional approval-based signup. | Closed registration + invite tokens + admin approval | Closed + invite-only + approval mode | Closed + invite | Closed + admin approval |
| **Moderation tools** | Reports, account freeze/suspend, domain blocks, allowlists, content warnings | Admin-FE panel, user roles, NSFW marking, MRF policies, invite management | Full admin panel, freeze/suspend/silence, domain blocks, email blacklists, appeals, API webhooks | Misskey-based admin, user roles, instance blocks | Reports, account management, content filters |
| **Content approval workflow** | Sign-up approval (not per-post) | MRF can filter content | Not per-post (moderation is reactive) | Not per-post | Not per-post |
| **Mobile apps** | Tusky, Megalodon, Feditext (iOS), any Mastodon client | Tusky, Fedilab, any Mastodon client | Official iOS/Android, Tusky, Megalodon, Ivory, 50+ clients | Misskey apps (Milktea, MissRirica), some Mastodon clients | Official Pixelfed app (beta) |
| **Web UI** | No built-in web client (use Phanpy, Semaphore, Elk) | Pleroma-FE (built-in), Mangane | Built-in (full-featured) | Built-in (Misskey UI, rich features) | Built-in (Instagram-like) |
| **Photo posts** | Yes (attachments) | Yes (attachments) | Yes (attachments) | Yes (attachments + reactions + drive) | **Primary focus** |
| **Video posts** | Yes (short clips) | Yes (short clips) | Yes (short clips) | Yes (short clips) | Yes |
| **License** | AGPL v3 | AGPL v3 | AGPL v3 | AGPL v3 | AGPL v3 |
| **Activity** | Active (Go, heading to v1.0 by end 2026) | Active (Elixir) | Very active (largest Fediverse project) | Active (Misskey fork) | Active (PHP) |
| **GitHub stars** | ~4k | ~1k | ~47k | ~3k | ~5.5k |

### 7.2 Lightweight Alternatives (Non-ActivityPub)

| Feature | **WriteFreely** | **Discourse** | **Friendica** |
|---------|----------------|--------------|--------------|
| **Type** | Minimalist blogging | Forum/community | Full social network (ActivityPub) |
| **RAM** | <500 MB | 1-2 GB | ~1-2 GB (PHP + MariaDB) |
| **Docker** | Community images | Official | Official |
| **OIDC** | No native OIDC | OIDC plugin available | No native OIDC |
| **Microblog features** | Blog-only, no feed/follow/comments | Topics, replies, profiles — **not microblog** | Posts, comments, photos, events — **heavy/complex** |
| **Verdict** | Too simple — no feed, no follow, no interactions | Wrong format — forum, not microblog | Too complex for kids, outdated UI |

**Verdict: None of the lightweight alternatives fit the microblog use case.**

Lemmy and Plume were also considered but rejected: Lemmy is Reddit-like (link aggregation, not microblog), and Plume is effectively abandoned.

### 7.3 Kid-Safety Analysis

| Requirement | **GoToSocial** | **Akkoma** | **Mastodon** |
|-------------|---------------|-----------|-------------|
| Admin-only registration | Default closed, CLI account creation | Closed + invite tokens | Closed + invite + approval |
| No external content (federation off) | Allowlist mode (empty = isolated) | SimplePolicy accept (empty = isolated) | LIMITED_FEDERATION_MODE |
| Moderation tools | Reports, freeze, suspend, domain control | Admin-FE, MRF policies, NSFW marking | Full panel, freeze/suspend/silence, appeals |
| Content approval (pre-moderation) | No per-post approval | MRF can auto-reject by keyword/pattern | No per-post approval |
| Age-appropriate UI | No built-in UI (can choose kid-friendly client like Phanpy/Elk) | Pleroma-FE (customizable themes) | Built-in UI (can customize CSS) |
| No external links/embeds | Configurable | MRF can strip links | Limited |

**Key finding:** None of the platforms have built-in per-post pre-moderation (approve before publish). All use reactive moderation (report/remove after the fact). For a kids' environment, this means relying on admin-only accounts + closed federation + community norms rather than content gating.

### 7.4 RAM Budget Analysis (Intel N100 / 8GB)

| Component | RAM |
|-----------|-----|
| OS + system | ~500 MB |
| Keycloak | ~400 MB |
| Jellyfin | ~500-800 MB |
| PostgreSQL (shared) | ~300-500 MB |
| **Remaining for microblog** | **~5.5-6.3 GB** |

- **GoToSocial**: 250-500 MB — fits easily, leaves 5+ GB free
- **Akkoma**: 700 MB - 1.3 GB (includes its own PostgreSQL usage) — fits, ~4-5 GB free
- **Mastodon**: 2-4 GB (web + Sidekiq + Redis + PostgreSQL) — tight, may swap under load

### 7.5 Keycloak OIDC Integration

| Platform | OIDC Status | Integration Effort |
|----------|------------|-------------------|
| **GoToSocial** | **Native OIDC support.** Config: `oidc-enabled: true`, `oidc-idp-name`, `oidc-issuer`, `oidc-client-id`, `oidc-client-secret`, scopes. [Documented with examples.](https://docs.gotosocial.org/en/latest/configuration/oidc/) | Low — config file only |
| **Mastodon** | **Native OIDC support.** Env vars: `OIDC_ENABLED=true`, `OIDC_ISSUER`, `OIDC_CLIENT_ID`, etc. [Keycloak guide exists.](https://frostillic.us/blog/posts/2022/11/10/tinkering-with-mastodon-keycloak-and-domino) | Low — env vars only |
| **Akkoma** | OAuth consumer via Ueberauth. `keycloak:ueberauth_keycloak_strategy`. **Known bug:** CSRF mismatch in newer Ueberauth versions breaks all OAuth providers ([issue #646](https://akkoma.dev/AkkomaGang/akkoma/issues/646)). | High — broken as of latest reports |
| **Sharkey/Misskey** | No OIDC support. [Open issue #9132.](https://github.com/misskey-dev/misskey/issues/9132) | Not possible without reverse proxy auth |
| **Pixelfed** | Unmerged PR #3436. Experimental, conflicts with other auth code. | Not production-ready |

### 7.6 Recommendation

**Winner: GoToSocial**

Reasons:
1. **Lowest RAM:** 250-500 MB idle, leaving 5+ GB for Jellyfin + Keycloak. No other platform comes close on an 8 GB machine.
2. **Native OIDC:** Documented Keycloak integration. Kids log in with the same account as Jellyfin.
3. **Allowlist federation mode:** Empty allowlist = fully isolated instance. No external content reaches the platform.
4. **Admin-only registration by default:** Admin creates all accounts via CLI. No self-signup.
5. **Mastodon API compatible:** Works with Tusky (Android), Feditext (iOS), Phanpy/Elk (web). Kids get real mobile apps.
6. **Written in Go:** Single binary, minimal dependencies, no Ruby/Node.js/Elixir runtime overhead.
7. **Active development:** Heading toward v1.0 (projected end 2026). Regular releases.

**Trade-offs:**
- No built-in web UI — must deploy a web client (Phanpy or Elk) as a separate static site or use mobile apps only.
- No per-post pre-moderation — reactive moderation only (freeze/suspend accounts, delete posts after the fact).
- Still in alpha/beta (pre-v1.0) — some features incomplete, but core posting/following/feed works.

**Runner-up: Mastodon** — if more RAM were available (16 GB machine), Mastodon would be the safest choice due to maturity, built-in web UI, and the largest ecosystem. But 2-4 GB base RAM makes it impractical on an 8 GB shared machine.

### 7.7 Proposed Architecture with GoToSocial

```
┌──────────────────────────────────────────────────────────────────────┐
│                    HomeCDN Server (Intel N100, 8GB RAM)               │
│                                                                      │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │  Jellyfin   │  │  GoToSocial  │  │  Keycloak    │               │
│  │  (video)    │  │  (microblog) │  │  (SSO/OIDC)  │               │
│  │  ~600 MB    │  │  ~300 MB     │  │  ~400 MB     │               │
│  │             │  │  OIDC→KC     │  │              │               │
│  └─────────────┘  └──────────────┘  └──────────────┘               │
│                                                                      │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │  PostgreSQL │  │  Phanpy/Elk  │  │  Nginx       │               │
│  │  (shared DB)│  │  (web client │  │  (reverse    │               │
│  │  ~400 MB    │  │  for GtS)   │  │  proxy)      │               │
│  │             │  │  static      │  │              │               │
│  └─────────────┘  └──────────────┘  └──────────────┘               │
│                                                                      │
│  Total estimated: ~2 GB (leaving ~6 GB for OS cache + media ops)    │
└──────────────────────────────────────────────────────────────────────┘
```

GoToSocial config highlights:
- `instance-federation-mode: "allowlist"` (empty allowlist = no federation)
- `accounts-registration-open: false` (admin creates accounts via CLI)
- `oidc-enabled: true` + Keycloak issuer/client settings
- `GTS_CACHE_MEMORY_TARGET=100MB` (tune for available RAM)
- SQLite backend option available (no PostgreSQL dependency, even lighter)

### 7.8 Sources

- [GoToSocial deployment considerations](https://docs.gotosocial.org/en/latest/getting_started/)
- [GoToSocial OIDC configuration](https://docs.gotosocial.org/en/latest/configuration/oidc/)
- [GoToSocial federation modes](https://docs.gotosocial.org/en/latest/admin/federation_modes/)
- [GoToSocial sign-ups](https://docs.gotosocial.org/en/latest/admin/signups/)
- [GoToSocial moderation](https://docs.gotosocial.org/en/latest/federation/moderation/)
- [GoToSocial Docker image](https://hub.docker.com/r/superseriousbusiness/gotosocial)
- [GoToSocial real-world RAM comparison](https://blog.klein.ruhr/gotosocial-ready-for-prime-time)
- [Mastodon OIDC PR #16221](https://github.com/mastodon/mastodon/pull/16221)
- [Mastodon LIMITED_FEDERATION_MODE](https://fedi.tips/creating-an-isolated-server/)
- [Mastodon moderation docs](https://docs.joinmastodon.org/admin/moderation/)
- [Mastodon 4.5 release](https://blog.joinmastodon.org/2025/11/mastodon-4.5/)
- [Mastodon Keycloak setup](https://frostillic.us/blog/posts/2022/11/10/tinkering-with-mastodon-keycloak-and-domino)
- [Akkoma Docker installation](https://docs.akkoma.dev/stable/installation/docker_en/)
- [Akkoma search memory usage](https://docs.akkoma.dev/stable/configuration/search/)
- [Akkoma OAuth bug #646](https://akkoma.dev/AkkomaGang/akkoma/issues/646)
- [Akkoma MRF configuration](https://docs.akkoma.dev/stable-docs/configuration/mrf/)
- [Misskey OIDC issue #9132](https://github.com/misskey-dev/misskey/issues/9132)
- [Pixelfed Docker prerequisites](https://jippi.github.io/docker-pixelfed/installation/prerequisites/)
- [Pixelfed OIDC PR #3436](https://github.com/pixelfed/pixelfed/pull/3436)
- [Firefish end of support](https://codeberg.org/firefish/firefish)
- [Sharkey overview](https://blog.elenarossini.com/sharkey-a-fediverse-project-that-is-beautiful-inside-out/)
- [Pleroma hosting metrics](https://write.as/golden-demise/hosting-a-pleroma-instance-metrics-and-costs)
- [Mastodon scaling down](https://gist.github.com/nolanlawson/fc027de03a7cc0b674dcdc655eb5f2cb)
- [Phanpy web client for GoToSocial](https://www.asty.org/2025-fediverse-update-gotosocial-server-and-the-phanpy-web-client-are-great/)

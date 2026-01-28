# HomeCDN

Local video hosting platform for home networks (~100 users).

**Problem:** Unstable internet, repeated content consumption, no local video library.

**Solution:** Self-hosted video platform with automatic content download from YouTube/RuTube by channel subscriptions, plus graceful fallback when external platforms are unavailable.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     HomeCDN Server (Intel N100)                  │
│                                                                  │
│  ┌──────────────┐   ┌──────────────┐   ┌─────────────────────┐  │
│  │   MediaCMS    │   │  Pinchflat   │   │  Error Page Server  │  │
│  │  (UI, search, │   │  (auto-dl    │   │  (landing when      │  │
│  │   accounts,   │   │   from YT/   │   │   external sites    │  │
│  │   tags)       │   │   RuTube)    │   │   return errors)    │  │
│  │              │   │              │   │                     │  │
│  │  video.home   │   │              │   │                     │  │
│  └──────┬───────┘   └──────┬───────┘   └─────────┬───────────┘  │
│         │                  │                      │              │
│         └──────────┬───────┘                      │              │
│                    ▼                              │              │
│  ┌─────────────────────────────┐                  │              │
│  │    Shared Storage (HDD)     │                  │              │
│  │    /media/videos/           │                  │              │
│  └─────────────────────────────┘                  │              │
│                                                    │              │
├────────────────────────────────────────────────────┘──────────────┤
│                    Keenetic Router                                │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  DNS: video.home → 192.168.1.100                         │    │
│  │  Gateway mode: HomeCDN as gateway for selected devices   │    │
│  │  Error fallback: non-200 from YT/RuTube → landing page  │    │
│  └──────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

## Core Components

### MediaCMS — Video Platform

Self-hosted YouTube-like platform providing:
- Web UI with video player, search, tags, categories
- User accounts with personalized recommendations
- Accessible at `video.home` on local network
- Stores transcoded video on shared HDD storage

### Pinchflat — Content Downloader

Automated video downloader:
- Subscribe to YouTube/RuTube channels
- Auto-download new videos on schedule
- Output to shared storage consumed by MediaCMS
- Web UI for managing subscriptions

### Graceful Fallback

When external video platforms are unavailable — show a helpful landing page instead of a browser error.

**How it works:**
1. User tries to access `youtube.com` or `rutube.ru`
2. Request goes through Keenetic gateway → HomeCDN server
3. If external platform returns non-200 (error, timeout, network issue) → serve landing page
4. Landing page shows: network status + link to `video.home` (local library)
5. If external platform responds normally → no interference, pass through transparently

**Key principle:** zero interference when internet works. Fallback only on actual errors.

## User Accounts & Recommendations

MediaCMS supports multiple user accounts:
- Each family member has their own profile
- Watch history and recommendations per user
- Age-appropriate content tagging
- ~100 users on the local network (apartment building / community)

## Content Management

### Auto-Download Workflow

1. Admin subscribes to channels in Pinchflat (cartoons, educational, etc.)
2. Pinchflat downloads new videos to shared HDD
3. MediaCMS picks up new files and indexes them
4. Users browse and watch via `video.home`

### Supported Source Platforms

| Platform | Download Support | Notes |
|----------|-----------------|-------|
| YouTube | yt-dlp | Full support, DASH/HLS |
| RuTube | yt-dlp | Full support, HLS |
| VK Video | yt-dlp | Partial, auth may be required |
| Odnoklassniki | yt-dlp | Partial |

## Network Setup

### Keenetic Gateway Mode

HomeCDN server acts as gateway for devices on the local network:

1. HomeCDN server: static IP `192.168.1.100`
2. Keenetic DHCP assigns HomeCDN as gateway for selected devices (by MAC)
3. DNS record: `video.home` → `192.168.1.100`

```
User device (192.168.1.50)
    → Gateway: 192.168.1.100 (HomeCDN)
        → Internet works: pass through transparently
        → Internet error: serve landing page with link to video.home
```

### Error Page Fallback Configuration

On the HomeCDN server (acting as gateway):
- Intercept DNS or HTTP responses from video platform domains
- On non-200 / timeout / network error → return landing page
- On success → transparent passthrough
- No TLS interception, no MITM — only error detection at network level

## System Requirements

- **Target OS:** Linux (Ubuntu 22.04 LTS)
- **Architecture:** x86_64 (amd64)
- **Hardware:**
  - CPU: Intel N100 or equivalent
  - RAM: 4GB minimum, 8GB recommended (MediaCMS + Pinchflat + transcoding)
  - Storage: HDD for video library (recommended >1TB)
  - Network: Gigabit Ethernet
- **Recommendation:** Install OS on a separate disk from video storage. If HDD fails, the system remains operational.

## Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Video platform | MediaCMS | UI, accounts, search, playback |
| Content downloader | Pinchflat | Auto-download by subscriptions |
| Video engine | yt-dlp | Download from YouTube/RuTube |
| Gateway/fallback | nginx or custom | Error detection, landing page |
| DNS | Keenetic / dnsmasq | `video.home` resolution |
| Storage | Filesystem (HDD) | Shared video library |
| Deployment | Docker Compose | All services containerized |

## Project Structure

```
homecdn/
├── README.md              # This file
├── docker-compose.yml     # All services
├── mediacms/              # MediaCMS configuration
├── pinchflat/             # Pinchflat configuration
├── fallback/              # Error page server + landing page
├── nginx/                 # Gateway/proxy configuration
└── materials/             # Research, prototypes
```

## References

### Research Sources

- [MediaCMS — Self-hosted CMS](https://mediacms.io/)
- [Pinchflat — YouTube auto-downloader](https://github.com/kieraneglin/pinchflat)
- [RuTube CDN Architecture (Habr)](https://habr.com/ru/companies/habr_rutube/articles/919360/)
- [RuTube 10 Tbps Architecture (Habr)](https://habr.com/ru/companies/habr_rutube/articles/887748/)
- [yt-dlp — Video downloader](https://github.com/yt-dlp/yt-dlp)
- [LanCache Documentation](https://lancache.net/docs/)

### Similar Projects

- [LanCache](https://lancache.net/) — game/update caching
- [Tube Archivist](https://www.tubearchivist.com/) — YouTube archive manager
- [Jellyfin](https://jellyfin.org/) — self-hosted media system

## License

MIT

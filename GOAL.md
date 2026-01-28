# HomeCDN — Project Goals

## Vision

Local video platform for communities (~100 users): festivals, apartment buildings, camps.
Admin-curated content library accessible via web, mobile apps, and Smart TV.

## Goal 1: Admin Curation Interface

Dedicated web UI for authorized curators (invite-only, added by super-admin).

### Search & Discovery
- Search bar: enter keywords (cartoon name, topic, etc.)
- Search across multiple sources: YouTube, RuTube, VK Video, peertube instances, torrent/P2P
- Results displayed as video cards: thumbnail, title, duration, source platform, rating

### Preview
- Click card → popup player or open source link in new tab
- Preview without downloading — stream from original platform
- Quick preview: play inline, close, move to next

### Selection & Download Queue
- Checkbox on each card to select videos for download
- "Download selected" button → adds to download queue
- Queue manager: priority, status, progress, retry on failure
- Downloads video file + metadata (title, description, tags, thumbnail, subtitles)
- Downloaded content automatically appears in the catalog

### Access Control
- Super-admin adds curators by invite link or manually
- Curators can search, preview, select, queue downloads
- Regular users can only browse and watch the catalog
- Role-based: super-admin, curator, viewer

## Goal 2: Multi-Platform Client Access

### Web Interface (Primary)
- YouTube-like UI: search, categories, tags, watch history
- User accounts with profiles
- SSO support: VK, Yandex, or other OAuth providers
- Responsive design for mobile browsers

### Mobile App (Android — Google Play)
- Native or hybrid app for Android
- Connect to server by URL (e.g., `video.home` or public domain)
- Browse catalog, search, watch videos
- Ideally: open-source client compatible with our backend
- User opens a link → app launches and plays content

### Smart TV App
- App for Android TV / Samsung Tizen / LG webOS
- Same functionality: browse, search, watch
- Install from app store or sideload
- Connect to server URL, auto-discover content

### Landing Page for Distribution
- Public landing page (accessible from internet or local network)
- Links to download mobile app (Google Play) and Smart TV app
- QR code for quick install
- After install, app auto-connects to the local server

## Goal 3: Content Pipeline

```
Curator searches → selects videos → download queue
    ↓
yt-dlp / torrent client downloads video + metadata
    ↓
Video stored on HDD (RAID for reliability)
    ↓
Platform indexes new content automatically
    ↓
Available to all users via web / mobile / TV
```

## Goal 4: Infrastructure

- Server: Intel N100, 8GB+ RAM, HDD RAID (2TB+)
- All services in Docker Compose
- Local network: Keenetic gateway, DNS `video.home`
- Optional: public domain for remote access (festivals, events)
- Graceful fallback: when YouTube/RuTube down → landing page → local library

## Success Criteria

1. Curator can search "смешарики" → see results from YouTube + RuTube → select 10 episodes → they download and appear in catalog within hours
2. User installs app on phone → enters server address → browses and watches videos
3. Smart TV app works the same way
4. 100 concurrent users can watch without buffering on local gigabit network
5. System runs unattended after initial setup

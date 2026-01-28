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

## Goal 5: Torrent Content Pipeline

Second content source alongside YouTube/RuTube/VK streaming platforms.

### Flow
1. Curator opens Jellyseerr → browses TMDB/Kinopoisk catalog (posters, descriptions)
2. Requests movie/series → Radarr/Sonarr searches torrent trackers via Jackett
3. Matches found → qBittorrent downloads → Radarr/Sonarr renames/organizes
4. Jellyfin indexes new content automatically

### Trackers
- RuTracker, Kinozal, RuTor, NNM-Club (all supported by Jackett)
- Primary content: children's cartoons, educational material, public domain

### Stack
- Jellyseerr (request UI) → Radarr + Sonarr (management) → Jackett (indexer) → qBittorrent (download)

## Goal 6: Community Social Network

Unified platform where the whole community socializes around content.

### Concept
One ecosystem where:
- Parents/curators add educational content and cartoons to the video library
- Kids browse, watch, and discover content through Jellyfin
- Kids create micro-blogs: write about what they learned, share favorite videos, post photos/short clips
- Kids follow each other, see a feed of friends' posts, comment, recommend content
- Adults can blog for each other too (though they have broader internet access)

### Requirements
- Integrated login (SSO via Keycloak — same account for video + microblog)
- Closed network (no external federation, no outside content)
- Admin-only account creation (super-admin creates accounts for each community member)
- Moderation tools (freeze/suspend accounts, remove posts)
- Mobile apps (Android — Tusky/similar; web — Phanpy/Elk)
- Lightweight (runs on same Intel N100 alongside Jellyfin)

### Not Two Separate Apps — One Experience
Although technically Jellyfin (video) + GoToSocial (microblog) are separate services, from the user's perspective:
- Same login (Keycloak SSO)
- Same landing page with links to both
- Kids can share Jellyfin video links in their GoToSocial posts
- Curators can announce new content via GoToSocial

## Success Criteria

1. Curator can search "смешарики" → see results from YouTube + RuTube + torrents → select 10 episodes → they download and appear in catalog within hours
2. User installs app on phone → enters server address → browses and watches videos
3. Smart TV app works the same way
4. Kid creates a micro-blog post about a cartoon they watched → friends see it in their feed
5. 100 concurrent users can watch without buffering on local gigabit network
6. System runs unattended after initial setup

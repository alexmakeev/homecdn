# HomeCDN

A high-performance content-addressable caching proxy for home networks, written in Rust.

**Problem:** Kids watch the same cartoons 50 times. Each replay downloads gigabytes from the internet.

**Solution:** Transparent caching proxy that identifies identical video content regardless of URL and serves it from local storage.

## How It Works

```
Internet ← Keenetic Router ← HomeCDN Server ← Kids' Devices
                                   ↓
                            Multi-TB Cache Disk
```

HomeCDN acts as a gateway for selected devices. All their traffic flows through it, and video content gets cached using content-addressable storage.

### Content-Addressable Caching

Traditional proxies use URLs as cache keys. This fails for video platforms because:
- URLs contain expiring tokens (YouTube: `expire=`, `signature=`)
- Same video = different URL every request
- Cache hit ratio: <7%

**HomeCDN approach:**
1. Intercept HTTPS traffic (MITM proxy)
2. Detect video CDN domains
3. Hash first 64KB of content → `content_key`
4. Same content = same hash, regardless of URL
5. Cache hit ratio: 90%+ for repeated content

## Research Summary

### Platform Analysis

| Platform | Protocol | Chunk Size | CDN Domains |
|----------|----------|------------|-------------|
| YouTube | DASH | Variable | `*.googlevideo.com` |
| RuTube | HLS | 4 sec | `vod*.rutube.ru` |
| VK Video | HLS | Variable | `*.vkuservideo.ru`, `*.vkcdn.ru` |
| Odnoklassniki | HLS | Variable | `*.mycdn.me` |
| ivi | HLS | Variable | `*.ivi.ru` |

### Why URL-Based Caching Fails

#### YouTube
- URLs contain time-based tokens: `expire=`, `signature=`
- Tokens expire within minutes
- Parameter `lmt` (last modified timestamp) is stable but requires URL parsing
- Example: `...&lmt=1649501793701287&expire=1654300000&signature=...`

#### RuTube
- Uses HLS with 4-second chunks
- CDN distributes load across 250+ servers
- Video ID extractable via API: `/api/play/options/{video_id}/`
- Chunks cached internally by RuTube's nginx proxy_cache

### Why Content-Hash Caching Works

Video chunks contain unique frame data. Same video + same quality = identical bytes.

```
Request 1: https://cdn.example.com/chunk?token=abc123&expire=1000
Request 2: https://cdn.example.com/chunk?token=xyz789&expire=2000
           ↓                              ↓
        [identical 2MB video data]     [identical 2MB video data]
           ↓                              ↓
        SHA256 = 0x1a2b3c...           SHA256 = 0x1a2b3c...
           ↓                              ↓
        CACHE HIT!                     Served from cache
```

First 64KB is sufficient for identification:
- Unique per chunk (different frames)
- Fast to compute
- Collision probability: negligible for video content

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      HomeCDN Server                         │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │   Network   │  │    TLS     │  │   Content Router    │ │
│  │   Gateway   │→│   MITM     │→│  (domain matching)   │ │
│  │  (iptables) │  │  (rustls)  │  │                     │ │
│  └─────────────┘  └─────────────┘  └──────────┬──────────┘ │
│                                                │            │
│                    ┌───────────────────────────┼───────────┐│
│                    │              ┌────────────┴─────────┐ ││
│                    │              ▼                      │ ││
│  ┌─────────────┐  │  ┌─────────────────────────────────┐│ ││
│  │  Passthrough│←─┼──│      Video Cache Handler        ││ ││
│  │   (non-CDN) │  │  │  ┌─────────────────────────┐   ││ ││
│  └─────────────┘  │  │  │  1. Start streaming     │   ││ ││
│                    │  │  │  2. Hash first 64KB    │   ││ ││
│                    │  │  │  3. Check cache        │   ││ ││
│                    │  │  │  4. Serve or store     │   ││ ││
│                    │  │  └─────────────────────────┘   ││ ││
│                    │  └─────────────────────────────────┘│ ││
│                    └────────────────────────────────────┬┘ ││
│                                                         │  ││
│  ┌──────────────────────────────────────────────────────┴┐ ││
│  │              Content-Addressable Storage              │ ││
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────┐  │ ││
│  │  │   Index    │  │   Cache    │  │    Cleanup     │  │ ││
│  │  │  (SQLite)  │  │   Files    │  │    (LRU)       │  │ ││
│  │  │            │  │ /aa/bb/... │  │                │  │ ││
│  │  └────────────┘  └────────────┘  └────────────────┘  │ ││
│  └───────────────────────────────────────────────────────┘ ││
└─────────────────────────────────────────────────────────────┘
```

## Video CDN Domains

```rust
const VIDEO_CDN_PATTERNS: &[&str] = &[
    // YouTube
    r".*\.googlevideo\.com",
    r".*\.youtube\.com",

    // RuTube
    r"vod.*\.rutube\.ru",
    r".*\.rutube\.ru",

    // VK Video
    r".*\.vkuservideo\.ru",
    r".*\.vkcdn\.ru",
    r".*\.vk-cdn\.net",

    // Odnoklassniki
    r".*\.mycdn\.me",

    // ivi
    r".*\.ivi\.ru",

    // CTC/Mult
    r".*\.ctc\.ru",

    // Generic CDNs (used by many platforms)
    r".*\.akamaized\.net",
    r".*\.cloudfront\.net",
    r".*\.fastly\.net",
];
```

## Network Setup

### Option 1: Gateway Mode (Recommended)

Configure Keenetic to use HomeCDN as gateway for specific devices:

1. HomeCDN server gets static IP (e.g., 192.168.1.100)
2. In Keenetic DHCP, set custom gateway for kids' devices MAC addresses
3. All traffic from those devices routes through HomeCDN

```
Kids' iPad (192.168.1.50)
    → Gateway: 192.168.1.100 (HomeCDN)
        → NAT to Keenetic (192.168.1.1)
            → Internet
```

### Option 2: Explicit Proxy

Configure devices to use HomeCDN as HTTP/HTTPS proxy:
- Proxy: 192.168.1.100:8080
- Requires CA certificate installation on devices

## System Requirements

- **Target OS:** Linux (Ubuntu 22.04 LTS)
- **Architecture:** x86_64 (amd64)
- **Hardware:**
  - RAM: 2GB minimum, 4GB recommended
  - Storage: HDD for cache (any size, recommended >100GB)
  - Network: Gigabit Ethernet
- **Recommendation:** Install OS on a separate disk from cache storage. If cache HDD wears out or fails, the system remains operational.

## Technology Stack

- **Language:** Rust (performance, safety, async)
- **Async Runtime:** tokio
- **TLS:** rustls + rcgen (certificate generation)
- **HTTP:** hyper
- **Storage:**
  - SQLite (index: hash → file path, metadata)
  - Filesystem (content: `/{hash[0:2]}/{hash[2:4]}/{hash}.bin`)
- **Hashing:** SHA-256 (first 64KB)
- **Network:** iptables/nftables for transparent proxy

## Expected Performance

### Bandwidth Savings

| Scenario | Without Cache | With HomeCDN | Savings |
|----------|--------------|--------------|---------|
| Same cartoon 50x | 50 × 500MB = 25GB | 500MB + 50 × ~0 | 98% |
| OS updates (4 devices) | 4 × 3GB = 12GB | 3GB | 75% |
| Mixed usage | 150GB/month | ~30GB/month | 80% |

### Latency

- Cache hit: <5ms (local SSD)
- Cache miss: +10-20ms overhead (hashing + storage)

## Limitations

1. **HTTPS MITM required** — need to install CA certificate on devices
2. **Certificate pinning** — some apps may reject proxy (banking apps)
3. **Live streams** — cannot cache real-time content
4. **DRM content** — Netflix, etc. use encryption that prevents caching

## Project Structure

```
homecdn/
├── README.md           # This file
├── materials/          # Research, prototypes, experiments
├── src/
│   ├── main.rs
│   ├── proxy/          # MITM proxy implementation
│   ├── cache/          # Content-addressable storage
│   ├── tls/            # Certificate generation
│   └── config/         # Configuration handling
├── Cargo.toml
└── config.example.toml
```

## References

### Research Sources

- [RuTube CDN Architecture (Habr)](https://habr.com/ru/companies/habr_rutube/articles/919360/)
- [RuTube 10 Tbps Architecture (Habr)](https://habr.com/ru/companies/habr_rutube/articles/887748/)
- [Reverse-Engineering YouTube](https://tyrrrz.me/blog/reverse-engineering-youtube)
- [YouTube itag Codes](https://gist.github.com/sidneys/7095afe4da4ae58694d128b1034e01e2)
- [yt-dlp format_lmt Discussion](https://github.com/yt-dlp/yt-dlp/issues/11840)
- [NGINX Content Caching](https://docs.nginx.com/nginx/admin-guide/content-cache/content-caching/)
- [HLS Caching with NGINX](https://help.cesbo.com/misc/tools-and-utilities/network/hls-caching-proxy-with-nginx/)
- [Varnish for Video Streaming](https://info.varnish-software.com/blog/three-features-varnish-ideal-video-streaming)
- [LanCache Documentation](https://lancache.net/docs/)

### Similar Projects

- [LanCache](https://lancache.net/) — game/update caching (HTTP only)
- [nginx-vod-module](https://github.com/kaltura/nginx-vod-module) — on-the-fly HLS/DASH
- [HLS-Proxy](https://github.com/warren-bank/HLS-Proxy) — Node.js HLS proxy

## License

MIT

## Contributing

Contributions welcome! See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

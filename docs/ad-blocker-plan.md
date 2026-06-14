# Rust DNS Ad Blocker — Research & Implementation Plan

> A network-wide, DNS-based ad blocker written from scratch in Rust, running on a
> Raspberry Pi, with your stock ISP router pointing at it.

_Last updated: 2026-06-14_

---

## 1. Your requirements (as gathered)

| Decision | Choice |
|---|---|
| **Platform** | System-wide DNS blocker (Pi-hole-style) |
| **Deployment** | Router-level — stock ISP router's DNS points at the Pi |
| **Host device** | Raspberry Pi (always-on) |
| **Language** | Rust, built from scratch |
| **Blocklists** | Public lists + your own custom allow/deny rules |
| **Goal** | No ads on websites, YouTube, smart-TV/streaming apps, and mobile apps |
| **First deliverable** | This research pass + written plan |

---

## 2. The honest reality check (read this first)

DNS blocking is excellent for **some** of your goals and **structurally incapable**
of others. Setting expectations up front saves disappointment later.

### What a DNS blocker WILL kill ✅
- Banner/display ads, pop-ups, and pop-unders on websites
- The vast majority of **trackers and telemetry** (this is the biggest privacy win)
- Most **in-app ads on mobile** (they call out to known ad-network domains)
- Many smart-TV telemetry/ad endpoints (Roku, Samsung, LG, Fire TV beaconing)
- Ad-heavy redirects and malvertising domains

### What a DNS blocker will NOT reliably kill ❌
- **YouTube video ads.** YouTube serves ads from the *same domains* as the video
  itself (and rotates randomized subdomains). Block the domain and the video breaks
  too; allow it and the ads come through. This is not a tuning problem — it's
  architectural, and it is **not fixable at the DNS layer**.
- **In-stream ads inside streaming apps** (Hulu, Peacock, etc.) — same
  same-domain / server-side-stitched problem.
- **Server-side ad injection (SSAI)**, which YouTube and others are expanding —
  the ad is spliced into the video stream itself, so there is no separate request
  to block at any layer.

**Conclusion:** DNS blocking is the right *foundation* for "no ads on websites and
trackers everywhere," but YouTube/streaming video ads need a **second, content-level
layer**. See §6.

Sources: see §9.

---

## 3. Recommended architecture

```
                        ┌─────────────────────────────────────────┐
                        │            Raspberry Pi                  │
   All LAN devices      │                                          │
  ┌───────────┐         │   ┌──────────────────────────────────┐  │
  │  Phones   │         │   │   rust-adblock-dns (our binary)   │  │
  │  Laptops  │  DNS    │   │                                   │  │
  │  Smart TV │────────▶│──▶│  1. UDP/TCP :53 listener          │  │
  │  Tablets  │ :53     │   │  2. Parse query                   │  │
  └───────────┘         │   │  3. Match against blocklist       │  │
        ▲               │   │     ├─ blocked → 0.0.0.0 / NXDOMAIN│  │
        │               │   │     └─ allowed → cache or forward  │  │
   Router DHCP hands     │   │  4. Cache (TTL-aware)             │  │
   out Pi's IP as DNS   │   │  5. Forward upstream via DoH/DoT  │──┼──▶ Cloudflare/
        │               │   └──────────────────────────────────┘  │    Quad9 (encrypted)
  ┌───────────┐         │            │                             │
  │   Router  │         │            ▼                             │
  │ (stock    │         │   ┌──────────────────────────────────┐  │
  │  ISP)     │         │   │  Blocklist updater (cron/loop)    │  │
  └───────────┘         │   │  pulls OISD + HaGeZi + custom     │  │
                        │   └──────────────────────────────────┘  │
                        └─────────────────────────────────────────┘
```

### Why this shape
- **Stock ISP router** can't run software, but almost all let you set a custom DNS
  server (either in router settings, or via DHCP). The Pi becomes that DNS server.
- **The Pi** runs our Rust binary 24/7. Low power, silent, perfect fit.
- **Encrypted upstream (DoH/DoT)** means the queries we *do* forward aren't visible
  to your ISP — a privacy upgrade over plain DNS.

---

## 4. Technology choices (Rust)

| Concern | Recommendation | Notes |
|---|---|---|
| Async runtime | **`tokio`** | Standard for network services in Rust. |
| DNS protocol | **`hickory-dns`** (formerly `trust-dns`) | Mature wire-format parsing for queries/responses, plus `hickory-resolver` and `hickory-server` building blocks. We use it for parsing/encoding and as the upstream resolver; we write the *blocking + caching* logic ourselves. |
| Upstream DoH/DoT | `hickory-resolver` (has DoT/DoH/DoQ support) or `reqwest` for raw DoH | Forward allowed queries to Cloudflare (`1.1.1.1`) / Quad9 (`9.9.9.9`) encrypted. |
| Config | `serde` + `toml` | `config.toml` for upstreams, lists, custom rules, ports. |
| Blocklist fetch | `reqwest` | Download + refresh public lists on a schedule. |
| Matching | `HashSet<String>` for exact domains; suffix-tree/`Aho-Corasick` if we add wildcard rules | Start simple, optimize later. |
| Logging/metrics | `tracing` | Query log + block stats; feeds a future dashboard. |

> **Note on `hickory-dns` maturity:** the project itself flags that running its
> *server* in production is not yet recommended. We mitigate by using it primarily
> as a **library** for parsing and upstream resolution, while owning the listener,
> blocking, and cache ourselves. This keeps the risky surface small and under our
> control.

---

## 5. Blocklists (public + custom)

Based on current (2026) community consensus, "quality over quantity" — huge lists
break sites without blocking more ads:

- **OISD Big** (`oisd.nl`) — aggressive but strict about not breaking sites. Great default.
- **HaGeZi Multi Pro** — the current "gold standard," tiered by aggressiveness.
- Optionally an **IoT/smart-TV telemetry list** to catch TV beaconing.

Plus **your custom layer**:
- `denylist.txt` — domains you always block.
- `allowlist.txt` — domains you always permit (overrides public lists; for when a
  list breaks a site you need).

Refresh public lists on a schedule (e.g. daily). Custom rules always win.

---

## 6. The YouTube / streaming layer (your "research options first" item)

Since DNS can't do this, here are the realistic 2026 options, by device. None of
these are the Rust binary — they're complementary tools we'd document and set up.

| Device | Best option | What it does |
|---|---|---|
| **Desktop browsers** | **uBlock Origin** (+ **SponsorBlock**) | Still the most effective; kills YouTube pre/mid-roll. SponsorBlock also skips creator-read sponsor segments. Needs filter updates every ~1–2 weeks. |
| **Android phones/tablets** | **ReVanced** (patched YouTube) or **NewPipe** | Removes all YouTube ads; ReVanced also unlocks Premium-style features. Requires sideloading. |
| **Android TV / Fire TV / Shield** | **SmartTube** | Open-source TV YouTube client, blocks all ads, remote-friendly. Sideloaded. |
| **Proprietary TVs (Roku/Samsung/LG native apps)** | No reliable blocker | Options: screen-mirror from an ad-blocked device, or accept ads. |

**Important caveat:** YouTube is expanding **server-side ad injection**, which can
eventually defeat even these client-side tools. This is an arms race, not a
permanent fix.

> **Decision needed from you later:** which of these device-level tools you actually
> want set up. We can keep the Rust project focused on DNS and treat these as a
> documented companion guide.

---

## 7. Build roadmap (phased)

### Phase 0 — Foundations (setup guide)
- Flash Raspberry Pi OS Lite, set a **static IP** for the Pi.
- Install Rust toolchain on the Pi (or cross-compile from your dev machine for ARM).
- Decide port strategy (`:53` needs `CAP_NET_BIND_SERVICE` or root).

### Phase 1 — MVP resolver (the core loop)
- [ ] UDP listener on `:53`, parse incoming DNS query (`hickory-proto`).
- [ ] Forward **all** queries to a single upstream (plain DNS first), return the answer.
- [ ] Verify end-to-end: `dig @<pi-ip> example.com` resolves.

### Phase 2 — Blocking
- [ ] Load a blocklist file into a `HashSet`.
- [ ] If queried domain is blocked → return `0.0.0.0` / `NXDOMAIN`; else forward.
- [ ] Add `allowlist`/`denylist` custom overrides (custom wins).

### Phase 3 — Caching & performance
- [ ] TTL-aware response cache to cut latency + upstream load.
- [ ] TCP listener (fallback for large responses).
- [ ] Concurrency hardening under `tokio`.

### Phase 4 — Privacy upstream
- [ ] Swap plain-DNS upstream for **DoH/DoT** (Cloudflare/Quad9).
- [ ] Config-driven upstream selection + failover.

### Phase 5 — Blocklist automation
- [ ] Scheduled fetch/refresh of OISD + HaGeZi.
- [ ] Atomic reload without dropping queries.

### Phase 6 — Observability (optional, toward a dashboard)
- [ ] Query log + per-domain block counters via `tracing`.
- [ ] (Later) small web dashboard for stats and live allow/deny.

### Phase 7 — Deployment
- [ ] `systemd` service so it survives reboots.
- [ ] Point router DHCP DNS → Pi's static IP.
- [ ] Document the YouTube/streaming companion tools (§6).

---

## 8. Open questions for the next round

1. **Upstream resolver** — Cloudflare (`1.1.1.1`), Quad9 (`9.9.9.9`, malware-filtering),
   or other? Affects privacy/speed/filtering posture.
2. **Block response** — `NXDOMAIN` (cleaner) vs `0.0.0.0` (faster failure on some clients)?
3. **YouTube tools** — which device-level companions from §6 do you want documented/set up?
4. **Dashboard** — in scope for v1, or DNS-core only first?
5. **Cross-compile vs build-on-Pi** — do you have a faster dev machine to cross-compile ARM binaries, or build directly on the Pi?

---

## 9. Sources

- [Does Pi-hole Block YouTube Ads — MiniTool](https://youtubedownload.minitool.com/news/does-pihole-block-youtube-ads.html)
- [YouTube ads getting through Pi-hole — Pi-hole Discourse](https://discourse.pi-hole.net/t/youtube-ads-getting-through-pihole-any-advances-in-100-blocking-without-also-blocking-youtube-videos/60951)
- [How to Watch YouTube Without Ads — Top10VPN](https://www.top10vpn.com/guides/how-to-block-youtube-ads/)
- [Watch YouTube Without Ads in 2026 — AdBlock Tester](https://adblock-tester.com/ad-blockers/youtube/how-to-watch-without-ads/)
- [How to Block YouTube Ads: 7 Methods (2026) — Blockify](https://getblockify.com/blog/how-to-block-youtube-ads/)
- [YouTube testing server-side ad injection — AlternativeTo](https://alternativeto.net/news/2024/6/youtube-is-testing-server-side-ad-injection-to-counteract-ad-blockers)
- [Hickory DNS (GitHub)](https://github.com/hickory-dns/hickory-dns)
- [hickory-resolver docs](https://docs.rs/hickory-resolver/latest/hickory_resolver/)
- [HaGeZi DNS Blocklists (GitHub)](https://github.com/hagezi/dns-blocklists)
- [OISD domain blocklist](https://oisd.nl/)
- [Pi-hole vs AdGuard Home 2026 — Privacy Smart Home](https://www.privacysmarthome.com/guides/pi-hole-vs-adguard-home-best-dns-filter-for-iot-2026/)

# vido.chat — Anonymous 1-to-1 Random Video Chat

A live, Omegle-style random video chat platform. Two strangers are matched in real time and connected over WebRTC, with a self-hosted TURN relay so calls succeed even behind strict NATs and corporate firewalls.

🔗 **Live:** [vido.chat](https://vido.chat)

> Source code is kept private because this is a running production service. This repo documents how it's built. Visit the live site to try it, or reach out for a walkthrough.

---

## What it does

- Click **Start** and you're matched with another waiting user within seconds.
- Video and audio flow peer-to-peer; the server only handles signalling and matchmaking.
- Skip to the next person at any time; report abusive users.
- Works on desktop and mobile browsers with no install and no account.
- An admin panel controls branding, moderation, and platform settings.

## Architecture

```
Browser A  ──┐                                    ┌──  Browser B
             │  Socket.IO (signalling, matchmaking) │
             └────────────►  Node.js / Express  ◄────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
                 Redis          MariaDB        Coturn
             (match queue)   (persistence)   (TURN relay)

Media: Browser A ◄══ WebRTC (P2P) ══► Browser B
       falls back to TURN relay when direct connection fails
```

**Flow of a call**

1. User joins → server pushes them onto a Redis queue.
2. Two waiting users are popped and paired; each is told who they're talking to.
3. Browsers exchange SDP offer/answer and ICE candidates through Socket.IO.
4. WebRTC connects directly. If NAT traversal fails, media relays through the TURN server (short-lived credentials issued per session).
5. Either side can skip → both return to the queue.

## Key engineering decisions

- **Self-hosted TURN (Coturn)** — without a relay, a meaningful share of real users behind symmetric NAT see a black screen. Verified by forcing relay-only mode and confirming calls still connect.
- **Redis for matchmaking** — in-memory queue makes pairing near-instant and survives server restarts without losing waiting users.
- **Client-first deploy order** — the client must be rebuilt before the server ships a signalling change, otherwise old clients silently fail to negotiate. This is documented as a hard rule in the deploy process.
- **Isolation on a shared VPS** — runs under PM2 on a loopback port behind an Apache reverse proxy, co-hosted with a separate production PHP app with zero interference between them.
- **reCAPTCHA-gated admin login** with a self-test gate so a misconfigured key can never lock the admin out.
- **One-click self-updater** — the admin panel can pull and deploy the latest release without SSH.
- **Web installer** — first-run setup wizard creates the database, admin account, and config.

## Tech stack

| Layer | Technology |
|---|---|
| Client | React (Vite), WebRTC APIs |
| Server | Node.js, Express, Socket.IO |
| Data | MariaDB (persistence), Redis (queue) |
| Media relay | Coturn (TURN/STUN) |
| Infra | Linux VPS, Apache reverse proxy, PM2, HTTPS |

## How it was built

Built end-to-end using **Claude Code** as the primary development environment — spec-driven: describe the feature, let the AI draft it, test it against real browsers and real network conditions, fix what breaks, ship. Every feature was verified with real human sessions before release.

The parts that needed a human were the ones AI can't see: production environment facts (Apache not Nginx, non-default DB ports), deploy ordering, and knowing that a "green" test run can hide silently-skipped suites.

## Author

**Kotha Nishanth Reddy** — [LinkedIn](https://linkedin.com/in/k-nishanth-reddy) · [GitHub](https://github.com/bbhavanar)

Other live projects: [cpmbid.com](https://cpmbid.com) (ad network) · [pressgt.com](https://pressgt.com) (news publishing platform)

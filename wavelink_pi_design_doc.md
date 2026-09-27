# WaveLink Pi Desk Dashboard Design Document

## Overview

This document defines the target architecture for a Raspberry Pi-based full-colour Waveshare e-paper desk dashboard that renders schedule, deadlines, scores, and modeled academic metrics on an e-ink panel while exposing a private control portal over Tailscale for remote management and UI editing. The intended operating model is a low-refresh appliance: the Pi fetches data, normalizes it into a local state model, renders a screen image matched to the panel’s capabilities, updates the display, and sleeps between refreshes to respect Waveshare’s e-paper guidance on long refresh intervals and safe operation.[cite:1]

The system is designed around modular adapters so that Canvas can be used first, Google Calendar can be added later, and any future OnTrack or custom data source can be plugged into the same internal schema without rewriting the renderer or control portal. Canvas provides documented APIs for enrollments, grades, assignments, and calendar events, making it the best initial integration point.[cite:50][cite:16]

## Product Goals

The product should behave like an ambient academic status board rather than a conventional interactive display. The e-paper panel has a long full-refresh cycle, so the design must prioritize stable, high-value information over rapid interaction.[cite:1]

Primary goals:
- Show tomorrow’s classes, current and upcoming due work, current module scores, and modeled academic standing.
- Support accurate low-frequency display updates on a Raspberry Pi 4B.
- Allow secure remote management over the owner’s Tailscale tailnet.
- Expose a private browser-based control portal available only inside the tailnet.
- Allow layout editing, UI code editing, visual drag/resize placement, and preview against the real display palette.
- Keep data ingestion modular so APIs and scrapers can be added later without changing core rendering logic.

Secondary goals:
- Support Google Calendar integration if useful and authorized through Google’s Calendar API quickstart workflow.[cite:122]
- Support future alternate sources such as OnTrack exports, institution feeds, or custom scraping connectors.
- Support deterministic rendering and background jobs that can recover after reboot.

## Non-Goals

The first release should not try to turn the e-paper into a high-frame-rate live dashboard, multi-user SaaS, or public web application. Tailscale Serve is a better fit than Tailscale Funnel because Serve limits access to tailnet members, whereas Funnel is intended for exposing services to the broader internet.[cite:118][cite:115]

The first release should also avoid a dependency on any undocumented OnTrack API until verified documentation or stable export formats are available.

## Core User Experience

The product has two surfaces:

1. **Display surface** — the e-paper screen itself, optimized for glanceability.
2. **Control surface** — a tailnet-only web portal hosted on the Pi.

The display surface should answer these questions in one glance:
- What classes are tomorrow?
- What is currently due, overdue, or due soon?
- What are the current scores per module?
- What is the estimated percentile or modeled rank band?
- Are there any urgent reminders?

The control surface should support:
- Login restricted to the owner’s Tailscale identity and device access policy.[cite:118][cite:123]
- Enabling or disabling data sources.
- Entering API credentials and refresh rules.
- Editing display code or templated layout logic.
- Visually dragging, resizing, recoloring, and toggling modules on a canvas sized to the real panel.
- Previewing the exact panel resolution and approximate display palette.
- Triggering manual refreshes and reviewing logs.

## Architecture Principles

### Separation of concerns

The architecture should be split into independent layers:
- **Adapters** acquire data from external systems.
- **Normalization** converts external data into internal typed models.
- **State engine** merges normalized data into a single canonical state.
- **Layout engine** determines visual placement and styling.
- **Renderer** converts the layout into a panel-accurate image.
- **Display driver** sends the final image to the Waveshare panel.
- **Portal backend** manages configuration, editing, preview, and jobs.

This separation ensures APIs can be swapped or extended without breaking the display stack.

### Offline-first state

The display should never depend on a live API call at draw time. Background jobs should fetch data, transform it, and store a local snapshot. The renderer should consume only local state so the screen can still refresh from cached data during internet outages.

### Deterministic rendering

Given a specific `state.json` plus `layout.json`, the same image should always be produced. Determinism makes preview accurate, debugging easier, and future versioning possible.

### Plugin-friendly ingestion

Every source should implement a shared adapter interface so that adding Google Calendar later is a matter of dropping in a new connector rather than rewriting the whole project. Google provides an official Calendar API quickstart and Python sample flow that makes it a realistic second integration.[cite:122][cite:116]

## System Components

### 1. Device runtime

The device runtime runs on Raspberry Pi OS Lite and hosts the background workers, display renderer, local data store, and web control portal. Raspberry Pi OS Lite is the appropriate lightweight OS for headless management on a Pi 4, while Tailscale can be installed on Linux via the official install path and brought online with `tailscale up`.[cite:117]

Responsibilities:
- Boot services on startup.
- Maintain network connectivity.
- Join the owner’s Tailscale tailnet.
- Run scheduled fetch and render jobs.
- Serve the private control portal.
- Manage secrets and local storage.

### 2. Tailscale access layer

The Pi should join the owner’s Tailscale account during provisioning. The control portal should be exposed using **Tailscale Serve**, which routes tailnet traffic to a local web service and keeps it accessible only to other devices in the tailnet.[cite:118]

Recommended policy:
- Use Tailscale Serve, not Funnel, for the control portal.[cite:118][cite:115]
- Restrict access to the owner’s user identity and optionally specific devices via tailnet ACL policy.[cite:123]
- Use MagicDNS so the portal is reachable via a stable internal HTTPS URL when supported by the tailnet configuration.[cite:119]

### 3. Control portal

The portal is a small single-user web application hosted locally on the Pi and reachable only over Tailscale. It should be intentionally simple and optimized for one operator rather than generalized account management.

Key features:
- Single-admin account model.
- Local configuration editor.
- API source configuration pages.
- Canvas-style visual layout editor with snap grid.
- Accurate preview canvas at panel resolution.
- Theme/palette simulation for the real e-paper display.
- Code editor for advanced UI logic.
- Manual “fetch now,” “render now,” and “push to display now” buttons.
- Service health, logs, and last-refresh metadata.

### 4. Adapter framework

The adapter framework is the boundary to external systems. Every adapter should implement a uniform contract such as:

```python
class DataAdapter:
    id: str
    def fetch(self, context) -> RawPayload: ...
    def normalize(self, raw: RawPayload) -> NormalizedRecords: ...
    def healthcheck(self) -> HealthStatus: ...
```

Planned adapters:
- `canvas_adapter`
- `google_calendar_adapter`
- `http_json_adapter`
- `html_scrape_adapter`
- `manual_entry_adapter`
- `ontrack_adapter` (only after validation)

### 5. State engine

The state engine merges all normalized records into a canonical snapshot. This should become the only source read by the renderer and layout preview.

Suggested top-level state domains:
- `profile`
- `calendar_events`
- `assignments`
- `courses`
- `scores`
- `predictions`
- `alerts`
- `system_status`
- `layout_meta`

### 6. Rendering pipeline

The rendering pipeline should consume canonical state plus a layout definition and output a final framebuffer image at the exact panel resolution. Because the Waveshare full-colour e-paper has long refresh characteristics and should not be updated too frequently, rendering should happen in the background and push only when content changes materially or on approved intervals.[cite:1]

Rendering substeps:
1. Load `state.json`.
2. Load `layout.json`.
3. Resolve data bindings.
4. Apply panel palette constraints.
5. Compose widgets into a framebuffer.
6. Generate preview PNG for the portal.
7. Generate device-ready image for the Waveshare driver.
8. Push update if allowed by refresh policy.

## Data Model

A canonical internal schema keeps the rest of the product stable even if APIs change.

### Event model

```json
{
  "id": "event_123",
  "source": "canvas",
  "title": "Computer Science Lecture",
  "start": "2026-09-28T09:00:00+01:00",
  "end": "2026-09-28T10:30:00+01:00",
  "location": "Room A",
  "kind": "class",
  "status": "confirmed",
  "meta": {}
}
```

### Assignment model

```json
{
  "id": "assign_456",
  "source": "canvas",
  "course_id": "course_cs101",
  "title": "Problem Set 2",
  "due_at": "2026-09-28T23:59:00+01:00",
  "points_possible": 100,
  "score": 87,
  "state": "submitted",
  "category": "coursework",
  "meta": {}
}
```

### Score model

```json
{
  "course_id": "course_cs101",
  "source": "canvas",
  "current_score": 82.4,
  "final_score": null,
  "current_grade": "A-",
  "class_size": 40,
  "known_mean": 71.0,
  "known_sd": 8.5,
  "weights": [
    {"name": "Coursework", "weight": 0.4},
    {"name": "Exam", "weight": 0.6}
  ]
}
```

### Prediction model

```json
{
  "course_id": "course_cs101",
  "expected_final_score": 81.7,
  "estimated_percentile": 0.91,
  "estimated_rank": 4,
  "rank_band_low": 2,
  "rank_band_high": 6,
  "confidence": "medium",
  "method": "weighted_monte_carlo"
}
```

## Background Processes

The system should be explicit about all background jobs so behavior is predictable and debuggable.

### Required background jobs

| Job | Purpose | Suggested cadence | Notes |
|---|---|---:|---|
| Connectivity watcher | Confirm Wi-Fi, internet, and Tailscale status | every 60s | Update system status widget |
| Adapter scheduler | Launch source fetches | every 5–30 min | Per-source cadence |
| Data normalizer | Convert raw payloads to canonical records | after each fetch | Deterministic transform |
| State merger | Build `state.json` snapshot | after normalization | Atomic write |
| Prediction engine | Recompute modeled metrics | after score change | Cache results |
| Preview renderer | Produce portal preview PNG | on state/layout change | Fast software render |
| Display render job | Create panel-ready image | on approved update | Full fidelity |
| Display push job | Push to e-paper | minimum policy interval | Respect Waveshare precautions[cite:1] |
| Log rotation | Keep storage healthy | daily | Compress or prune logs |
| Secret verifier | Test API credentials | daily / on save | Report failures |
| Backup export | Snapshot config + layout | daily | Local or synced export |

### Trigger model

Two trigger types should exist:
- **Scheduled triggers** for regular refresh.
- **Event triggers** for changes in config, layout, or source data.

Display pushes should be gated by both:
- minimum refresh interval,
- content-diff threshold,
- manual override flag.

## API Strategy

### Canvas first

Canvas is the best first-class adapter because it has public documentation for enrollments, courses, assignments, and calendar events.[cite:50][cite:16][cite:60]

Priority Canvas data:
- upcoming events,
- tomorrow’s classes,
- assignments due now/soon,
- enrollment scores and grades.

### Google Calendar second

Google Calendar is a sensible second integration because it can supplement personal reminders, meetings, and external events not present in Canvas. Google publishes a Calendar API quickstart and Python sample flow, which makes integration feasible once OAuth credentials are configured.[cite:122][cite:116]

Best use cases:
- personal reminders,
- travel or life calendar overlays,
- custom manual events,
- calendar-based suppression or priority rules.

### Generic HTTP data functions

To make future integrations easy, the system should support a light declarative data function layer. A target syntax could be:

```yaml
functions:
  get_courses:
    use: canvas.enrollments
  get_personal_events:
    use: google_calendar.events
  get_data_from_url:
    use: http_json
    config:
      url: "https://example.com/api"
```

The operator intent is not to hardcode every API connector by hand, but to define simple “get data” functions and let the adapter layer fetch, parse, cache, and expose uniform records. When raw HTML is the only source, an `html_scrape_adapter` can fetch a page and extract values with CSS selectors or parser rules.

## Visual Editor Design

The control portal should include a visual editor tailored to the fixed e-paper canvas.

### Editor requirements

- Real panel resolution canvas.
- Palette-aware preview matching the Waveshare color limitations as closely as possible.[cite:1]
- Drag to move widgets.
- Resize handles.
- Alignment guides and optional grid snapping.
- Layer ordering.
- Safe zones and margin visualization.
- Property inspector for text size, colors, bindings, and visibility conditions.
- Live sample data mode.
- Real data preview mode.
- Version history for layout definitions.

### Rendering modes

The editor should provide three preview modes:
- **Design mode** — clean canvas with boxes and labels.
- **Approximate panel mode** — dithered/palette-limited preview simulating device output.
- **Actual render mode** — same renderer used for production, generating a final preview image.

## Code Editing Model

There should be two supported customization paths:

1. **No-code / low-code layout editing** through widget placement and property panels.
2. **Advanced code editing** through template functions and UI modules.

Recommended rule:
- Layout and basic bindings live in JSON/YAML.
- Advanced presentation logic lives in a sandboxed Python or template layer.
- Data adapters remain a separate backend concern.

This keeps the product approachable while still allowing direct control over the display code.

## Storage Layout

Suggested on-disk structure:

```text
/opt/wavelink/
  app/
    backend/
    frontend/
    adapters/
    renderer/
    workers/
  config/
    app.yaml
    adapters.yaml
    tailscale.yaml
  data/
    raw/
    normalized/
    state.json
    previews/
    backups/
  secrets/
    canvas.env
    google.env
  logs/
  layouts/
    active.layout.json
    history/
```

## Security Model

The portal should not be exposed publicly. Tailscale Serve is the preferred access mechanism because it shares a local service only with devices in the tailnet, unlike Funnel, which intentionally exposes services to the broader internet.[cite:118][cite:115]

Security rules:
- Require device enrollment in the owner’s tailnet.
- Restrict access with ACLs to the owner identity.[cite:123]
- Keep a local app password only as a second factor, not the primary trust boundary.
- Store API secrets outside the editable UI code area.
- Log login attempts and config changes.
- Disable any unauthenticated internet exposure by default.

## Recommended Stack

### Backend
- Python 3
- FastAPI for portal/backend API
- APScheduler or systemd timers for jobs
- Pillow + panel palette utilities for rendering
- SQLite for metadata and history
- JSON files for state snapshots

### Frontend
- Small React or Svelte app for the editor
- A fixed canvas workspace matching device resolution
- Monaco editor for code editing
- Lightweight drag/resize system for widgets

### System services
- `tailscaled`
- `wavelink-backend.service`
- `wavelink-worker.service`
- `wavelink-render.service` or integrated worker

## Deployment Flow

1. Flash Raspberry Pi OS Lite on the Pi.[cite:117]
2. Install Tailscale and join the owner’s tailnet using `tailscale up`.[cite:117]
3. Enable Tailscale Serve for the local portal service.[cite:118]
4. Install the WaveLink application services.
5. Configure Canvas credentials first.
6. Optionally add Google Calendar credentials later.[cite:122]
7. Validate preview output.
8. Run Waveshare display test and then production render pipeline.[cite:1]

## Phased Roadmap

### Phase 1: Device foundation
- Raspberry Pi OS Lite installation.
- Tailscale enrollment.
- Private portal served through Tailscale Serve.
- Waveshare display driver validation.[cite:117][cite:118][cite:1]

### Phase 2: Canvas MVP
- Courses, assignments, and calendar ingestion.[cite:50][cite:16]
- Canonical state model.
- Static layout rendering.
- Manual refresh controls.

### Phase 3: Editor and preview
- Drag/resize visual editor.
- Palette-aware preview.
- Layout save/versioning.
- Basic code editing.

### Phase 4: Prediction layer
- Weighted grade model.
- Rank band estimation.
- Confidence scoring.
- Display widget bindings for modeled metrics.

### Phase 5: Additional adapters
- Google Calendar integration.[cite:122]
- Generic HTTP/JSON adapter.
- Generic HTML scrape adapter.
- OnTrack adapter only if documentation or stable extraction path is confirmed.

## Final Product Statement

The final product should be articulated as a private, remotely managed e-paper appliance: a Waveshare-powered Raspberry Pi desk dashboard joined to the owner’s Tailscale account, exposing a tailnet-only control portal that allows secure configuration, UI editing, visual layout authoring, previewing against the actual panel constraints, and modular connection of current and future data sources. Tailscale Serve is the preferred method for hosting the portal because it keeps the application private to the tailnet, while Canvas should serve as the first production data source and Google Calendar should be designed as the next optional plug-in integration.[cite:118][cite:50][cite:122]

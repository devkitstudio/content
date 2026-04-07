---
title: New Features
order: 1
icon: sparkles
---

### System Monitor Dashboard

- Real-time server metrics at `/monitor`
- Memory, CPU, heap, event loop tracking with sparkline charts
- Progress ring gauges for system resources
- V8 internals panel (native/detached contexts, heap limit)
- Active handles & HTTP request monitoring
- Live log stream with color-coded levels
- Click any chart for detailed stats + documentation

### Metrics Integration

- Powered by `@ecosy/core` Syhemo engine
- State management with `@ecosy/store`
- 5-second snapshot interval, up to 1 hour history

### UI Fixes

- Changelog page now supports light/dark theme
- Changelog URL hash deep linking (`/changelog#version/section`)
- Monitor page with back-to-home navigation
- Replaced Material Symbols with Lucide icons
- Theme switcher available on all pages (Home, Changelog, Monitor)

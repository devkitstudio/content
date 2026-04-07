---
title: New Features
order: 1
icon: sparkles
---

### Tool Deep Links

- Each tool on homepage links directly to `/app#<toolId>`
- Sidebar navigation updates URL hash when selecting tools
- Opening `/app#json-formatter` auto-activates the matching tool

### Homepage Tool Showcase

- Tools reorganized into categorized groups (Data, Image, String, Date, JSON, CSV, SEO)
- Staggered animations and refined group title styling

### DB Diagram

- Zoomable & pannable canvas with n8n-style dot grid background
- Draggable table nodes with colored headers and inline editing
- Inline rename: click edit icon → title becomes input field
- Settings popover for changing table header color
- Inline column editing with name input, custom type dropdown popover
- Add/remove columns directly on the card
- PK/FK indicator toggle per column
- Relationship lines with smart auto-routing (SVG bezier curves)
- Lines auto-switch sides when tables are dragged across each other
- Stub lines extend straight before curving for clean paths
- Connection dots at table edges for source and target
- Auto-expanding canvas based on table positions
- Add Table dialog for creating new tables

### UI Improvements

- Changelog title text clipping fix (`line-height` adjustment)

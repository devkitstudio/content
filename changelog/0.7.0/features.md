---
title: "New Features"
icon: "sparkles"
order: 1
---

> [!TIP]
> The DevKit Studio toolkit just got significantly more powerful! We've introduced a state-of-the-art native 3D Model Explorer and a highly secure, client-side RAR to ZIP Converter. All designed with our unified VSCode-like UI.

### What's New?

#### 📦 Client-Side RAR to ZIP Converter

- A fully secure, 100% offline file converter that ensures your data never leaves your browser.
- **VSCode-Like Split Pane UI:** Drop your `.rar` files on the left and monitor the live Terminal Output console on the right.
- **Encryption Support & Tree Preview:** Automatically detects password-protected archives, prompts for a password, and generates a visual **Folder Tree Preview** so you can inspect the contents before securely packaging and auto-downloading them directly as a `.zip`.

#### 🧊 3D Models Explorer Refinements

- **Material Preservation:** We fixed the rendering engine so 3D models no longer default to white, beautifully preserving the original textures and material colors of your `.gltf` and `.glb` files.
- **Enhanced Camera Controls:** Introduced a new `toolMode` toggle that lets you seamlessly switch between Orbit (rotation) and Pan (movement) tools using the left mouse button, complete with helpful toolbar tooltips.
- **Properties Pane:** Added a "Reset to Original" capability so you can easily revert colored meshes back to their default texture states.

#### 📝 GitHub Alerts Integration
- **Native Markdown Callouts:** The ReactMarkdown engine powering this changelog and other documentation pages has been upgraded with a bespoke AST parser. It now natively intercepts and renders GitHub-flavored Markdown Callouts (`[!TIP]`, `[!NOTE]`, `[!WARNING]`, etc.) directly into beautiful, Tailwind-styled UI blocks with corresponding Lucide icons!

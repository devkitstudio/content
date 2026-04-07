# 🚀 DevKit Studio Content Engine

Welcome to the open-source **"Docs-as-Code" Headless Content Repository** for [DevKit Studio](https://devkitstudio.com).

DevKit Studio is a premium, unified ecosystem for modern software engineers. While our core platform engine is proprietary, we believe that technical knowledge, algorithms, and engineering practices belong to the community. 

This repository drives our entire knowledge base using a **Decoupled Markdown-first Architecture**.

## 🏗️ Architecture: Headless & Edge Delivery

This repository is **NOT** bundled at build-time. We use a headless content strategy:

1. **Markdown-First**: All data (Algorithms, Cheatsheets, System Design Facts, Changelogs) is written purely in Markdown and YAML Frontmatter.
2. **Manifest Indexing**: Modules use an `index.md` manifest to declaratively map visual UI boundaries, sort weight, and categorical metadata.
3. **jsDelivr Edge CDN**: At runtime, the DevKit Studio Next.js application dynamically fetches raw markdown from this repository directly through the [jsDelivr CDN](https://www.jsdelivr.com/).

This completely decouples our UI deployment lifecycle from our content strategy. A typo fix or a new algorithm added here instantly propagates to production with zero build time.

## 📂 Repository Structure

Our knowledge base is structured intelligently to mirror the application routing:

- `/changelog` - Version release histories and bug fix logs.
- `/studio/cheatsheet` - Quick-reference syntax and usage guides for various DevKit Studio Tools (e.g., Regex, Hashes).
- `/studio/algorithms` - In-depth breakdown of DSA concepts.
- `/studio/practices` - Clean Architecture, System Design patterns, and Engineering best practices.
- `/facts` - Bite-sized engineering trivia and computing fundamentals.

## 🤝 How to Contribute

Because we employ a Docs-as-Code architecture, contributing is as simple as sending a Pull Request to a text file!

1. **Fixing Typos:** Just edit the Markdown file directly on GitHub and submit a PR!
2. **Adding Cheatsheets:** 
   - Add a new `.md` file to the respective tool folder inside `/studio/cheatsheet/`.
   - Open that folder's `index.md` manifest and map your new file in the YAML `sections` array.
3. **Requesting Tools:** If you have an idea for a brand new application tool, head over to the [Issues tab](https://github.com/devkitstudio/content/issues) and submit a Feature Request.

---
*Programming is the art of architectural thinking. Thank you for helping us curate the ultimate technical library for senior engineers!* ❤️

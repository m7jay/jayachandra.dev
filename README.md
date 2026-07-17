# jayachandra.dev

My personal portfolio website built with Hugo/BlowFish to showcase projects, experience, and thoughts.

## Tech Stack

- **Static Site Generator:** [Hugo](https://gohugo.io/)
- **Theme/Template:** [Blowfish](https://blowfish.page/)
- **Language:** HTML5, CSS3 (SCSS), JavaScript
- **Hosting/Deployment Platform(s):**
  - GitHub Actions
  - Firebase App

## How to Run Locally

1. Prerequisites: Make sure you have [Hugo](https://gohugo.io/) installed on your machine (`brew install hugo` for macOS or download from the official site).
2. Clone this repository into a local directory (if not already cloned)
3. Run `hugo -D -E -F server`
4. Clean `hugo --gc --cleanDestinationDir`
5. Style changes `hugo --gc --minify`

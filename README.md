# Voronoify

A tiny, client-side web app (ideal for GitHub Pages) that lets you generate and export a Voronoi diagram over a provided image.

This repository now ships with a minimal **Vite + Vue 3** scaffold (with **Tailwind CSS** wired through Vite) so you can
iterate locally and publish the generated static files straight to GitHub Pages.

## Local development

1. Install dependencies (requires Node 18+):
   ```bash
   npm install
   ```
2. Start the dev server:
   ```bash
   npm run dev
   ```
3. Build the static site:
   ```bash
   npm run build
   ```

> Note: the dev dependencies are not committed. If you are offline or behind a restricted network you may need to configure
> your npm registry to download packages before running the commands above.

## Deploying to GitHub Pages

1. The Vite config sets `base: '/voronoify/'` so assets resolve correctly when hosted at `https://<user>.github.io/voronoify/`.
2. Build the site with `npm run build`. The output is written to `dist/`.
3. Choose one of the following:
   - **Pages from the `dist` folder (no CI):** push the contents of `dist/` to a branch and configure Pages to serve from that
     folder (e.g., via the `gh-pages` branch).
   - **GitHub Actions:** use the official [`peaceiris/actions-gh-pages`](https://github.com/peaceiris/actions-gh-pages) action or
     the [Vite static deploy guide](https://vitejs.dev/guide/static-deploy.html#github-pages) to build and publish automatically
     on push.

Once configured, your Hello World page will be live and ready for the upcoming Voronoi editor.

## Styling

- Tailwind is enabled via `@tailwindcss/vite` in `vite.config.js`.
- The global stylesheet lives at `src/assets/app.css` and imports Tailwind with `@import "tailwindcss";` (add your custom
  styles below that line).

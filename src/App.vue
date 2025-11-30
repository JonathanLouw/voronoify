<template>
  <main>
    <header class="hero">
      <p class="eyebrow">Vite + Vue 3 starter</p>
      <h1>Voronoify</h1>
      <p class="lede">
        A minimal scaffold for the future Voronoi-over-image playground. Replace the hero image, wire up the controls, and ship
        it straight to GitHub Pages.
      </p>
      <div class="cta-row">
        <a class="button primary" href="https://vitejs.dev/guide/static-deploy.html#github-pages" target="_blank" rel="noreferrer">
          GitHub Pages guide
        </a>
        <a class="button ghost" href="https://vuejs.org/guide/essentials/application.html" target="_blank" rel="noreferrer">
          Vue 3 docs
        </a>
      </div>
    </header>

    <section class="preview">
      <div class="canvas-shell">
        <div class="bg"></div>
        <div class="overlay">
          <p class="pill">Voronoi layer</p>
          <p class="hint">Drop your points, render cells, and export as an image.</p>
        </div>
      </div>
      <div class="copy">
        <h2>Hello, world! 👋</h2>
        <p>
          This starter keeps everything client-side so it can be served directly from <code>dist/</code>. Run
          <code>npm run build</code>, push the built files, and point GitHub Pages at the <code>dist</code> folder (or wire up
          Actions) to publish.
        </p>
        <p>
          Next steps: hook up image uploads, generate Voronoi sites, style the cells, and add an export button. The
          <code>base</code> option in <code>vite.config.js</code> is pre-set for the <code>voronoify</code> repository name so that
          asset URLs resolve correctly on GitHub Pages.
        </p>
      </div>
    </section>
  </main>
</template>

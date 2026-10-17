# AbyssWM.github.io

Source of the Abyss documentation website, served at <https://abysswm.github.io>.

Built with [VitePress](https://vitepress.dev) and a custom dark "Katana" theme.

## Develop

```sh
npm install
npm run docs:dev       # live preview on http://localhost:5173
npm run docs:build     # static build into docs/.vitepress/dist
npm run docs:preview   # serve the production build
```

## Structure

```
docs/
  index.md                 home page
  wiki/                    Wiki: index, getting-started, configuration, abyssctl, ecosystem
  roadmap.md               what is planned
  credits.md               credits and license
  .vitepress/config.mts    site config, navigation
  .vitepress/theme/        Katana theme (index.ts, custom.css)
  public/                  logo and favicon
```

## Deploy

Every push to `main` is built and published by `.github/workflows/deploy.yml`.
In the repository settings, set **Pages > Source** to **GitHub Actions** once.

## Custom domain (later)

The site is served at the root (`base: '/'`). If a custom domain is added later, put it in a
`docs/public/CNAME` file (one line, e.g. `docs.example.org`). There is deliberately no CNAME file yet.

## License

This website and its original documentation are licensed under BSD-3-Clause;
see [LICENSE](LICENSE). Third-party dependencies retain their own licenses.
Licenses for the compositor, libraries and package recipes are documented
separately in [Credits](docs/credits.md).

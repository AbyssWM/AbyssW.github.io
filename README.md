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

The documentation text is available under the BSD 3-Clause License, like the Abyss projects.
Abyss is not affiliated with or endorsed by hyprwm.

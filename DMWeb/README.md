# Dark Matter Consulting — Web Site

Blazor WebAssembly static site for [darkmatter.consulting](https://darkmatter.consulting), hosted on GitHub Pages.

## Prerequisites

- [.NET 10 SDK](https://dotnet.microsoft.com/download)

## Local Development

```bash
dotnet run
```

Browse to `http://localhost:5000` (or the port shown in terminal).

## Production Build

```bash
dotnet publish DMWeb.csproj -c Release -o release
```

Output is in `release/wwwroot/` — a fully static folder, no server required.

## Deployment

Push to `master` — GitHub Actions builds and deploys to the `gh-pages` branch automatically.

## Post-Deployment Configuration

### Formspree (contact form)
1. Create a free account at [formspree.io](https://formspree.io)
2. Create a form pointed at `info@darkmatter.consulting`
3. Replace `YOUR_FORMSPREE_ID` in `Components/ContactSection.razor` with your form ID

### Custom Domain
To serve from `darkmatter.consulting` instead of the GitHub Pages subdomain:
1. Add `wwwroot/CNAME` containing: `darkmatter.consulting`
2. In `wwwroot/index.html` change: `<base href="/dark-matter-static-web/" />` → `<base href="/" />`
3. In `wwwroot/404.html` change: `var pathSegmentsToKeep = 1;` → `var pathSegmentsToKeep = 0;`
4. Configure DNS: CNAME `www` → `darkmatteroregon.github.io`, A records for apex per GitHub docs

## License

MIT

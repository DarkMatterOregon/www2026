# Dark Matter Consulting — Blazor WASM Static Site Build Prompt

## Mission

Replace the existing Gatsby site in the GitHub repo `DarkMatterOregon/dark-matter-static-web` with a new Blazor WebAssembly standalone application. The result must be a fully static build deployable to GitHub Pages at `https://darkmatteroregon.github.io/dark-matter-static-web/`.

The site is a single-page marketing site for Dark Matter Consulting — a Eugene, Oregon-based team of senior .NET engineers specializing in modernizing legacy applications for small government agencies and nonprofits.

---

## Prerequisites / Environment Notes

- Requires **.NET 10 SDK** (`dotnet --version` should show `10.x.x`)
- If not installed: `winget install Microsoft.DotNet.SDK.10` or download from https://dotnet.microsoft.com/download
- The template name for standalone Blazor WASM in .NET 10 is `blazorwasm`
- .NET 10 WASM-specific notes:
  - `blazor.boot.json` no longer exists as a separate file — boot config is inlined into `dotnet.js`. Do not reference or look for this file.
  - Environment is controlled via `<WasmApplicationEnvironmentName>` in the `.csproj`, not `launchSettings.json`
  - Response streaming is enabled by default. For simple `HttpClient` POST to Formspree this is fine.
  - Asset fingerprinting is available via `<OverrideHtmlAssetPlaceholders>true</OverrideHtmlAssetPlaceholders>` — include this in the `.csproj`
  - Hot Reload is enabled by default for Debug builds — no configuration needed

---

## Step 1 — Scaffold the Project

```bash
# From the repo root, delete old Gatsby files first (preserve LICENSE)
# Then scaffold:
dotnet new blazorwasm --name DarkMatterWeb --output DarkMatterWeb --no-https --empty
```

Use the `--empty` flag to get a minimal scaffold without the weather forecast demo code, Bootstrap, or unnecessary sample files.

Then delete any remaining sample/demo files the template may have generated:
- `Pages/Weather.razor` (if present)
- `WeatherForecast.cs` (if present)  
- `wwwroot/sample-data/` (if present)

---

## Step 2 — Project File

`DarkMatterWeb/DarkMatterWeb.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk.BlazorWebAssembly">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <OverrideHtmlAssetPlaceholders>true</OverrideHtmlAssetPlaceholders>
  </PropertyGroup>
</Project>
```

---

## Step 3 — Program.cs

```csharp
using Microsoft.AspNetCore.Components.Web;
using Microsoft.AspNetCore.Components.WebAssembly.Hosting;
using DarkMatterWeb;

var builder = WebAssemblyHostBuilder.CreateDefault(args);
builder.RootComponents.Add<App>("#app");
builder.RootComponents.Add<HeadOutlet>("head::after");

builder.Services.AddScoped(sp => new HttpClient());

await builder.Build().RunAsync();
```

Note: `BaseAddress` is intentionally not set on `HttpClient` — the contact form POSTs to an absolute Formspree URL, not a relative API.

---

## Step 4 — wwwroot/index.html

This file handles the GitHub Pages SPA routing fix inline. Replace the entire file:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Dark Matter Consulting</title>
  <base href="/dark-matter-static-web/" />
  <link rel="stylesheet" href="css/app.css" />

  <!-- GitHub Pages SPA redirect handler -->
  <script>
    (function(l) {
      if (l.search[1] === '/') {
        var decoded = l.search.slice(1).split('&').map(function(s) {
          return s.replace(/~and~/g, '&');
        });
        window.history.replaceState(null, null,
          l.pathname.slice(0, -1) + decoded[0] +
          (decoded[1] ? '&' + decoded[1] : '') +
          l.hash
        );
      }
    }(window.location))
  </script>
</head>
<body>
  <div id="app">
    <div class="loading-screen">
      <img src="img/logo.png" alt="Dark Matter Consulting" class="loading-logo" />
    </div>
  </div>

  <div id="blazor-error-ui" role="alert">
    An unhandled error has occurred.
    <a href="." class="reload">Reload</a>
    <span class="dismiss">🗙</span>
  </div>

  <script src="_framework/blazor.webassembly.js"></script>
</body>
</html>
```

---

## Step 5 — wwwroot/404.html

This file is how GitHub Pages handles deep-link routing for SPAs. Create it as a sibling to `index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <title>Dark Matter Consulting</title>
  <script>
    // GitHub Pages SPA routing hack
    // Encodes the path into the query string so index.html can restore it
    var pathSegmentsToKeep = 1; // 1 = hosted under /repo-name/; change to 0 for custom domain at root
    var l = window.location;
    l.replace(
      l.protocol + '//' + l.hostname + (l.port ? ':' + l.port : '') +
      l.pathname.split('/').slice(0, 1 + pathSegmentsToKeep).join('/') + '/?/' +
      l.pathname.slice(1).split('/').slice(pathSegmentsToKeep).join('/').replace(/&/g, '~and~') +
      (l.search ? '&' + l.search.slice(1).replace(/&/g, '~and~') : '') +
      l.hash
    );
  </script>
</head>
<body></body>
</html>
```

---

## Step 6 — wwwroot Assets

### wwwroot/.nojekyll
Create an empty file — prevents GitHub Pages from running Jekyll on the output:
```
(empty file)
```

### wwwroot/img/
- `logo.png` — the Dark Matter Consulting wordmark (black on transparent). Place the provided logo file here.
- `black_hole_milkyway_header.jpg` — download from:
  `https://raw.githubusercontent.com/DarkMatterOregon/dark-matter-static-web/master/static/img/black_hole_milkyway_header.jpg`

---

## Step 7 — _Imports.razor

```razor
@using System.Net.Http
@using System.Net.Http.Json
@using System.Net.Http.Headers
@using Microsoft.AspNetCore.Components.Forms
@using Microsoft.AspNetCore.Components.Routing
@using Microsoft.AspNetCore.Components.Web
@using Microsoft.AspNetCore.Components.Web.Virtualization
@using Microsoft.AspNetCore.Components.WebAssembly.Http
@using Microsoft.JSInterop
@using DarkMatterWeb
@using DarkMatterWeb.Components
```

---

## Step 8 — App.razor

```razor
<Router AppAssembly="typeof(App).Assembly">
    <Found Context="routeData">
        <RouteView RouteData="routeData" DefaultLayout="typeof(Layout.MainLayout)" />
        <FocusOnNavigate RouteData="routeData" Selector="h1" />
    </Found>
    <NotFound>
        <PageTitle>Not found</PageTitle>
        <LayoutView Layout="typeof(Layout.MainLayout)">
            <p role="alert">Sorry, there's nothing at this address.</p>
        </LayoutView>
    </NotFound>
</Router>
```

---

## Step 9 — Layout/MainLayout.razor

```razor
@inherits LayoutComponentBase

<NavBar />
<main>
    @Body
</main>
<FooterSection />
```

---

## Step 10 — Pages/Home.razor

```razor
@page "/"

<PageTitle>Dark Matter Consulting — .NET Modernization for Government</PageTitle>

<HeroSection />
<ServicesSection />
<AboutSection />
<TestimonialsSection />
<ContactSection />
```

---

## Step 11 — Components

### Components/NavBar.razor

```razor
<header class="site-header">
  <nav class="nav-container">
    <a href="#hero" class="nav-logo">
      <img src="img/logo.png" alt="Dark Matter Consulting" />
    </a>

    <input type="checkbox" id="nav-toggle" class="nav-toggle" />
    <label for="nav-toggle" class="nav-toggle-label" aria-label="Toggle navigation">
      <span></span><span></span><span></span>
    </label>

    <ul class="nav-links">
      <li><a href="#services">Services</a></li>
      <li><a href="#about">About</a></li>
      <li><a href="#testimonials">Testimonials</a></li>
      <li><a href="#contact" class="nav-cta">Let's Talk</a></li>
    </ul>
  </nav>
</header>
```

---

### Components/HeroSection.razor

```razor
<section id="hero" class="hero">
  <div class="hero-overlay"></div>
  <div class="hero-content">
    <img src="img/logo.png" alt="Dark Matter Consulting" class="hero-logo" />
    <h1>20+ Years of .NET.<br />Built for Government.</h1>
    <p class="hero-sub">
      We help county, state, and nonprofit agencies modernize aging applications —
      on budget, on time, and with the tools to maintain them independently.
    </p>
    <a href="#contact" class="btn btn-primary">Let's Talk</a>
  </div>
</section>
```

---

### Components/ServicesSection.razor

```razor
<section id="services" class="section section-light">
  <div class="container">
    <h2>What We Do</h2>
    <div class="services-grid">

      <div class="service-card">
        <h3>AI-Assisted Modernization</h3>
        <p>
          Experienced engineers using modern AI tools to dramatically reduce the cost
          of modernizing legacy systems. We accelerate discovery, documentation, and
          rewriting of old code — in any language, even MS Access.
        </p>
      </div>

      <div class="service-card">
        <h3>Legacy .NET Modernization</h3>
        <p>
          We extract the business logic from aging .NET Framework, VB6, and legacy apps
          and deliver modern, maintainable Blazor WebAssembly applications.
        </p>
      </div>

      <div class="service-card">
        <h3>Azure Cloud Migration</h3>
        <p>
          Move your on-premise .NET applications to Azure — serverless, containerized,
          or hybrid. We right-size infrastructure for government budgets.
        </p>
      </div>

      <div class="service-card">
        <h3>Team Enablement</h3>
        <p>
          We don't just deliver software. We work alongside your agency's team,
          transfer knowledge, and leave you with the skills and tools to own your
          modernized system.
        </p>
      </div>

    </div>
  </div>
</section>
```

---

### Components/AboutSection.razor

```razor
<section id="about" class="section section-dark">
  <div class="container about-grid">
    <div class="about-text">
      <h2>About Us</h2>
      <p>
        Dark Matter Consulting is a Eugene, Oregon-based team of senior .NET engineers
        with 20+ years delivering software for county, state, and nonprofit clients.
        We bring deep government domain knowledge and modern AI tooling together to
        make legacy modernization affordable and sustainable for smaller agencies.
      </p>
      <p>
        We specialize in extracting business logic from legacy systems in any language
        and delivering modern, cloud-ready .NET applications — then handing your team
        the keys.
      </p>
    </div>
    <div class="about-credentials">
      <ul class="credential-list">
        <li>20+ years .NET experience</li>
        <li>County &amp; State government clients</li>
        <li>Nonprofit sector experience</li>
        <li>Oregon-based, remote-capable</li>
        <li>Blazor &amp; Azure specialists</li>
        <li>AI-assisted development tools</li>
      </ul>
    </div>
  </div>
</section>
```

---

### Components/TestimonialsSection.razor

```razor
<section id="testimonials" class="section section-light">
  <div class="container">
    <h2>What Clients Say</h2>
    <div class="testimonials-grid">

      <blockquote class="testimonial">
        <p>
          "Dark Matter Consulting has a broad depth of talent where they can consult
          and manage any project. This is what I have appreciated about them."
        </p>
        <footer>— Laurel King, Owner, OutdoorIndustryJobs.com</footer>
      </blockquote>

      <blockquote class="testimonial">
        <p>
          "Doug, Mark and the Dark Matter team are well versed in Web and Mobile
          application development and their ability to solve complex development
          problems has led to successful project outcomes. Totally recommend."
        </p>
        <footer>— Adam Wendt, CEO, Trifoia</footer>
      </blockquote>

    </div>
  </div>
</section>
```

---

### Components/ContactSection.razor

```razor
<section id="contact" class="section section-dark">
  <div class="container contact-container">
    <h2>Get In Touch</h2>
    <p class="contact-intro">
      Ready to modernize a legacy system? We'd love to hear about your project.
    </p>

    @if (statusMessage != null)
    {
      <div class="alert @(isSuccess ? "alert-success" : "alert-error")" role="alert">
        @statusMessage
      </div>
    }

    <div class="contact-grid">
      <div class="contact-form-wrapper">
        <EditForm Model="model" OnValidSubmit="HandleSubmit">
          <DataAnnotationsValidator />

          <div class="form-group">
            <label for="name">Name</label>
            <InputText id="name" class="form-control" @bind-Value="model.Name" placeholder="Your name" />
            <ValidationMessage For="() => model.Name" />
          </div>

          <div class="form-group">
            <label for="email">Email</label>
            <InputText id="email" class="form-control" @bind-Value="model.Email" placeholder="your@email.gov" />
            <ValidationMessage For="() => model.Email" />
          </div>

          <div class="form-group">
            <label for="message">Message</label>
            <InputTextArea id="message" class="form-control" @bind-Value="model.Message" rows="5"
              placeholder="Tell us about your legacy system or modernization needs..." />
            <ValidationMessage For="() => model.Message" />
          </div>

          <button type="submit" class="btn btn-primary" disabled="@isSubmitting">
            @(isSubmitting ? "Sending..." : "Send Message")
          </button>
        </EditForm>
      </div>

      <div class="contact-direct">
        <h3>Or reach us directly</h3>
        <p>
          <a href="mailto:info@darkmatter.consulting" class="email-link">
            info@darkmatter.consulting
          </a>
        </p>
        <p class="contact-note">
          Based in Eugene, Oregon.<br />
          Remote-capable. Government-experienced.
        </p>
      </div>
    </div>
  </div>
</section>

@code {
    @inject HttpClient Http

    private ContactModel model = new();
    private string? statusMessage;
    private bool isSuccess;
    private bool isSubmitting;

    private async Task HandleSubmit()
    {
        isSubmitting = true;
        statusMessage = null;

        try
        {
            var request = new HttpRequestMessage(HttpMethod.Post,
                "https://formspree.io/f/YOUR_FORMSPREE_ID");
            request.Headers.Accept.Add(
                new MediaTypeWithQualityHeaderValue("application/json"));
            request.Content = JsonContent.Create(new
            {
                name = model.Name,
                email = model.Email,
                message = model.Message
            });

            var response = await Http.SendAsync(request);
            isSuccess = response.IsSuccessStatusCode;
            statusMessage = isSuccess
                ? "Thanks! We'll be in touch soon."
                : "Something went wrong. Please email us directly at info@darkmatter.consulting.";
        }
        catch
        {
            isSuccess = false;
            statusMessage = "Unable to send message. Please email us directly at info@darkmatter.consulting.";
        }

        isSubmitting = false;
    }

    public class ContactModel
    {
        [System.ComponentModel.DataAnnotations.Required(ErrorMessage = "Name is required")]
        public string Name { get; set; } = "";

        [System.ComponentModel.DataAnnotations.Required(ErrorMessage = "Email is required")]
        [System.ComponentModel.DataAnnotations.EmailAddress(ErrorMessage = "Enter a valid email")]
        public string Email { get; set; } = "";

        [System.ComponentModel.DataAnnotations.Required(ErrorMessage = "Message is required")]
        public string Message { get; set; } = "";
    }
}
```

---

### Components/FooterSection.razor

```razor
<footer class="site-footer">
  <div class="container footer-inner">
    <img src="img/logo.png" alt="Dark Matter Consulting" class="footer-logo" />
    <p class="footer-copy">&copy; 2026 Dark Matter Consulting. Eugene, Oregon.</p>
    <a href="https://github.com/DarkMatterOregon" class="footer-link" target="_blank" rel="noopener">
      GitHub
    </a>
  </div>
</footer>
```

---

## Step 12 — wwwroot/css/app.css

Delete the default `app.css` generated by the template (which references Bootstrap). Replace entirely with:

```css
/* ============================================================
   Dark Matter Consulting — app.css
   Design reference: https://designsystem.digital.gov/
   ============================================================ */

:root {
  --color-navy:       #1b2a4a;
  --color-navy-dark:  #111d33;
  --color-blue:       #0050d8;
  --color-blue-hover: #0040b0;
  --color-white:      #ffffff;
  --color-gray-light: #f0f4f8;
  --color-gray-mid:   #dde3ea;
  --color-gray-text:  #5c6370;
  --color-success:    #00a91c;
  --color-error:      #d63e04;
  --font-stack:       'Source Sans Pro', 'Helvetica Neue', Helvetica, Roboto, Arial, sans-serif;
  --max-width:        1100px;
  --nav-height:       64px;
}

/* ---- Reset & Base ---- */
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

html { scroll-behavior: smooth; }

body {
  font-family: var(--font-stack);
  font-size: 1rem;
  line-height: 1.6;
  color: #1b2a4a;
  background: var(--color-white);
}

img { max-width: 100%; display: block; }

a { color: var(--color-blue); text-decoration: none; }
a:hover { text-decoration: underline; }

h1, h2, h3 { line-height: 1.25; font-weight: 700; }

/* ---- Loading screen ---- */
.loading-screen {
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 100vh;
  background: var(--color-navy);
}
.loading-logo { width: 220px; filter: invert(1) brightness(2); opacity: 0.7; }

/* ---- Blazor error UI ---- */
#blazor-error-ui {
  background: var(--color-error);
  color: white;
  padding: 0.75rem 1rem;
  position: fixed;
  bottom: 0; left: 0; right: 0;
  display: none;
  z-index: 9999;
  font-size: 0.9rem;
}
#blazor-error-ui .reload { color: white; font-weight: bold; margin-left: 0.5rem; }
#blazor-error-ui .dismiss { float: right; cursor: pointer; }

/* ---- Layout utilities ---- */
.container {
  max-width: var(--max-width);
  margin: 0 auto;
  padding: 0 1.25rem;
}

.section { padding: 4rem 0; }
.section-light { background: var(--color-gray-light); }
.section-dark  { background: var(--color-navy); color: var(--color-white); }
.section-dark h2, .section-dark h3 { color: var(--color-white); }

/* ---- Buttons ---- */
.btn {
  display: inline-block;
  padding: 0.75rem 1.75rem;
  border-radius: 4px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  border: 2px solid transparent;
  text-decoration: none;
  transition: background 0.15s, border-color 0.15s;
}
.btn-primary {
  background: var(--color-blue);
  color: var(--color-white);
  border-color: var(--color-blue);
}
.btn-primary:hover {
  background: var(--color-blue-hover);
  border-color: var(--color-blue-hover);
  text-decoration: none;
}
.btn-primary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* ---- Nav ---- */
.site-header {
  position: fixed;
  top: 0; left: 0; right: 0;
  background: var(--color-navy);
  height: var(--nav-height);
  z-index: 100;
  box-shadow: 0 2px 6px rgba(0,0,0,0.3);
}

.nav-container {
  max-width: var(--max-width);
  margin: 0 auto;
  padding: 0 1.25rem;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.nav-logo img {
  height: 36px;
  filter: invert(1) brightness(2);
}

.nav-links {
  display: flex;
  list-style: none;
  gap: 2rem;
  align-items: center;
}

.nav-links a {
  color: rgba(255,255,255,0.85);
  font-weight: 500;
  font-size: 0.95rem;
}
.nav-links a:hover { color: var(--color-white); text-decoration: none; }

.nav-cta {
  background: var(--color-blue);
  color: var(--color-white) !important;
  padding: 0.4rem 1.1rem;
  border-radius: 4px;
  font-weight: 600 !important;
}
.nav-cta:hover { background: var(--color-blue-hover); }

/* Pure CSS hamburger — hidden on desktop */
.nav-toggle       { display: none; }
.nav-toggle-label { display: none; cursor: pointer; flex-direction: column; gap: 5px; }
.nav-toggle-label span {
  display: block; width: 24px; height: 2px;
  background: var(--color-white); border-radius: 2px;
}

/* ---- Hero ---- */
.hero {
  position: relative;
  min-height: 100vh;
  background: url('../img/black_hole_milkyway_header.jpg') center center / cover no-repeat;
  display: flex;
  align-items: center;
  justify-content: center;
  padding-top: var(--nav-height);
}

.hero-overlay {
  position: absolute;
  inset: 0;
  background: rgba(0, 0, 0, 0.62);
}

.hero-content {
  position: relative;
  z-index: 1;
  text-align: center;
  color: var(--color-white);
  max-width: 760px;
  padding: 2rem 1.25rem;
}

.hero-logo {
  width: 280px;
  filter: invert(1) brightness(2);
  margin: 0 auto 2rem;
}

.hero-content h1 {
  font-size: clamp(2rem, 5vw, 3.25rem);
  color: var(--color-white);
  margin-bottom: 1.25rem;
}

.hero-sub {
  font-size: 1.2rem;
  color: rgba(255,255,255,0.88);
  margin-bottom: 2rem;
  max-width: 600px;
  margin-left: auto;
  margin-right: auto;
}

/* ---- Services ---- */
.section-light h2,
.services-grid h3 { color: var(--color-navy); }

.section h2 {
  font-size: 1.9rem;
  margin-bottom: 2.5rem;
  padding-bottom: 0.75rem;
  border-bottom: 3px solid var(--color-blue);
  display: inline-block;
}

.services-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 1.5rem;
}

.service-card {
  background: var(--color-white);
  border: 1px solid var(--color-gray-mid);
  border-radius: 4px;
  padding: 1.75rem;
  box-shadow: 0 1px 4px rgba(0,0,0,0.07);
}

.service-card h3 {
  font-size: 1.15rem;
  margin-bottom: 0.75rem;
  color: var(--color-navy);
}

.service-card p { color: var(--color-gray-text); line-height: 1.65; }

/* ---- About ---- */
.about-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 2.5rem;
}

.about-text p { margin-bottom: 1rem; color: rgba(255,255,255,0.88); }

.credential-list {
  list-style: none;
  padding: 0;
}
.credential-list li {
  padding: 0.6rem 0;
  border-bottom: 1px solid rgba(255,255,255,0.12);
  color: rgba(255,255,255,0.9);
  font-weight: 500;
  padding-left: 1.25rem;
  position: relative;
}
.credential-list li::before {
  content: '✓';
  position: absolute;
  left: 0;
  color: var(--color-blue);
  font-weight: 700;
}

/* ---- Testimonials ---- */
.testimonials-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 1.5rem;
}

.testimonial {
  background: var(--color-white);
  border-left: 4px solid var(--color-blue);
  border-radius: 0 4px 4px 0;
  padding: 1.5rem 1.75rem;
  box-shadow: 0 1px 4px rgba(0,0,0,0.07);
}

.testimonial p {
  font-size: 1.05rem;
  color: var(--color-navy);
  font-style: italic;
  margin-bottom: 1rem;
  line-height: 1.7;
}

.testimonial footer {
  font-size: 0.9rem;
  font-weight: 600;
  color: var(--color-gray-text);
}

/* ---- Contact ---- */
.contact-intro {
  color: rgba(255,255,255,0.85);
  font-size: 1.1rem;
  margin-bottom: 2rem;
}

.contact-grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 3rem;
}

.form-group { margin-bottom: 1.25rem; }

.form-group label {
  display: block;
  font-weight: 600;
  margin-bottom: 0.35rem;
  color: rgba(255,255,255,0.9);
  font-size: 0.95rem;
}

.form-control {
  width: 100%;
  padding: 0.6rem 0.75rem;
  font-size: 1rem;
  font-family: var(--font-stack);
  border: 1px solid var(--color-gray-mid);
  border-radius: 4px;
  background: var(--color-white);
  color: var(--color-navy);
}
.form-control:focus {
  outline: 2px solid var(--color-blue);
  outline-offset: 1px;
  border-color: var(--color-blue);
}

textarea.form-control { resize: vertical; }

.validation-message { color: #f8a; font-size: 0.85rem; margin-top: 0.25rem; }

.alert {
  padding: 0.9rem 1.25rem;
  border-radius: 4px;
  margin-bottom: 1.5rem;
  font-weight: 500;
}
.alert-success { background: #e6f4ea; color: #1a5c28; border: 1px solid #a8d5b5; }
.alert-error   { background: #fce8e0; color: #8c2800; border: 1px solid #f0b8a0; }

.contact-direct h3 { color: rgba(255,255,255,0.9); margin-bottom: 1rem; }

.email-link {
  color: #7eb8ff;
  font-size: 1.1rem;
  font-weight: 600;
  word-break: break-all;
}
.email-link:hover { color: var(--color-white); }

.contact-note {
  margin-top: 1.5rem;
  color: rgba(255,255,255,0.65);
  font-size: 0.95rem;
  line-height: 1.7;
}

/* ---- Footer ---- */
.site-footer {
  background: var(--color-navy-dark);
  padding: 2rem 0;
  border-top: 3px solid var(--color-blue);
}

.footer-inner {
  display: flex;
  align-items: center;
  justify-content: space-between;
  flex-wrap: wrap;
  gap: 1rem;
}

.footer-logo {
  height: 32px;
  filter: invert(1) brightness(1.5);
  opacity: 0.75;
}

.footer-copy { color: rgba(255,255,255,0.55); font-size: 0.875rem; }

.footer-link { color: rgba(255,255,255,0.6); font-size: 0.875rem; }
.footer-link:hover { color: var(--color-white); text-decoration: none; }

/* ============================================================
   Responsive — Tablet 768px+
   ============================================================ */
@media (min-width: 768px) {
  .services-grid    { grid-template-columns: 1fr 1fr; }
  .testimonials-grid { grid-template-columns: 1fr 1fr; }
  .contact-grid     { grid-template-columns: 2fr 1fr; }
  .about-grid       { grid-template-columns: 3fr 2fr; }
}

/* ============================================================
   Responsive — Desktop 1024px+
   ============================================================ */
@media (min-width: 1024px) {
  .services-grid { grid-template-columns: 1fr 1fr 1fr 1fr; }
}

/* ============================================================
   Mobile nav — hamburger
   ============================================================ */
@media (max-width: 767px) {
  .nav-toggle-label { display: flex; }

  .nav-links {
    display: none;
    position: absolute;
    top: var(--nav-height);
    left: 0; right: 0;
    background: var(--color-navy-dark);
    flex-direction: column;
    padding: 1rem 0;
    gap: 0;
    text-align: center;
  }

  .nav-links li { width: 100%; }
  .nav-links a {
    display: block;
    padding: 0.75rem 1.25rem;
    font-size: 1rem;
  }
  .nav-links a:hover { background: rgba(255,255,255,0.06); }

  .nav-toggle:checked ~ .nav-links { display: flex; }

  .nav-cta { border-radius: 0; padding: 0.75rem 1.25rem !important; }
}
```

---

## Step 13 — GitHub Actions Deployment

Create `.github/workflows/deploy.yml` at the repo root:

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [master]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET 10
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'

      - name: Publish Blazor WASM
        run: dotnet publish DarkMatterWeb/DarkMatterWeb.csproj -c Release -o release --nologo

      - name: Copy 404.html
        run: cp release/wwwroot/404.html release/wwwroot/404.html || true

      - name: Add .nojekyll
        run: touch release/wwwroot/.nojekyll

      - name: Deploy to gh-pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: release/wwwroot
          force_orphan: true
```

---

## Step 14 — README.md

Replace the existing README:

```markdown
# Dark Matter Consulting — Web Site

Blazor WebAssembly static site for [darkmatter.consulting](https://darkmatter.consulting), hosted on GitHub Pages.

## Prerequisites

- [.NET 10 SDK](https://dotnet.microsoft.com/download)

## Local Development

```bash
cd DarkMatterWeb
dotnet run
```

Browse to `https://localhost:5000` (or the port shown in terminal).

## Production Build

```bash
dotnet publish DarkMatterWeb/DarkMatterWeb.csproj -c Release -o release
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
```

---

## Final File Tree

```
(repo root)
├── .github/
│   └── workflows/
│       └── deploy.yml
├── DarkMatterWeb/
│   ├── DarkMatterWeb.csproj
│   ├── Program.cs
│   ├── App.razor
│   ├── _Imports.razor
│   ├── Layout/
│   │   └── MainLayout.razor
│   ├── Pages/
│   │   └── Home.razor
│   ├── Components/
│   │   ├── NavBar.razor
│   │   ├── HeroSection.razor
│   │   ├── ServicesSection.razor
│   │   ├── AboutSection.razor
│   │   ├── TestimonialsSection.razor
│   │   ├── ContactSection.razor
│   │   └── FooterSection.razor
│   └── wwwroot/
│       ├── index.html
│       ├── 404.html
│       ├── .nojekyll
│       ├── img/
│       │   ├── logo.png              ← place the provided logo here
│       │   └── black_hole_milkyway_header.jpg
│       └── css/
│           └── app.css
├── LICENSE                           ← preserve existing MIT license
└── README.md
```

---

## Post-Build Checklist (Manual Steps)

1. **Drop `logo.png`** into `DarkMatterWeb/wwwroot/img/`
2. **Download hero image** into `DarkMatterWeb/wwwroot/img/black_hole_milkyway_header.jpg`
   - Source: `https://raw.githubusercontent.com/DarkMatterOregon/dark-matter-static-web/master/static/img/black_hole_milkyway_header.jpg`
3. **Set Formspree ID** — replace `YOUR_FORMSPREE_ID` in `ContactSection.razor`
4. **Enable GitHub Pages** in repo Settings → Pages → Source: `gh-pages` branch
5. **Custom domain** — see README instructions above if pointing `darkmatter.consulting` to this site

---

## Known .NET 10 Gotchas to Watch For

- The `--empty` template flag may not be available in all .NET 10 preview builds. If scaffolding fails, use `dotnet new blazorwasm --name DarkMatterWeb --output DarkMatterWeb --no-https` and manually delete `Pages/Weather.razor`, `WeatherForecast.cs`, and `wwwroot/sample-data/`.
- `blazor.boot.json` does not exist in .NET 10 — do not reference it anywhere.
- The `_framework/blazor.webassembly.js` script reference in `index.html` is correct. Do not change it to `blazor.web.js` (that is for Blazor Web App / server-side hosting).
- `launchSettings.json` no longer controls the WASM environment. Use `<WasmApplicationEnvironmentName>` in `.csproj` if you need to set staging/production explicitly.
- `HttpClient` response streaming is on by default in .NET 10. The Formspree POST pattern above handles this correctly — no changes needed.

# My personal blog

[https://adurrr.github.io](https://adurrr.github.io)

## Deployment

This site is built with [Hugo](https://gohugo.io/) and deployed automatically to GitHub Pages.

### Automatic Deployment

Push to the `main` branch to trigger automatic deployment via GitHub Actions. The workflow:
1. Installs Node.js dependencies (`npm ci`)
2. Builds the site with Hugo 0.145.0 extended (`hugo --minify`)
3. Deploys to GitHub Pages

### Local Development

To build and preview locally:

```bash
# Install dependencies
npm ci

# Install Hugo (if not already installed)
# See https://gohugo.io/installation/

# Start the development server
hugo server

# Build the site
hugo --minify
```

The built site outputs to `./public/`.
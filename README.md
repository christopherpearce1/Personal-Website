# Deploying this site

It's a single static file (`index.html`), so any of these work in a few minutes:

## Vercel (recommended, matches your target aesthetic/example)
1. Create a free account at vercel.com if you don't have one.
2. Install the CLI: `npm i -g vercel`
3. From this folder, run: `vercel`
4. Follow the prompts (accept defaults). You'll get a live `.vercel.app` URL immediately.
5. To use a custom domain later, add it in the Vercel dashboard under the project's Domains tab.

## GitHub Pages (free, no CLI installs needed)
1. Create a new GitHub repo, e.g. `chris-pearce-site`.
2. Push this folder's contents to the repo (`index.html` at the root).
3. In the repo, go to Settings → Pages → set source to the `main` branch, root folder.
4. Your site goes live at `https://<your-username>.github.io/chris-pearce-site/`.

## Netlify
1. Go to app.netlify.com and drag-and-drop this folder onto the dashboard.
2. It deploys instantly with a `.netlify.app` URL.

Any of the three are free and take under 5 minutes.

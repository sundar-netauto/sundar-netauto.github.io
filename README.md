# sundar-blog

Personal blog — network engineering, automation, and infrastructure notes. Built with [Hugo](https://gohugo.io/) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme, deployed to GitHub Pages via GitHub Actions.

## Local preview

Requires Hugo (extended) installed locally: `brew install hugo`.

```bash
hugo server -D
```

Then open http://localhost:1313/.

## New post

```bash
hugo new posts/my-new-post.md
```

Edit the file under `content/posts/`, set `draft: false` when ready, commit and push to `main` — the GitHub Actions workflow builds and deploys automatically.

## First-time setup (do this before your first push)

1. In `hugo.toml`, replace `<your-github-username>` in `baseURL` and the GitHub social icon URL with your actual GitHub username.
2. Create an **empty** repo on GitHub named exactly `<your-github-username>.github.io` — that exact name is what makes GitHub Pages serve it at the clean root URL instead of a `/reponame/` subpath.
3. `git remote add origin git@github.com:<your-github-username>/<your-github-username>.github.io.git`
4. `git add -A && git commit -m "init blog" && git push -u origin main`
5. On GitHub: repo Settings → Pages → Source → set to "GitHub Actions" (one-time).
6. Wait for the Action to finish (Actions tab), then visit `https://<your-github-username>.github.io`.

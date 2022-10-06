# This Cute World / Ryan4Yin's Space

My personal blog, built using [Hugo](https://github.com/gohugoio/hugo) and hosted on [Vercel](https://vercel.com/dashboard/usage) or [GitHub Pages](https://pages.github.com/).

Address: https://thiscute.world/

## Editing

Use the `new` command to create a new post:

```shell
# new posts
hugo new posts/my-first-post/index.md

# new posts with special category name as a prefix
hugo new posts/category/my-first-post/index.md
```

Launch a local Hugo server including live reload by running:

```shell
# serve in debug mode, with all drafts
hugo server --debug --buildDrafts --disableFastRender

# serve in production mode
hugo serve -e production --disableFastRender
```

Now edit the newly created file under `content/posts`, and then you can view the live changes in the browser <http://localhost:1313/>


## Github Action

Push updates to `main` branch will trigger a github action to deploy the updates automatically.

The action workflow will:

- Fetch Posts Trending Data from Google Analytics([website_statistics.json](./data/website_statistics.json)).
- Deploy to Github Pages(branch `gh-page`).
- Push Argolia Index for Search.

See [.github/workflows/](/.github/workflows/) for details.

## Related Repositories

- Serverless Comment System
  - [ryan4yin/waline-comment-api](https://github.com/ryan4yin/waline-comments-api)
  - [ryan4yin/waline-comments-backup](https://github.com/ryan4yin/waline-comments-backup)


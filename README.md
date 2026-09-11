
__Visit https://a-creative.website__

# Templates and Guides for me

### Blog Post Tags
__ETC__

- 2026

__PROGRAMMING__

- Odin
- Raspberry Pi
- SBC
- 500+
- SDK

__TOPICS__

- Videogame
- FuzzyBuddyFarms

# Create

To create new pages, posts, etc

```
hugo new content posts/hello-world.md
hugo new content about.md
```

## Create Blog Post

```
git submodule update --init --recursive
hugo new content posts/EXAMPLE/index.md
```

# Build 

```
hugo new project my-website
cd my-website

git init -b main

git submodule add https://github.com/athul/archie.git themes/archie
```

then

```
cd my-website
touch .gitignore
nvim .gitignore
```

and paste into ```.gitignore```

```
/public/
/resources/_gen/
/.hugo_build.lock
```

Configure ```hugo.toml``` to whatever

Create the workflow directory:

```
mkdir -p .github/workflows
```

Now create ```.github/workflows/hugo.yaml``` with:

```
name: Build and deploy Hugo site

on:
  push:
    branches: ["main"]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: "0.166.0"
      HUGO_ENVIRONMENT: production

    steps:
      - name: Check out source and theme
        uses: actions/checkout@v7
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Configure Pages
        id: pages
        uses: actions/configure-pages@v6

      - name: Install Hugo
        run: |
          mkdir -p "${RUNNER_TEMP}/hugo"
          curl --fail --location --retry 3 \
            "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_${HUGO_VERSION}_linux-amd64.tar.gz" \
            -o "${RUNNER_TEMP}/hugo.tar.gz"
          tar -xzf "${RUNNER_TEMP}/hugo.tar.gz" \
            -C "${RUNNER_TEMP}/hugo"
          echo "${RUNNER_TEMP}/hugo" >> "${GITHUB_PATH}"

      - name: Build website
        run: |
          hugo build --gc --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"

      - name: Upload website
        uses: actions/upload-pages-artifact@v5
        with:
          path: ./public

  deploy:
    runs-on: ubuntu-latest
    needs: build

    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v5
```
# Merge back

If anything gets funky simply

```
git merge origin/main --allow-unrelated-histories -m "Merge GitHub repository with local Hugo site"
```

and

```
git add .
git push origin main
```

# Test Locally

```
hugo build --gc --minify
```

# Optional Assets

create ```assets/css/custom.css``` 

Within ```custom.css```

```
/* Sans-serif typography instead of Archie's default appearance. */
html,
body {
  font-family: Helvetica, Arial, sans-serif;
}

p {
  font-family: "Helvetica Neue", Helvetica, Arial, sans-serif;
  line-height: 1.65;
}

/* Wider, centered reading column. */
.content {
  max-width: 920px;
  max-inline-size: 920px;
  margin-inline: auto;
}

/* Larger site title, responsive on smaller screens. */
header .main a {
  font-family: Helvetica, Arial, sans-serif;
  font-size: clamp(1.8rem, 5vw, 3rem);
  line-height: 1.15;
  border-bottom: none;
  border-block-end: none;
}

h1 {
  font-size: 1.9rem;
  line-height: 1.25;
}

h2 {
  font-size: 1.6rem;
}

h3 {
  font-size: 1.4rem;
}

.list-item {
  margin-bottom: 2.5rem;
}

img {
  max-width: 100%;
  height: auto;
}
```

Then add this line inside the existing [params] section in hugo.toml, before [[params.social]]:

```
  customCSS = ["css/custom.css"]
```




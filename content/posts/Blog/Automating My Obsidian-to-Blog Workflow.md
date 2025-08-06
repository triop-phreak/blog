---
tags:
  - github
  - cicd
  - github_actions
description: A walkthrough of how I automated publishing Obsidian notes to my blog using GitHub Actions and tighter plugin integration.
images:
  - /ob/attachments/Pasted%20image%2020250504180948.png
title: Automating My Obsidian-to-Blog Workflow
date: 2025-05-27T19:48:43.000Z
lastmod: 2025-05-27T19:48:43.000Z
---
## Overview

I was inspired to create this blog after watching a NetworkChuck [video](https://www.youtube.com/watch?v=dnE7c0ELEH8) from about 5 months ago. He talked about building out a pipeline to automatically format notes from [Obsidian](https://obsidian.md/) using [Hugo](https://gohugo.io/), push the results to a GitHub repository, and then have GitHub send a webhook to Hostinger which would trigger Hostinger to clone his repository and update his website. I think that the pipeline he created is really cool and he has a detailed post about it on his [blog](https://blog.networkchuck.com/posts/my-insane-blog-pipeline/) so I decided to try it out for myself. I figured that if I was going down this rabbit hole anyway, I figured I might as well try to make some improvements to his pipeline and get hands-on with [GitHub Actions](https://docs.github.com/en/actions) along the way!

## Improvements

After watching Chuck's video, the biggest opportunities for improvement in my opinion were to offload as much of the processing/work from the endpoint as possible, and to make the deployment of new posts more integrated with Obsidian (instead of maintaining a separate script).

### 1. Offload Processing and Configuration with GitHub Actions

Chuck put together a Python script that copies blog posts and images from his Obsidian vault, reformats image references so that Hugo will understand them, runs Hugo, and then pushes the results to his GitHub repository. I decided that instead of running Python locally, this would be a perfect opportunity to get hands-on with GitHub actions!

I created a GitHub action workflow that installs Hugo, builds the site based on the data in my GitHub repository, and then posts it to GitHub Pages:

```
name: hugo-build-deploy

# Controls when the workflow will run
on:
  # Triggers the workflow on push or pull request events but only for the "main" branch
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

  # Allows you to run this workflow manually from the Actions tab
  workflow_dispatch:

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  build:
    runs-on: ubuntu-latest
    env: 
      HUGO_VERSION: 0.147.0

    steps:
      - name: Install hugo
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb \
          && sudo dpkg -i ${{ runner.temp }}/hugo.deb

      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5

      - name: Install Node.js dependencies
        run: "[[ -f package-lock.json || -f npm-shrinkwrap.json ]] && npm ci || true"

      - name: Build with hugo
        env:
          HUGO_CACHEDIR: ${{ runner.temp }}/hugo_cache
          HUGO_ENVIRONMENT: production
        run: |
          hugo \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/"

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    runs-on: ubuntu-latest
    needs: build
    permissions:
      contents: write
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    steps:
      - name: Deploy to GitHub pages
        id: deployment
        uses: actions/deploy-pages@v4

        

```

You'll also notice that I'm deploying to GitHub pages rather than Hostinger. GitHub pages is free (even with custom domains) and integrates tightly with GitHub actions and the broader GitHub ecosystem.

Now whenever I push updated content to my GitHub repo, the workflow I created automatically runs and deploys the latest content to pages:

![Pasted image 20250504180948.png](/ob/attachments/Pasted%20image%2020250504180948.png)

### 2. Tighter Integration with Obsidian

The other missing piece for me was the ability to publish directly from within Obsidian and to solve that I found the [Hugo Publish](https://github.com/kirito41dd/obsidian-hugo-publish) plug-in for Obsidian. You can install this plug-in from Obsidian’s Community Plug-Ins section in the settings.

![Pasted image 20250504180737.png](/ob/attachments/Pasted%20image%2020250504180737.png)

Once the plug-in is installed, there are a few settings you need to configure including the local folder to publish to as well as the content/posts folder that you want to use within the folder structure. You can also configure the plug-in to only post notes with a certain tag so that your private notes don't get published to your blog.

![Pasted image 20250504181112.png](/ob/attachments/Pasted%20image%2020250504181112.png)

This plugin also handles all of the image copying and image reference conversions to get images working in your blog posts just like they do in Obsidian.

## Future Improvements

The one obvious weak point that still exists in my pipeline is pushing changes to my GitHub repository. The Obsidian plug-in I found posts changes to a local folder, but it doesn't automatically push them up to GitHub and my workflows only trigger when a new push occurs. In the future, I’d like to set up a button, keyboard shortcut, or cron job to automatically commit and push changes, removing the need to do this manually.

## Conclusion

I'm really glad that NetworkChuck started a blog and inspired me to create one as well. The pipeline he set up was a great idea and I hope that the minor improvements I've made here will be helpful (or at least interesting) to others!

## Resources

* NetworkChuck's Video: [My Insane Blog Pipeline (YouTube)](https://www.youtube.com/watch?v=dnE7c0ELEH8)
* NetworkChuck's Blog Post: [My Insane Blog Pipeline (Blog)](https://blog.networkchuck.com/posts/my-insane-blog-pipeline/)

Tools & Platforms

* Obsidian (note-taking app): <https://obsidian.md/>
* Hugo (static site generator): <https://gohugo.io/>
* GitHub Actions (CI/CD platform): <https://docs.github.com/en/actions>
* GitHub Pages (static site hosting): <https://pages.github.com/>
* Hostinger (web hosting): <https://www.hostinger.com/>

Obsidian Plugins

* Hugo Publish Plug-in: <https://github.com/kirito41dd/obsidian-hugo-publish>\
  *(Can be installed from the Community Plug-ins section in Obsidian settings)*

---
layout: post
title: Publishing Sphinx websites using GitHub Actions
subtitle: Complete with published previews of requested changes!
tags: [GitHub Actions, Sphinx, continuous integration]
readtime: true
---

[Pangeo](https://pangeo.io), an open-source project I frequently contribute to, has historically relied on popular continuous integration (CI) service [Travis CI](https://travis-ci.com/) for most of its CI workflows on GitHub.
Recently, Travis [changed its pricing model](https://blog.travis-ci.com/2020-11-02-travis-ci-new-billing), removing unlimited build time for public repositories on GitHub and consequently causing many open-source CI workflows to fail.
This change created a situation where Pangeo had to rapidly shift many of its repositories using Travis to [GitHub Actions](https://github.com/features/actions), another rapidly growing CI service providing unlimited build time for public repositories.
One important repository to shift over was the [source code](https://github.com/pangeo-data/pangeo) for Pangeo's website,
which is generated using [Sphinx](https://www.sphinx-doc.org/en/master/) and published through [GitHub Pages](https://pages.github.com/).
Being relatively new to GitHub Actions, I decided to take on this task as a small project to familiarize myself with the service.

# Automating Sphinx builds

The first thing to do is find a suitable Action in the [GitHub Marketplace](https://github.com/marketplace?type=actions) that can build the website; an Action is essentially one step in a larger GitHub Actions workflow that can be supplied arguments to perform a singular task.
I came across [ammaraskar/sphinx-action](https://github.com/ammaraskar/sphinx-action), a user-contributed Action which does exactly what I need; a Sphinx website can be built from its source code by adding the following [YAML](https://yaml.org/)-formatted file to its repository in a directory named `.github/workflows`:

```yaml
name: Build Sphinx site

on:
  push:
    branches: [master]
    paths: ["docs/**"]

jobs:
  build_and_publish:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2
      - name: Build Sphinx site
        uses: ammaraskar/sphinx-action@master
        with:
          docs-folder: "docs/"
```

Note the workflow's conditions set in the `on` block; these specify that it only runs when changes are pushed to the `master` branch, and when those changes affect files in the `docs` directory, which contains the source code for the Sphinx website.
This workflow contains one job with two steps:

1. Using [actions/checkout](https://github.com/actions/checkout), it clones the GitHub repository to the working directory of the workflow.
2. Using sphinx-action, it builds the Sphinx website using source code contained in the `docs/` folder of the repository.

Now I have a basic workflow established to build the Sphinx website whenever its source code in the main branch is updated and return an error if the build fails.
It even reports build warnings as annotations within the workflow run:

![Sphinx build warnings in GitHub Actions](/assets/img/2020-12-17/warnings.jpg "Sphinx build warnings in GitHub Actions"){: .mx-auto.d-block :}

The next step is figuring out how to publish the output of this build online.

# Publishing builds through GitHub Pages

With GitHub Pages, it is relatively easy to publish a static website for any repository on GitHub; there are two ways to do this:

- Create a repository named _username_.github.io, where _username_ is the name of your account or organization.
  The source code in the default branch of this repository will be published as is at **https://_username_.github.io**.
- Create a branch named `gh-pages` in any repository other than the one listed above.
  Source code in this branch will be published as is at **https://_username_.github.io/_repo_**, where _repo_ is the name of the repository.

In my case, Pangeo's website had already been published using the second method, so I need to figure out how to push the results of my build from earlier to the `gh-pages` branch within my workflow.
This process can be split up into two parts:

1. Getting the current copy of the `gh-pages` branch and committing to it any changes made in this new Sphinx build.
2. Pushing this updated branch back to its origin on GitHub.

To handle the first part of this, I add a step containing a series of [`git`](https://git-scm.com/) commands to the workflow using a default Action called [`run`](https://docs.github.com/en/free-pro-team@latest/actions/reference/workflow-syntax-for-github-actions#jobsjob_idstepsrun):

```yaml
- name: Commit site changes
  run: |
    git clone https://github.com/pangeo-data/pangeo.git --branch gh-pages --single-branch gh-pages
    cp -r docs/_build/html/* gh-pages/
    cd gh-pages
    git config --local user.email "action@github.com"
    git config --local user.name "GitHub Action"
    git add .
    git commit -m "Update site" -a || true
```

These commands do the following:

- Clone the current version of the `gh-pages` branch to the working directory.
- Copy the results of the build (located in `docs/_build/html/`) to the folder containing this branch.
- Stage any changes made to the website under the username "GitHub Action."
- Commit these changes to the branch, or return `true` if no changes have been made so that GitHub Actions doesn't see this as a build failure.

With the changes committed, all that is left is pushing them to the origin.
This can be done using another user-contributed Action called [ad-m/github-push-action](https://github.com/ad-m/github-push-action):

<!-- {% raw %} -->

```yaml
- name: Push site changes to gh-pages
  uses: ad-m/github-push-action@master
  with:
    branch: gh-pages
    directory: gh-pages
    github_token: ${{ secrets.GITHUB_TOKEN }}
```

<!-- {% endraw %} -->

Here, the arguments specify to push the contents of the `gh-pages` directory in the workspace to the `gh-pages` branch of the repository.
Note the inclusion of `github_token` as an argument; any Action that changes the repository it is contained in will require authentication to work.
This is provided by using `secrets.GITHUB_TOKEN`, a variable which represents the authentication token for the repository.
Also note that this variable is an [encrypted secret](https://docs.github.com/en/free-pro-team@latest/actions/reference/encrypted-secrets) and cannot be accessed if the workflow is triggered by a forked repository.

Now that the workflow is complete, any updates to the Sphinx website's source code will trigger a build, with any changes published online.
But with no way to track if these updates actually _work_, it is entirely possible for a user to commit changes to the website that break it!
To avoid this, I will make another workflow specifically for testing.
Users can then open a pull request proposing changes to the website to trigger a test build; if it fails, the user can check what went wrong and make the necessary corrections before committing the changes.

# Testing and previewing proposed changes

Now that I have a workflow to build and publish the Sphinx website, making one to test proposed changes is relatively simple; I can just take the workflow I started with and adjust the `on` block to account for the different triggering condition:7

```yaml
name: Build Sphinx site

on:
  pull_request:
    branches: [master]
    paths: ["docs/**"]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2
      - name: Build Sphinx site
        uses: ammaraskar/sphinx-action@master
        with:
          docs-folder: "docs/"
```

With the `on` condition set to `pull_request`, this workflow will only run when changes are proposed for the `master` branch affecting files in the `docs` directory.

On its own, this workflow will ensure that users know if their proposed changes will result in a successful build.
But what if they aren't concerned with build errors, and their primary concern is how some changes will look once they are rendered on the website?
In cases like this, it can be useful to share a preview of the website based on the proposed changes, where users can view the result of their changes without having to publish.

Sharing previews of the website is not too different from publishing, with the main differences being that:

- Once the preview website is built, instead of pushing it to the `gh-pages` branch, it will be pushed to its own dedicated preview branch.
- Instead of publishing the preview website, it will be shared using an awesome website called [raw.githack.com](https://raw.githack.com/) to make sure it is rendered as a webpage instead of a plain text file.

**clarify what is being shared**

I only have to make a few adjustments to the publishing workflow to get it working for previews, considering a few minor obstacles to look out for:

- Since a dedicated preview branch for any given pull request may or may not exist, the step that clones and commits to it must use some logic to decide what to do in each case.
- Once the preview website is built, I would like a bot to comment a link to it on the pull request that triggered the build.
  However, I only want the bot to comment once, so that if a user continues to make changes after opening the request, the issue isn't flooded with bot comments.

I am able to handle both concerns by adding an `if..else` block into the original `run` step:

<!-- {% raw %} -->

```yaml
- name: Commit site changes to preview branch
  run: |
    if git clone https://github.com/pangeo-data/pangeo.git --branch ${{ github.head_ref }}-preview --single-branch gh-pages ; then
      cd gh-pages
      echo "COMMENT_ON_PR=false" >> $GITHUB_ENV
    else
      git clone https://github.com/pangeo-data/pangeo.git --branch gh-pages --single-branch gh-pages
      cd gh-pages
      git checkout -b ${{ github.head_ref }}-preview
      echo "COMMENT_ON_PR=true" >> $GITHUB_ENV
    fi
    cp -r ../docs/_build/html/* .
    git config --local user.email "action@github.com"
    git config --local user.name "GitHub Action"
    git add .
    git commit -m "Update site" -a || true
```

<!-- {% endraw %} -->

First, to explain the new variables being used here:

- `github.head_ref` refers to the branch from which changes are being proposed and is used to decide the name of the preview branch.
  For example, if website changes are being proposed from `my-cool-branch`, then the preview website will be committed to `my-cool-branch-preview`.
- `GITHUB_ENV` refers to the environment variables in use during the workflow's run, and in this context, I use it to set a variable called `COMMENT_ON_PR`; this will come in handy later when the workflow needs to decide if a bot should leave a comment on the pull request.

With that out of the way, the `if..else` statement reads as follows:

- If the preview branch is cloned successfully, it must already exist, so the bot should have already commented on the issue; set `COMMENT_ON_PR` to `false`.
- Otherwise, clone the `gh-pages` branch and make a preview branch based on it; in this case, I assume the pull request has just been opened and so the bot should comment; set `COMMENT_ON_PR` to `true`.

From here, the rest of the commands are almost identical to the original, committing the changes to the preview branch.
To push these changes to GitHub, I use nearly the same step from before, replacing `gh-pages` with the name of the preview branch:

<!-- {% raw %} -->

```yaml
- name: Push site changes to preview branch
  uses: ad-m/github-push-action@master
  with:
    branch: ${{ github.head_ref }}-preview
    directory: gh-pages
    github_token: ${{ secrets.GITHUB_TOKEN }}
```

<!-- {% endraw %} -->

With the preview website built and pushed to GitHub, the workflow just needs to leave a comment with a link to the website under the pull request if it is required.
This can be done using [actions/github-script](https://github.com/actions/github-script), conditional on the value of `COMMENT_ON_PR`:

<!-- {% raw %} -->

```yaml
- name: Leave comment on pull request
  if: env.COMMENT_ON_PR == 'true'
  uses: actions/github-script@v3
  with:
    github-token: ${{secrets.GITHUB_TOKEN}}
    script: |
      github.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: 'Thank you for your contributions!\n\nA preview of your changes can be viewed at:\n- https://raw.githack.com/pangeo-data/pangeo/${{ github.head_ref }}-preview/index.html'
        })
```

<!-- {% endraw %} -->

With this done, once a pull request is opened proposing changes to the website, a bot will comment with a link to the preview build if it succeeds:

![Bot comment in action](/assets/img/2020-12-17/bot.jpg "Bot comment in action"){: .mx-auto.d-block :}

There's only one more use case to consider; since there are steps in this workflow that require authentication that is unavailable to forked repositories, I must also make a simpler testing workflow for pull requests from forked repositories, with an `if` statement checking whether or not this is the case.
It's a little convoluted, but this can be done by comparing the full name of the repository from which changes are being proposed to the name of the main repository:

```yaml
name: Build and preview Sphinx site

on:
  pull_request:
    branches: [ master ]
    paths: [ 'docs/**' ]

jobs:
  build_and_preview:
    if: github.event.pull_request.head.repo.full_name == github.repository
    ...
  build:
    if: github.event.pull_request.head.repo.full_name != github.repository
    ...
```

In cases where they are equal, indicating that changes are being proposed from a local branch, the workflow can build and share the website preview using the repository's authentication token.
For all other cases, the workflow will only run a test build, which doesn't require any authentication.

# Final thoughts

Setting up these workflows was a lot easier than I expected.
I owe much of that to the vast array of user-contributed Actions that were able to do generally any complex task I needed.
I'd also like to think that many aspects of these workflows are easier to understand, as they generally keep complex command line operations to a minimum and tend to rely on short, named steps which can be examined individually.

There are some bugs in the system that I may try to fix later, but they seem like relatively uncommon cases that aren't likely to happen on lower activity repositories; some examples include:

- Trying to create a preview branch for `my-cool-branch` will fail if `my-cool-branch-preview` already exists from a previous pull request.
- Manually pushing changes to a preview branch will cause it to fall out of sync with the pull request workflow, causing all subsequent runs to fail until it is deleted.
- If the preview branch is deleted and rebuilt, it will qualify as a "new" branch, and another bot comment will be added to the pull request.

Overall, this was a fun project that gave me a useful workflow to use for any future Sphinx websites I might need to make.
It also taught me a lot about the behavior of `git clone` and using conditionals in GitHub Actions, knowledge I'm sure will be invaluable in the future; hopefully you learned something as well!

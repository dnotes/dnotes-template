# Project Template

This is a template repository for creating new Node.js projects with a modern monorepo setup. It includes automated CI/CD, versioning, and release management.

## Features

- PNPM Workspaces (monorepo): libraries in `packages/`, websites in `sites/`
- Supply-chain protections enabled in `pnpm-workspace.yaml`
- Automated testing on every pull request
- Automated versioning and npm releases with Changesets, using [trusted publishing](https://docs.npmjs.com/trusted-publishers) (no npm tokens)
- Sites set up to auto-deploy (e.g. with Cloudflare) only when a new version is released
- CodeQL security scanning and Dependabot updates

## What's in `.github/`

| File | What it does |
|---|---|
| `workflows/release.yml` | Tests every pull request. On `main`, runs the tests again, then handles releases (release PRs and npm publishing). |
| `workflows/codeql.yml` | Scans the code for security mistakes on every pull request, on `main`, and weekly. |
| `dependabot.yml` | Opens weekly pull requests to update npm packages and GitHub Actions. |
| `settings.yml` | Repository settings and branch protection, applied by the [Settings app](https://github.com/apps/settings). |

Each file starts with a plain-language explanation.

## Requirements

- Node.js 24 (see `.nvmrc`)
- pnpm, at the version pinned in `package.json` under `packageManager`
- The [GitHub CLI](https://cli.github.com) (`gh`), for creating new projects

## Creating a new project

This repository is a GitHub template repository. Create a new repository from it with:

```sh
gh repo create my-project --template dnotes/dnotes-template --public --clone
cd my-project
pnpm install
```

You can also click **Use this template** on the repository page on GitHub.

The repository should be public. CodeQL scanning (`codeql.yml`) is free only for public repositories; on a private repository without a paid GitHub Code Security plan it fails, so delete `codeql.yml` there. npm also only adds provenance to packages published from public repositories.

Tools like `degit` and `tiged` still work, but they only copy the files; you'd then have to create the GitHub repository and push it yourself. A GitHub template creates the repository and its first commit in one step.

## Setup checklist

Do this once for each new project.

- [ ] **Rename the project**
  - Update `name` and `description` in the root `package.json`

- [ ] **Install the Settings app** (once for your whole account, not per project)
  - Go to https://github.com/apps/settings and click "Install" (or "Configure" if it's already installed)
  - Choose your account, then "All repositories", so new projects are covered automatically
  - After creating a project, check that it worked: in the repository go to Settings > Branches and look for a rule on `main`. If there isn't one, make any small edit to `.github/settings.yml` and push it to `main`; the app applies the file whenever it changes.

- [ ] **Allow the workflow to open release pull requests**
  - Settings > Actions > General > Workflow permissions
  - Check "Allow GitHub Actions to create and approve pull requests", then save

- [ ] **Turn on security features**
  - Settings > Advanced Security
  - Enable "Private vulnerability reporting", "Dependabot alerts" and "Dependabot security updates"
  - Leave CodeQL "default setup" off. This project uses `codeql.yml` instead, and GitHub allows only one of the two.

- [ ] **If you publish packages to npm**
  - Create the publishing environment:
    - Settings > Environments > New environment, named `npm-publish`
    - Under "Deployment protection rules", check "Required reviewers" and add yourself, so every publish waits for your approval
    - Under "Deployment branches and tags", choose "Selected branches and tags" and add `main`
  - Set up trusted publishing for each package:
    - On https://npmjs.com, open the package and go to "Settings"
    - Under "Trusted Publisher", choose "GitHub Actions" and enter your GitHub user, the repository name, workflow filename `release.yml`, and environment `npm-publish`
    - Under "Publishing access", choose to disallow tokens
    - npm may not let you configure a package that has never been published. If so, publish the first version from your computer with `npm publish` and then set this up.
  - You don't need an `NPM_TOKEN` secret. Delete any old one.

- [ ] **If you have sites to deploy**, follow [Deploying sites](#deploying-sites) below.

## Making changes and releasing

1. Create a branch, make your changes, and run `pnpm changeset`. It asks which packages changed and whether each change is a patch, minor or major, and writes a short note to `.changeset/`. Commit that file with your changes.
2. Open a pull request. The `test` check must pass before it can be merged.
3. When it's merged, the workflow opens (or updates) a pull request called "chore: new release candidate", which bumps the version numbers and updates each package's CHANGELOG. More merged changes are added to the same release pull request.
4. When you're ready to release, merge the release pull request. The workflow runs the tests on `main`, then publishes the new versions to npm (after your approval). Hosting services connected to your sites deploy the ones whose version changed.

**New packages are published automatically.** Whenever `main` has a package whose version isn't on npm yet, the workflow publishes it (after your approval). Keep `"private": true` in a new package's `package.json` until you're ready to publish it. Remember that npm may require you to publish the very first version by hand (see the setup checklist).

**Merging when you're the only maintainer.** Branch protection requires one approving review, and GitHub doesn't let you approve your own pull requests. Because you're an admin and `settings.yml` sets `enforce_admins: false`, GitHub offers you "Merge without waiting for requirements to be met". Use it once the tests pass.

**The release pull request won't show a `test` check.** GitHub doesn't run workflows on pull requests opened by a workflow. You can merge it the same way (as admin): nothing is published until the tests pass on `main`. If you want to see the tests run on the release pull request itself, start them by hand: go to the Actions tab, choose "CI & Release", click "Run workflow", and pick the `changeset-release/main` branch. Or, from the command line:

```sh
gh workflow run release.yml --ref changeset-release/main
```

A manual run only runs the tests; it never releases anything.

## Deploying sites

Each folder in `sites/` is a website with its own `package.json`. Sites are versioned with Changesets like the packages are, but they're never published to npm. Deploys are handled by your hosting service's own GitHub integration, not by this repository's workflows.

The trick is to have the host deploy only when a site gets a new version. Every time Changesets releases a site, it updates that site's `CHANGELOG.md`, and nothing else changes that file. So if the host only builds when `sites/<name>/CHANGELOG.md` changes, the site deploys exactly when a release pull request that bumps it is merged.

A site is also released automatically when a workspace package in its `dependencies` is released, so it redeploys with the new version. Packages listed only in `devDependencies` don't trigger this, so put workspace packages the site uses in `dependencies`.

### Adding a site

1. Create the site in `sites/<name>/` with any framework. Its `package.json` needs `"private": true` (to keep it off npm) and a `build` script:

   ```json
   {
     "name": "example",
     "version": "0.0.0",
     "private": true,
     "scripts": {
       "build": "..."
     }
   }
   ```

2. If the site deploys with Cloudflare's `wrangler`, add it to the site with `pnpm --filter example add -D wrangler`. pnpm will refuse to finish because wrangler's dependencies `esbuild` and `workerd` have install scripts. Approve them by adding this to `pnpm-workspace.yaml` (or by running `pnpm approve-builds`), then run `pnpm install` again:

   ```yaml
   allowBuilds:
     esbuild: true
     workerd: true
   ```

3. Commit the site, including the updated `pnpm-lock.yaml`.

### Connecting a site to Cloudflare

These steps use Cloudflare Workers, which can host static sites as well as server code. Cloudflare Pages projects have similar settings, including build watch paths.

1. In the Cloudflare dashboard, go to Workers & Pages > Create application > GitHub, authorize GitHub if asked, and pick this repository.
2. Configure the project, then select Save and Deploy:
   - **Root directory:** `sites/example` (the folder that contains the site's `wrangler.jsonc`)
   - **Build command:** `pnpm --filter example... build` (this also builds any workspace packages the site uses)
   - **Deploy command:** `npx wrangler deploy` (the default)
3. After it's created, open the Worker and go to Settings > Build:
   - **Branch control:** keep the production branch as `main`. Turn off "Enable Preview Builds" unless you want preview deploys of the release pull request.
   - **Build watch paths:** set "Include paths" to `sites/example/CHANGELOG.md` and leave "Exclude paths" empty.

Cloudflare deploys once when you first connect the site, and afterwards only when its CHANGELOG changes. One exception: Cloudflare builds every connected site when a single push contains 20 or more commits or 3,000 or more changed files, which would deploy unreleased changes. To prevent that, `settings.yml` allows only squash merging, so each merged pull request adds a single commit to `main`. Avoid pushing many commits directly to `main`.

Cloudflare's build image uses the pnpm version pinned in the root `package.json` (`packageManager`) and Node.js 24 by default, so no extra build settings are needed.

Other hosts can do the same thing: Netlify and Vercel both let you skip builds based on which files changed ("ignore builds" settings).

## Supply-chain settings

`pnpm-workspace.yaml` sets:

- `minimumReleaseAge: 10080`: pnpm won't install any package version published less than 7 days ago. To make an exception, add it to `minimumReleaseAgeExclude`, for example `- 'some-package@1.2.3'`. You'll need this for urgent security fixes: Dependabot security pull requests aren't delayed, so their tests fail until the fixed version is either 7 days old or excluded.
- `trustPolicy: no-downgrade`: an install fails if a package that used to be published with provenance suddenly isn't. Occasionally a legitimate package trips this; after checking the release is genuine, add it to `trustPolicyExclude` the same way.
- `allowBuilds`: dependency install scripts are blocked unless listed here. Run `pnpm approve-builds` to review them.

Dependabot uses the same 7-day delay.

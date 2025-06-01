# Project Template

This is a template repository for creating new Node.js projects with a modern monorepo setup. It includes automated CI/CD, versioning, and release management.

## Features

- PNPM Workspaces (monorepo)
- GitHub Actions for CI/CD
- Automated versioning and releases with Changesets
- Automated testing setup

## Setup Checklist

- [ ] Create a new NPM token
  - Login to https://npmjs.com
  - Go to "Access Tokens" under your profile image
  - Click "Generate New Token > Granular Access Token"
  - Some recommended configuration:
    - Name: your repository name
    - Packages and Scopes: Read and write for only the repo that you created

- [ ] Set up the publishing environment
  - Go to your repository's Settings tab
  - Click "Environments" in the left sidebar
  - Click "New environment"
  - Name it: `npm-publish`
  - Click "Add secret"
    - Name: `NPM_TOKEN`
    - Value: paste your NPM token


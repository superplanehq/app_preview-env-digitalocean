# Preview Environments on DigitalOcean

[![Launch in SuperPlane](https://superplane.com/badges/launch-in-superplane.svg)](https://app.superplane.com/install?repo=github.com/superplanehq/app_preview-env-digitalocean)

A SuperPlane app that spins up preview environments on DigitalOcean for GitHub pull requests. Comment `/start` on a PR, get a running app on a fresh droplet in ~2 minutes. Close the PR, environment auto-destroys.

## How it works

1. Open a PR — the bot posts a welcome comment with instructions
2. Comment `/start` — creates a DigitalOcean droplet, deploys your app via SSH, runs a health check, and posts the preview URL
3. Comment `/destroy` — tears everything down
4. Close or merge the PR — environment auto-destroys
5. A scheduled TTL check cleans up environments older than 72 hours

GitHub Deployments are created for each environment, so you get the native "View deployment" button and status badges on the PR.

## Install

Click **Launch in SuperPlane** above, or go to:

```
https://app.superplane.com/install?repo=github.com/superplanehq/app_preview-env-digitalocean
```

The install wizard will ask you to:

1. **Connect integrations** — GitHub and DigitalOcean
2. **Select your repository** — the GitHub repo to watch for pull requests
3. **Pick an SSH secret** — a SuperPlane secret containing the private key for accessing droplets
4. **Choose droplet settings** — region, size, and image (defaults: nyc1, s-1vcpu-1gb, ubuntu-24-04-x64)

### Prerequisites

Before installing, make sure you have:

- An SSH key registered on DigitalOcean
- The corresponding private key stored as a SuperPlane secret (key name: `private_key`)
- The SSH key fingerprint configured on the Create Droplet node after install

## Customizing the setup script

The app includes `scripts/preview-setup.sh` in the **Files** tab. This script runs on each new droplet to install dependencies and start your application.

The default script sets up a Node.js app with nginx. You will need to edit it to match your own application — different runtime, different build steps, different service configuration.

The script receives these environment variables from the workflow:

- `PR_NUMBER` — the pull request number
- `REPO_URL` — the full clone URL of the repository (from the GitHub event)

## License

MIT

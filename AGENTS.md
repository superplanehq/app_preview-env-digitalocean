# Agent Guidelines — Preview Environments on DigitalOcean

This file provides context for AI agents (SuperPlane built-in agent, Cursor, or any external agent) operating on this canvas.

## What this app does

This app manages the full lifecycle of ephemeral preview environments for GitHub pull requests on DigitalOcean:

- **Provision** — create a droplet, deploy the app via SSH, run a health check, post the preview URL
- **Teardown** — delete the droplet and clean up memory on `/destroy` command or PR close
- **TTL** — a scheduled check reaps environments older than 24 hours

## Flows

### Deploy (`/start` command)
```
/start comment → Ack Deploy → Check Exists → Create Droplet → Create Deployment
  → Status: In Progress → Setup App (SSH) → Health Check
  → (pass) Save Environment → Post Preview URL → Status: Success
  → (fail) Setup Failed comment → Cleanup Failed Droplet → Status: Failure
```

If an environment already exists for the PR, it posts "Already Running" and stops.

### Destroy (`/destroy` command)
```
/destroy comment → Ack Destroy → Read Env → Delete Droplet
  → Destroyed Comment → Cleanup Memory → Status: Inactive
```

If no environment exists, it posts "No Env Found."

### PR Closed
```
PR Closed → Read Env (Close) → Delete Droplet (Close)
  → Destroyed Comment (Close) → Cleanup Memory (Close) → Status: Inactive (Close)
```

### TTL Check (scheduled)
```
TTL Check (schedule) → Read All Envs → Older than 24h?
  → (yes) TTL Delete → TTL Expired Comment → TTL Cleanup → Status: Inactive (TTL)
```

## Install parameters

These values were set during install and are substituted throughout the canvas:

| Parameter | Where it's used |
|-----------|----------------|
| `repository` | All GitHub nodes (triggers, comments, deployments, reactions) |
| `ssh_secret` | Setup App node SSH authentication |
| `region` | Create Droplet node |
| `size` | Create Droplet node |
| `image` | Create Droplet node |

## Memory

The app uses one namespace: `preview-envs`

Each entry stores:
- `pr_number` — the PR number (match key)
- `droplet_id` — DigitalOcean droplet ID
- `droplet_ip` — droplet public IP
- `app_name` — the repository name
- `created_at` — when the environment was created

## What's safe to change

- **`scripts/preview-setup.sh`** — the main customization point. Edit this to match your app's runtime, build steps, and service configuration. It receives `PR_NUMBER` and `REPO_URL` as environment variables.
- **Comment text** — any of the `github.createIssueComment` nodes (welcome, ready, destroyed, failed, etc.)
- **TTL duration** — the `check-ttl` (Older than 24h?) node's expression. Change `24` to your preferred hours.
- **Droplet tags** — tags on the `create-droplet` node
- **Health check URL** — the `http-health-check` node's URL path

## What not to change (without understanding the flow)

- **Memory namespace and match keys** — the deploy, destroy, close, and TTL flows all read/write `preview-envs` keyed on `pr_number`. Changing the namespace or keys breaks cross-flow coordination.
- **Edge wiring between core nodes** — the sequence matters. For example, `Save Environment` must run after `Create Droplet` because it reads the droplet ID from the output.
- **Integration references** — integration IDs are wired at install time. Don't change them unless re-connecting an integration.

## Common tasks

**Change the droplet size or region:**
Update the `create-droplet` node's `size` or `region` field.

**Change the TTL:**
Edit the `check-ttl` node expression. The default checks if `createdAt` is older than 24 hours.

**Add a Slack notification:**
Add a `slack.sendMessage` node after `Post Preview URL` or `Destroyed Comment`. Connect it to the same edge.

**Change the welcome comment:**
Edit the `welcome-comment` node's `body` field.

**Add a build step before the health check:**
The `setup-app` SSH node runs `scripts/preview-setup.sh`. Add your build steps to that script rather than adding more SSH nodes.

## Common issues

The most common failure point is the **SSH setup step** (`setup-app` node). When debugging a failed run, start here.

**SSH authentication fails:**
The secret referenced in the SSH node doesn't exist, has the wrong key name, or contains the wrong private key. Check that the secret selected during install exists in the org and the key name is `private_key`. Also verify the corresponding public key is registered on DigitalOcean and its fingerprint matches the one on the `create-droplet` node.

**Setup script fails:**
The default `scripts/preview-setup.sh` is written for a Node.js app. It will fail for any other stack. Users must edit this script in the **Files** tab to match their application — install the right runtime, build their app, configure their web server, and start their services. Check the SSH node's `stderr` output for the actual error.

**Health check returns non-200:**
The setup script includes a health check that curls `localhost:3000/`. If the app listens on a different port or path, update both the health check in `scripts/preview-setup.sh` and the `http-health-check` node URL.

**"Already Running" when the environment is actually gone:**
The memory entry still exists but the droplet was deleted outside of SuperPlane (e.g., from the DigitalOcean dashboard). Delete the stale entry from the **Memory** tab to allow a fresh deploy.

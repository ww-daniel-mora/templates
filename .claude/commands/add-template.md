# Add Portainer Template

Add a new service template to this repo. The repo powers the App Templates UI in Portainer at `https://portainer.morastein.com`.

## How the system works

Portainer fetches `templates-2.0.json` from GitHub. Each entry in the `templates` array renders as a card in the App Templates UI. When a user clicks Deploy, Portainer shows a form with inputs for each `env` item, substitutes the values into the stack file as `${VAR_NAME}`, then deploys the stack.

## Step 1 — Read the official docs

Before writing anything, fetch the service's official Docker documentation to find:
- The correct image name and tag
- Required environment variables
- Required volumes and their paths
- Required ports
- Any prerequisite setup (external secrets, config files, etc.)

## Step 2 — Create the stack file

Create `stacks/<service-name>/<service-name>-stack.yml`.

Rules:
- **Never hardcode** domain names, email addresses, passwords, API keys, or host-specific paths. Use `${VAR_NAME}` for everything deployment-specific.
- Sensitive values (passwords, API keys, tokens) must use Docker Swarm secrets (`external: true`), not environment variables. Reference them via `/run/secrets/<name>`.
- Use `traefik-public` as the external network.
- Define Traefik routing via deploy labels using `${SERVICE_FQDN}`:
  ```yaml
  - "traefik.http.routers.<service>.rule=Host(`${SERVICE_FQDN}`)"
  - "traefik.http.routers.<service>.middlewares=authelia@docker"
  - "traefik.http.routers.<service>.tls=true"
  - "traefik.http.routers.<service>.entrypoints=websecure"
  - "traefik.http.services.<service>.loadbalancer.server.port=<port>"
  ```
- Use `homeallowlist@file` instead of `authelia@docker` for LAN-only services.
- Timezone: use `${SERVICE_TZ}` or hardcode `America/New_York` only if TZ is genuinely not deployment-specific.

## Step 3 — Add the template entry to templates-2.0.json

Naming conventions:
- `name` field: `<SERVICE>_FQDN`, `<SERVICE>_CONFIG_DIR`, `<SERVICE>_DATA_DIR`, `<SERVICE>_TZ`
- Variable names must match `${VAR_NAME}` usage in the stack file exactly.

Template structure:
```json
{
  "type": 2,
  "title": "Service Name",
  "description": "One sentence description of what the service does.",
  "note": "HTML shown in the deploy form. List ALL prerequisites here — Docker secrets that must exist, config files that must be on the host, DNS records needed. Example: <b>Before deploying:</b><ul><li>Create secret: <code>docker secret create my-secret /path/to/value</code></li></ul>",
  "platform": "linux",
  "logo": "https://raw.githubusercontent.com/ww-daniel-mora/templates/master/resources/<service>.svg",
  "categories": ["<category>"],
  "repository": {
    "url": "https://github.com/ww-daniel-mora/templates",
    "stackfile": "stacks/<service-name>/<service-name>-stack.yml"
  },
  "env": [
    {
      "name": "SERVICE_FQDN",
      "label": "Service FQDN",
      "description": "The fully qualified domain name for this service. Ensure DNS resolves to this host before deploying.",
      "default": "service.example.com"
    },
    {
      "name": "SERVICE_CONFIG_DIR",
      "label": "Config directory",
      "description": "Host path for persistent config files.",
      "default": "/opt/<service>/config"
    }
  ]
}
```

Rules for `env` entries:
- Every item needs `name`, `label`, `description`, and `default`.
- `default` uses `example.com` for domains — never real domains (public repo).
- `description` is shown as a tooltip — make it actionable ("Ensure DNS resolves before deploying", "Generate with openssl rand -hex 32").
- Use `"preset": true` to pass a fixed value without showing an input (e.g. a hardcoded image tag or internal path the user shouldn't change).
- Use `"select"` for fields with a fixed set of valid values (e.g. log levels, regions).
- Secrets handled via Swarm secrets do NOT need an env entry — they're external, not user-supplied at deploy time.

## Step 4 — Validate and commit

```bash
python3 -m json.tool templates-2.0.json > /dev/null && echo "valid" || echo "INVALID"
git add stacks/<service>/ templates-2.0.json
git commit -m "feat(<service>): add Portainer template"
git push origin master
```

## What NOT to put in this repo

- Real domain names (use `example.com`)
- Real email addresses
- Actual secret values or passwords
- Host-specific paths that aren't configurable (make them `env` vars)
- Traefik file provider configs (`.yaml` files for `/opt/traefik/config/`) — those are operational files, not templates

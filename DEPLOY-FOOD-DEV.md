# Deploying food-dev.YOUR-DOMAIN

A standalone test deployment of the EN/ES translation feature, running
alongside your live instance on the same host, backed by a **copy** of your
live database — not the live database itself.

Written against your actual setup: Caddy on `web`/`backend` docker networks
(defined in your main compose file, `internal: true` on `backend`), Caddy
doing the `/api/*` split itself rather than the bundled selfhost proxy
container.

## Worth knowing before you start

- **Version gap**: your live instance is `v3.0.10`; this branch is based on
  current (`v4.0.3`+). Between those, both the search backend and the job
  queue infrastructure changed — not just a data migration.
  `prisma migrate deploy` (run automatically by
  `scripts/selfhost-startup.sh`) should walk a restored `v3.0.10`-era
  database through the full chain since it's one continuous migration
  history, but **test with an empty database first** (steps 1–4 below)
  before restoring your real data (step 5), so a migration hiccup shows up
  on throwaway data, not your backup.
- **Resources**: this adds ~1.5GB of `mem_limit` budget on top of whatever
  else is already running on this host. Check `free -h` first. Tear
  down with `docker compose -f docker-compose.food-dev.yml -p food-dev down
  -v` any time to reclaim it — nothing here touches your live stack's
  containers, volumes, or networks.
- **The two Dockerfiles `api`/`static` build from
  (`docker/selfhost-api.Dockerfile` / `docker/selfhost-static.Dockerfile` in
  the `RecipeSage` fork) are a reconstruction, not the official image build
  pipeline** — I couldn't find where `julianpoy/recipesage-selfhost:api-vX`/
  `:static-vX` are actually built. I ran the underlying `nx` build commands
  directly and they produce correct output, but the full `docker build` has
  not been run end-to-end (this session's sandbox blocks Docker Hub pulls).
  Treat step 3 below as the real first test and watch the logs.

## 1. Secrets

Create a `.env` file next to `docker-compose.food-dev.yml` (same directory —
Compose reads it automatically) with fresh values, distinct from your live
secrets:

```bash
cat > .env <<'EOF'
recipesage_fooddev_pg_pwd=<generate a new random password>
recipesage_fooddev_secret=<generate a new random secret>
EOF
```

## 2. DNS

Point `food-dev.YOUR-DOMAIN` (A/AAAA) at the same IP `YOUR-DOMAIN` uses.

## 3. Caddy

Add a block to your existing Caddyfile, right next to your `food.YOUR-DOMAIN`
block — same pattern, new domain and container names:

```
# --- RecipeSage (food-dev) ---
food-dev.YOUR-DOMAIN {
	handle_path /api/* {
		reverse_proxy recipesage-fooddev-api:3000
	}
	handle {
		reverse_proxy recipesage-fooddev-static:80
	}
}
```

Then reload Caddy however you normally do (e.g. `docker exec <your-caddy-container> caddy reload --config /etc/caddy/Caddyfile` if it's containerized).

(No `/grip/*` websocket route — matching your live `food.YOUR-DOMAIN`
block, which doesn't have one either.)

## 4. Bring up the stack (empty database first)

```bash
docker compose -f docker-compose.food-dev.yml -p food-dev up -d --build
docker compose -f docker-compose.food-dev.yml -p food-dev logs -f
```

This is the real test of the two reconstructed Dockerfiles — watch it
rather than walking away. Expect: a several-minute `api`/`static` build
from source, then `recipesage-fooddev-libretranslate` downloading its
`en`/`es` models. If the build fails, send me the error rather than
guessing further — that's exactly the scenario the caveat above flagged.

Once it's up, open `https://food-dev.YOUR-DOMAIN` — you should get a
working, empty RecipeSage instance. Register a throwaway account and
confirm the app itself works (create a recipe, use the EN/ES toggle) before
touching your real data at all.

## 5. Restore a copy of your live database

Once step 4 checks out, pull in real data to test against:

```bash
docker exec recipesage_postgres pg_dump -U recipesage_selfhost -d recipesage_selfhost \
  --no-owner --no-acl | gzip > food-dev-import.sql.gz

docker compose -f docker-compose.food-dev.yml -p food-dev exec -T recipesage-fooddev-postgres \
  psql -U recipesage_food_dev -d recipesage_food_dev -c \
  "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = 'recipesage_food_dev' AND pid <> pg_backend_pid();"

gunzip -c food-dev-import.sql.gz | docker exec -i recipesage_fooddev_postgres \
  psql -U recipesage_food_dev -d recipesage_food_dev

rm food-dev-import.sql.gz
```

A handful of `ERROR: role "recipesage_selfhost" does not exist` lines on
`GRANT`/`OWNER TO` statements are expected (from `--no-owner --no-acl`) and
safe to ignore; anything else (failed `CREATE TABLE`/`COPY`) is not — stop
and show me the output.

## 6. Re-run the migration + reindex against the restored data

```bash
docker compose -f docker-compose.food-dev.yml -p food-dev restart recipesage-fooddev-api
docker compose -f docker-compose.food-dev.yml -p food-dev logs -f recipesage-fooddev-api
```

Watch for the full migration chain completing and the search-index rebuild
step before testing.

## 7. Test

Log in with an account from your restored data, open a recipe, use the
Original/EN/ES toggle. Full checklist is in the `RecipeSage` PR's "How to
test" section (cache population, cache-hit reuse, staleness re-translation,
the unavailable-translation error state).

## Tearing down / redoing

```bash
docker compose -f docker-compose.food-dev.yml -p food-dev down -v
```

Removes food-dev's containers and volumes (including its database). Your
live stack, its data, and its networks are entirely untouched — this only
ever *joins* your existing `web`/`backend` networks, never modifies them.

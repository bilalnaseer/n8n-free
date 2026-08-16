# Updating n8n on Google Cloud

Instructions for updating an existing n8n install that was set up using the
[main guide](../README.md). Your workflows and credentials live in the `n8n_data`
Docker volume, not in the container — so updating means replacing the container
while leaving the volume untouched.

> Replace `your-domain.com` throughout with your actual domain, and make sure the
> `docker run` command in Step 6 matches the one you originally used.

---

## Step 1 - Create a backup

Stop the container first. n8n uses SQLite, and copying the database while it is
being written to can produce a backup that looks fine but fails to restore.

```bash
mkdir -p ~/backups

sudo docker stop n8n

sudo docker run --rm -v n8n_data:/data -v ~/backups:/backup alpine \
  tar czf /backup/n8n-pre-upgrade.tar.gz -C /data .

sudo docker start n8n
```

## Step 2 - Verify the backup

```bash
ls -lh ~/backups/
sudo tar tzf ~/backups/n8n-pre-upgrade.tar.gz | head -20
```

The listing **must** contain `./database.sqlite` (your workflows) and `./config`
(your encryption key). If either is missing, the backup is not usable — do not
continue.

## Step 3 - Take a Compute Engine snapshot

Compute Engine → Disks → your n8n boot disk → **Create snapshot**

This captures the whole machine and is the easiest way to roll back. n8n's database
migrations are one-way: once a newer version upgrades the schema, the old version
**cannot** start against it. The snapshot is your only route back.

## Step 4 - Pull the newer image

This does not affect the running container — n8n keeps serving while it downloads.

```bash
sudo docker pull docker.n8n.io/n8nio/n8n:latest
```

## Step 5 - Swap out the container

```bash
sudo docker stop n8n
sudo docker rm n8n
```

Removing the container does **not** delete your data. Everything is in the
`n8n_data` volume.

## Step 6 - Recreate on the new image

Use the exact same flags you originally installed with — only the image tag changes.

```bash
sudo docker run -d --restart unless-stopped \
  --name n8n \
  -p 5678:5678 \
  -e N8N_HOST="n8n.your-domain.com" \
  -e N8N_PROTOCOL="https" \
  -e WEBHOOK_URL="https://n8n.your-domain.com/" \
  -v n8n_data:/home/node/.n8n \
  docker.n8n.io/n8nio/n8n:latest
```

⚠️ Check `-v n8n_data:/home/node/.n8n` character by character. A typo in the volume
name makes Docker silently create a new empty volume, and n8n will greet you with a
fresh setup screen. That looks like total data loss but isn't — fix the typo and
re-run.

## Step 7 - Watch the migration

```bash
sudo docker logs -f n8n
```

You'll see `Starting migration ...` / `Finished migration ...` lines stepping through
each schema change between your old version and the new one. On a jump of several
months expect a minute or two, and the UI will not respond until they finish.

**Do not interrupt this** — a half-applied migration is the one case where you'd
actually need that snapshot. Wait for `Editor is now accessible via:` before touching
anything. Ctrl+C stops following the logs (it does not stop the container).

## Step 8 - Check the version

```bash
sudo docker exec n8n n8n --version
sudo docker ps
```

`docker ps` should show the container as `Up`, not restarting.

---

## Step 9 - Verify your workflows survived

**Do not skip this.** Compare the number of workflows the UI shows against what is
actually in the database:

```bash
sudo docker run --rm -v n8n_data:/data alpine sh -c \
  'apk add --no-cache sqlite >/dev/null 2>&1
   echo "--- workflows in database ---"
   sqlite3 /data/database.sqlite "SELECT id, name, active FROM workflow_entity;"
   echo "--- ownership links (must not be empty) ---"
   sqlite3 /data/database.sqlite "SELECT COUNT(*) FROM shared_workflow;"'
```

If the two numbers match what you see in the browser, you're done.

If workflows are listed in the database but **missing from the UI**, see the
troubleshooting section below.

Also open each workflow and confirm its nodes still render. A large version jump can
deprecate node parameters, which shows up as a broken node inside a workflow rather
than as any kind of error during the upgrade.

---

# Troubleshooting: workflows missing after the update

## Symptom

The update completes with no errors, the new version starts fine, you log in
normally — but the workflow list is **empty**.

## Cause

Your workflows are still in the database and nothing was lost. Modern n8n versions
organise workflows into **projects**, and the UI doesn't read `workflow_entity`
directly — it asks "which workflows are shared with my projects?" and renders that.
That link lives in the `shared_workflow` table.

When upgrading across the version that introduced projects, the migration can create
your personal project but fail to link your existing workflows to it. The workflows
become orphaned: present on disk, owned by nobody, invisible in every view. Nothing
errors, and the logs look clean.

## Diagnosis

Run the diagnostic below. n8n can stay running — these are read-only queries.

```bash
sudo docker run --rm -v n8n_data:/data alpine sh -c \
  'apk add --no-cache sqlite >/dev/null 2>&1
   echo "--- workflows ---"
   sqlite3 /data/database.sqlite "SELECT id, name, active FROM workflow_entity;"
   echo "--- projects ---"
   sqlite3 /data/database.sqlite "SELECT id, name, type FROM project;"
   echo "--- users ---"
   sqlite3 /data/database.sqlite "SELECT id, email, firstName, lastName FROM user;"
   echo "--- project_relation (links users to projects) ---"
   sqlite3 /data/database.sqlite "SELECT * FROM project_relation;"
   echo "--- shared_workflow (links workflows to projects) ---"
   sqlite3 /data/database.sqlite "SELECT * FROM shared_workflow;"'
```

Read the result like this:

| What you see | What it means |
|---|---|
| Workflows listed, `shared_workflow` **empty** | Orphaned workflows — apply the fix below |
| Workflows listed, `shared_workflow` has rows | Data is fine; it's a UI/login issue — see "Other causes" |
| **No** workflows listed at all | Wrong volume mounted, or genuinely empty — restore your backup |

Before assuming the worst, confirm the container is mounting the volume you think
it is:

```bash
sudo docker inspect n8n --format '{{json .Mounts}}'
sudo docker volume ls
```

A healthy `database.sqlite` is well over 1MB. A fresh, empty install is only a few
hundred KB — file size alone tells you whether your data is there:

```bash
sudo docker exec --user node n8n ls -la /home/node/.n8n
```

## The fix

This links every unlinked workflow to your personal project. It works out the IDs
itself, so there is nothing to substitute.

**Stop n8n first** — never write to the database while it is running.

```bash
sudo docker stop n8n

sudo docker run --rm -i -v n8n_data:/data alpine sh -s <<'OUTER'
apk add --no-cache sqlite >/dev/null 2>&1

# safety copy before writing
cp /data/database.sqlite /data/database.sqlite.bak-preinsert

sqlite3 /data/database.sqlite <<'SQL'
INSERT INTO shared_workflow (workflowId, projectId, role)
SELECT w.id,
       (SELECT id FROM project WHERE type = 'personal' LIMIT 1),
       'workflow:owner'
FROM workflow_entity w
WHERE w.id NOT IN (SELECT workflowId FROM shared_workflow);

INSERT INTO shared_credentials (credentialsId, projectId, role)
SELECT c.id,
       (SELECT id FROM project WHERE type = 'personal' LIMIT 1),
       'credential:owner'
FROM credentials_entity c
WHERE c.id NOT IN (SELECT credentialsId FROM shared_credentials);
SQL

echo "--- workflows now linked ---"
sqlite3 /data/database.sqlite "
  SELECT w.name, p.name, sw.role
  FROM shared_workflow sw
  JOIN workflow_entity w ON w.id = sw.workflowId
  JOIN project p ON p.id = sw.projectId;"
OUTER

sudo docker start n8n
```

Paste it as a single block. The `-i` flag is required — without it the container
receives no stdin and the whole thing is silently discarded.

You should see your workflows listed against your project name. If you get an error
instead, stop and restore from backup rather than re-running.

Then hard-refresh the browser (Ctrl+Shift+R / Cmd+Shift+R). Your workflows will
reappear under **Personal**.

⚠️ They come back **inactive**. Re-enable anything that runs on a schedule or webhook,
or it will silently stop firing.

Once everything looks right, remove the safety copy:

```bash
sudo docker run --rm -v n8n_data:/data alpine rm /data/database.sqlite.bak-preinsert
```

### Notes on the fix

- This assumes a **single-user install** with one personal project. If you have
  multiple users, `LIMIT 1` will assign everything to whichever personal project
  comes first — link the workflows explicitly by ID instead.
- The credentials half is included because the same migration handles credential
  ownership. It's a no-op if you have no credentials stored.
- If a `FOREIGN KEY` error appears, the project or workflow IDs don't line up. Stop
  and diagnose rather than retrying.

## Other causes of "missing" workflows

Work through these before touching the database:

1. **Browser cache.** n8n caches the workflow list aggressively. Hard-refresh, or try
   a private window.
2. **Wrong view.** Check **Overview** in the sidebar rather than a filtered or
   archived view, and clear any active search filters.
3. **Wrong account.** If the update prompted you to create an owner account again,
   you're logged into a new user with an empty project. Check the `user` table in the
   diagnostic above — more than one row means you should log in with your original
   email.
4. **Wrong volume.** Covered in Diagnosis above.

---

# Restoring from backup

If the update goes wrong, **do not** try to run the old image again — the schema has
already moved forward and the old version will refuse to start.

**From the Compute Engine snapshot (easiest):** Compute Engine → Snapshots → your
snapshot → Create disk, then attach it as the boot disk or create a new VM from it.
This rolls the entire machine back.

**From the tarball:**

```bash
sudo docker rm -f n8n
sudo docker volume rm n8n_data && sudo docker volume create n8n_data

sudo docker run --rm -v n8n_data:/data -v ~/backups:/backup alpine \
  tar xzf /backup/n8n-pre-upgrade.tar.gz -C /data

# then re-run the docker run command from Step 6 with your OLD version tag
```

---

# General advice

- **Pin your version** instead of using `latest`. With `latest` you have no idea what
  you're jumping to and no clean version to name when rolling back. Use an explicit
  tag such as `docker.n8n.io/n8nio/n8n:1.128.0`.
- **Read the [release notes](https://github.com/n8n-io/n8n/releases)** between your
  current version and the target, and skip any release flagged as breaking.
- **Update in smaller hops.** Jumping many versions at once runs many migrations
  together and makes problems like the one above far more likely.
- **Don't use Watchtower** or any auto-updater with n8n. Unattended updates plus
  one-way migrations plus occasional breaking releases is how you discover your
  automations broke overnight.
- **Back up your encryption key.** The `config` file inside the volume encrypts every
  stored credential. Lose it and every saved credential is unrecoverable:
  ```bash
  sudo docker exec n8n cat /home/node/.n8n/config
  ```

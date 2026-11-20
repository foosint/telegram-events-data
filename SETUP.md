# Setup: telegram-events-data (publishing with a deploy key)

This repository only holds the **published output** of [telegram-events](https://github.com/<GITHUB_USER>/telegram-events).
The publishing machine (the Raspberry Pi) commits and pushes it automatically after each run that changed the
output. Pushing uses a **deploy key**: an SSH key with write access to **this repository only**. It isn't tied to
your GitHub account and can be revoked at any time without affecting anything else.

**Placeholders**

| Placeholder | Meaning | Example |
|---|---|---|
| `<USER>` | Linux user on the Pi that runs telegram-events | `pi` |
| `<PROJECT_DIR>` | The telegram-events code checkout | `/home/<USER>/telegram-events` |
| `<DATA_REPO_DIR>` | This repository, cloned **next to** `<PROJECT_DIR>` | `/home/<USER>/telegram-events-data` |
| `<PI_HOST>` | Hostname or IP of the Pi | `raspberrypi.local` |

```
/home/<USER>/
  telegram-events/          <PROJECT_DIR>     code (normal HTTPS clone)
  telegram-events-data/     <DATA_REPO_DIR>   this repo (SSH clone, deploy key)
```
The code finds this repo via `[publish] repo_dir = "../telegram-events-data"` in each profile's `profile.toml`.
Don't nest the two repos, and don't rename this folder.

---

## 1. Repository on GitHub (once)
1. Create `<GITHUB_USER>/telegram-events-data` as a **public** repository. The road-alerts site reads the files
   through `raw.githubusercontent.com`, which only works without login for public repos.
2. Push the first commit from your PC with your normal GitHub login (HTTPS). The repo must not be empty, so
   the Pi has a branch to clone and push to:
   ```sh
   cd telegram-events-data
   git remote add origin https://github.com/<GITHUB_USER>/telegram-events-data.git
   git add README.md SETUP.md
   git commit -m "Initial commit"
   git push -u origin main
   ```
   Add `road-alerts/` to that commit only if you want sample data published. It contains original chat
   message texts. Otherwise the Pi's first run creates it.

## 2. Deploy key (on the Pi)
1. **Create the key.** It has no passphrase, because cron uses it unattended:
   ```sh
   ssh-keygen -t ed25519 -N "" -C "<USER>@<PI_HOST>: telegram-events-data publish" \
     -f /home/<USER>/.ssh/telegram_events_data_deploy
   chmod 600 /home/<USER>/.ssh/telegram_events_data_deploy
   cat /home/<USER>/.ssh/telegram_events_data_deploy.pub
   ```
   The private key (the file without `.pub`) never leaves the Pi.
2. **Register it on GitHub.** Open *<GITHUB_USER>/telegram-events-data → Settings → Deploy keys → Add deploy key*.
   - Title: `pi publish (<PI_HOST>)`
   - Key: paste the `.pub` line
   - **Tick "Allow write access"**, then click *Add key*.
3. **Test the key:**
   ```sh
   ssh -i /home/<USER>/.ssh/telegram_events_data_deploy -o IdentitiesOnly=yes -T git@github.com
   ```
   Expected: `Hi <GITHUB_USER>/telegram-events-data! You've successfully authenticated, but GitHub does not provide shell access.`
   Answer `yes` once to the host-key question, or let telegram-events accept it automatically.

## 3. Clone this repository on the Pi

A deploy key is an SSH key, so **pushes must go over SSH**. GitHub never accepts a deploy key over HTTPS.
Reading is different: the repo is public, so cloning and pulling over HTTPS need no key.

**Option A: everything over SSH (recommended)**
```sh
cd /home/<USER>
GIT_SSH_COMMAND="ssh -i /home/<USER>/.ssh/telegram_events_data_deploy -o IdentitiesOnly=yes" \
  git clone git@github.com:<GITHUB_USER>/telegram-events-data.git
```

**Option B: clone/pull over HTTPS, push over SSH**
```sh
cd /home/<USER>
git clone https://github.com/<GITHUB_USER>/telegram-events-data.git
git -C telegram-events-data remote set-url --push origin git@github.com:<GITHUB_USER>/telegram-events-data.git
```

Check the remotes:
```sh
git -C <DATA_REPO_DIR> remote -v
# origin  git@github.com:<GITHUB_USER>/telegram-events-data.git (fetch)     <- or https://… with option B
# origin  git@github.com:<GITHUB_USER>/telegram-events-data.git (push)      <- must be git@github.com:
```
You don't need to configure the key in git. telegram-events passes it to every pull and push itself (see section 4).

## 4. Configure telegram-events to publish
In `<PROJECT_DIR>`:

1. **The profile's `[publish]` section,** e.g. `profiles/road-alerts/profile.toml`. These are the defaults:
   ```toml
   [publish]
   repo_dir = "../telegram-events-data"
   ssh_key = "~/.ssh/telegram_events_data_deploy"
   git_name = "telegram-events bot"
   git_email = "telegram-events-bot@users.noreply.github.com"

   [output]
   dir = "../telegram-events-data/road-alerts"   # must be inside repo_dir
   ```
2. **Machine-wide switch** in `<PROJECT_DIR>/.env`, **only on the Pi**:
   ```sh
   PUBLISH=1
   ```
   Other machines, like your dev PC, leave it unset. Each profile folder then has exactly one writer.
3. **Test it:**
   ```sh
   cd <PROJECT_DIR>
   uv run tg-events -p road-alerts export --publish
   ```
   It prints `publish: pushed (1 commit)`, or `nothing to publish` when the remote already has the same files.

From then on, every `tg-events run` (cron, every 10 minutes) that changed the output commits **only the
profile's folder** and pushes it:
```
[road-alerts] … 42 events (1 new, telegram.json written), publish: pushed (1 commit)
```
Commit messages look like `telegram/road-alerts: 3 new events`, or `… update (window/archive)` when events
only aged out of the window.

## 5. Check from outside
```sh
curl -s https://raw.githubusercontent.com/<GITHUB_USER>/telegram-events-data/main/road-alerts/telegram.json | head -c 300
```
`raw.githubusercontent.com` caches files for up to about 5 minutes, so a fresh push can take a moment to show up.

## 6. Troubleshooting

| Message / symptom | Cause | Fix |
|---|---|---|
| `!! publish skipped: … deploy key only works over SSH` | The push URL is `https://…` | `git -C <DATA_REPO_DIR> remote set-url --push origin git@github.com:<GITHUB_USER>/telegram-events-data.git` |
| `!! publish skipped: deploy key … not found` | Wrong `ssh_key` path | Fix `[publish] ssh_key` in `profile.toml` |
| `!! publish skipped: … is not a git clone` | `repo_dir` points to the wrong folder, or the data repo isn't cloned yet | Clone as in section 3, next to `<PROJECT_DIR>` |
| `!! publish skipped: output dir … is not inside …` | `[output] dir` isn't inside `[publish] repo_dir` | Make `dir` a subfolder of `repo_dir` |
| `!! publish failed … Permission denied (publickey)` | The key isn't registered, or it's registered **without write access** | Section 2.2: re-add the key with "Allow write access" |
| `!! publish failed … ERROR: The key you are authenticating with has been marked as read only` | Write access not ticked | Same as above |
| `!! publish failed … rejected (fetch first)` | Someone else pushed. It is handled automatically with `pull --rebase` on the next run | Make sure only the Pi publishes (`PUBLISH=1` on one machine only) |
| `publish: disabled` | `PUBLISH=1` is missing from `.env` | Add it (on the Pi only) |
| The Pi is offline | The push fails, but the commit stays local | Pushed automatically on the next successful run |

## 7. Maintenance
- **Revoke or rotate the key:** delete it under *Settings → Deploy keys*, then repeat section 2 with a new key.
  Nothing else changes, as long as the file name stays the same.
- **Move to a new Pi:** create a new key there (section 2) and remove the old one on GitHub. Don't copy private keys between machines.
- **Repository size:** each push adds a small diff, because the file has one event per line. Over a year this
  stays in the low megabytes.
- **A new profile:** it gets its own folder here (`[output] dir = "../telegram-events-data/<profile>"`) and
  can use the same deploy key. For a separate repository, give the profile its own repo, its own deploy key
  (`[publish] ssh_key`) and its own `repo_dir`.

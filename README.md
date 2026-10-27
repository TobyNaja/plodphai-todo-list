# Plod Phai Todo — Todo List workload

A small, self-contained task manager used as a **demo workload** for the
Plod Phai Cloud DevSecOps pipeline. CI in this repository tests, scans, builds
and pushes the image to ECR, then rolls it into the running EC2 instances. The
infrastructure itself (VPC, ALB, ASG, …) is created from the separate
`prodpai_cloud` repository (see [CI pipeline](#ci-pipeline)).

It is a static web app: HTML, CSS and vanilla JavaScript served by Nginx. It
has no backend, no database, no external API calls, and loads nothing from a CDN.

## Features

- Dashboard layout with a sidebar (status views and data actions), a greeting, and summary cards for **total / completed / pending** tasks plus a progress bar
- **Add tasks** with a title (required, trimmed, max 120 chars), an optional description, a priority (Low / Medium / High), and an optional due date
- **Edit** a task in a dialog that is pre-filled with its current values. Completion status is kept
- **Complete / reopen** tasks with a checkbox button. Counters update immediately
- **Delete** tasks after a confirmation dialog
- **Filter** by All / Pending / Completed and **search** title + description. The filters and search can be combined
- **Sort** by newest, oldest, priority, or due date (tasks without a due date go last)
- Badges for overdue and due-today tasks
- Empty states for "no tasks" and "no matches", inline validation errors, and toast feedback
- **Load sample tasks** (never duplicated) and **Clear all tasks** (with confirmation)
- Responsive from 360 px phones to desktop, with light and dark themes, visible focus rings, a skip link, and reduced-motion support

## Technologies

| Layer | Choice |
|---|---|
| UI | HTML5, CSS (custom properties, grid/flex), vanilla JavaScript (ES5-compatible), inline SVG icons |
| Storage | Browser `localStorage` (key `plodphai.todo.tasks.v1`) |
| Server | Nginx `nginxinc/nginx-unprivileged:1.30.6-alpine3.24-slim`, non-root (uid 101), port 8080 |
| Tests | Node.js ≥ 22 built-in test runner (`node:test`). There are **no npm dependencies** |

## Directory structure

```text
plodphai-todo-list/
├── index.html            # markup, icon sprite, dialogs
├── css/style.css         # theme + responsive layout
├── js/core.js            # pure logic: validation, filter/sort, storage (unit-tested)
├── js/app.js             # DOM rendering and event handling
├── assets/favicon.svg
├── nginx.conf            # full Nginx config (non-root, /health, security headers)
├── Dockerfile
├── .dockerignore
├── package.json          # test scripts only, no dependencies
├── .github/
│   ├── config.env        # ECR_REPOSITORY, read by the CI workflow
│   ├── scripts/rollout.sh # rolling instance swap (copy of prodpai_cloud/scripts/rollout_asg.sh)
│   └── workflows/ci.yml  # test/lint/security → build, scan, push to ECR → rollout
└── tests/
    ├── core.test.js      # unit tests for js/core.js
    ├── static.test.js    # file/markup/security/Dockerfile checks
    ├── e2e.test.js       # real-browser tests (headless Chrome/Edge)
    └── helpers/browser.js
```

## Run locally without Docker

Any static file server works. For example, from this directory:

```bash
python -m http.server 8081
# or: npx --yes http-server -p 8081   (needs internet the first time)
```

Open <http://localhost:8081>. Note that `/health` exists only in the Nginx
container. A plain static server returns 404 for it.

## Build and run with Docker

From this directory:

```bash
docker build -t todo-list:local .
docker run --rm -p 8081:8080 todo-list:local
```

- App: <http://localhost:8081>
- Health check: <http://localhost:8081/health>

```bash
curl -i http://localhost:8081/health    # HTTP/1.1 200 OK ... "ok"
```

The image also defines a Docker `HEALTHCHECK` that calls `/health`. For an
AWS ALB target group, use path `/health` and success code `200`. In
`prodpai_cloud` the instances run `docker run -p 80:8080`, so the target group
itself listens on port `80`.

## Run the tests

You need Node.js 22 or newer. Nothing has to be installed.

```bash
npm test               # all tests (same as: node --test)
npm run test:unit      # core + static checks only
npm run test:e2e       # browser tests only
```

- **Unit tests** (`core.test.js`) cover validation, trimming, create/edit/toggle/delete, counters, filters plus search, sorting, sample de-duplication, and recovery from malformed or throwing storage.
- **Static checks** (`static.test.js`) confirm that the required UI elements exist and that there is no `innerHTML`, `eval`, inline handlers or external URLs. They also check that Nginx listens on 8080 with `/health` and that the Dockerfile is pinned and non-root.
- **Browser tests** (`e2e.test.js`) start a local server and drive headless Chrome/Edge over the DevTools protocol. They create, edit, complete and delete tasks, check filters and search, and confirm persistence across reloads and that malformed `localStorage` doesn't crash the app. They are **skipped automatically** when no Chromium-based browser is found. Set `CHROME_PATH` to point at one.

## CI pipeline

`.github/workflows/ci.yml` runs on every push and pull request to `main`, and
can also be started by hand. On `main` it goes all the way: test, scan, build,
push the image to ECR, then roll that image into the running Auto Scaling group.

| Job | What it does |
|---|---|
| `Test` | `npm test` on Node 22, with `timeout-minutes: 10` so a stuck browser can't hang the run for hours |
| `Dockerfile lint` | Hadolint on `Dockerfile` (fails on warnings). Trivy catches security misconfig; this catches quality issues |
| `Security checks` | Gitleaks over the **full git history**, Semgrep (`p/javascript`, `p/dockerfile`, `p/secrets`) and a Trivy filesystem scan (vuln, misconfig, secret — CRITICAL/HIGH fail the job). Semgrep and Trivy results are also uploaded as SARIF to the repository's **Security** tab |
| `Build, scan and push image` | Needs the three jobs above. `docker build` → smoke test `/health` and `/` against the running container → Trivy image scan → CycloneDX SBOM artifact → push to ECR |
| `Roll the new image into the ASG` | `main` only. Writes the new tag to SSM and swaps the ASG instances one by one (see below) |

The image that gets pushed is the exact one that was just built, smoke tested
and scanned, never a rebuild. It is pushed under two tags: the commit SHA and
`latest`. The target repository name comes from `ECR_REPOSITORY` in
`.github/config.env`, and the region is `us-east-1` (`AWS_REGION` at the top of
the workflow). `ECR_REPOSITORY` must match `APP_ECR_REPOSITORY` in
`.github/config.env` of `prodpai_cloud`.

**Pull requests are tested and scanned but never push.** The AWS steps of the
build job carry `if: github.event_name != 'pull_request'`, so nothing from a PR
can reach the registry. Note that a **manual run (`workflow_dispatch`) from any
branch does push**, and will move `latest` onto that branch's build. The rollout
job, however, runs only for `refs/heads/main`, so a manual run from another
branch never changes what production serves.

### Rollout

The rollout changes only *which image the instances run*. It does not create or
delete any resource, does not touch Terraform and does not create launch
template versions. It does two things:

1. `aws ssm put-parameter --name /prod-pai/app/image_tag --value <commit SHA> --overwrite`
2. `bash .github/scripts/rollout.sh <asg>`

Instances read the tag from that SSM parameter when they boot and `docker pull`
it from ECR. `rollout.sh` does a rolling replacement by hand (AWS Academy denies
`autoscaling:StartInstanceRefresh`): it raises desired capacity by one, waits
for the new instance to be healthy in the target group, then terminates the old
instances one at a time, waiting for each replacement. If a new instance is not
healthy in time (45 × 20 s by default) it stops without touching the remaining
old instances, so the site stays on the previous version.

Before rolling, the job waits for the ASG to settle (all instances
`InService`), and a `prodpai-rollout` concurrency group keeps two rollouts from
overlapping when pushes land back to back.

`rollout.sh` is a copy of `scripts/rollout_asg.sh` in `prodpai_cloud` (used by
its `Launch Image` workflow for rollbacks). Change both together.

**First setup.** If the account has no ASG yet (`Infra Setup` in
`prodpai_cloud` has never run), the rollout job shows a warning and skips
itself — the image is already in ECR. Then run `prodpai_cloud` → Actions →
**Infra Setup** with `image_tag=latest`. Every push after that rolls out on its
own. If the warning appears although the ASG exists, this repository's secrets
point at a different AWS account.

**Rollback.** Use `prodpai_cloud` → Actions → **Launch Image** with an older
commit SHA. It runs the same two steps.

### Secrets

Settings → Secrets and variables → Actions:

| Secret | Why |
|---|---|
| `AWS_ACCESS_KEY_ID` | ECR push and rollout credentials |
| `AWS_SECRET_ACCESS_KEY` | ECR push and rollout credentials |
| `AWS_SESSION_TOKEN` | AWS Academy sessions expire roughly every 4 hours, so all three need replacing that often |

They must belong to the same AWS account as `prodpai_cloud`. Besides pushing to
ECR, they need `ssm:PutParameter` on `/prod-pai/app/image_tag` and the
Auto Scaling / ELB calls `rollout.sh` uses (`DescribeAutoScalingGroups`,
`SetDesiredCapacity`, `TerminateInstanceInAutoScalingGroup`,
`DescribeTargetHealth`). The AWS Academy `LabRole` session has all of these.

After a run on `main`, the summary shows the image reference that was pushed
and the result of the rollout.

## Where data is stored

Tasks are stored **only in the visitor's browser** in `localStorage`. They are
not stored in Docker, AWS, or any database, and they never leave the browser.
Each browser and each origin (host + port) has its own separate list.

## Resetting data

- In the app, click **Clear all tasks** and confirm.
- Or, in the browser DevTools console, run `localStorage.removeItem('plodphai.todo.tasks.v1')`.
- Clearing site data for the origin also removes the tasks.

## Known limitations

- No sync between devices or browsers, and no accounts (by design).
- Data is lost if the user clears site data. Private windows discard it on close.
- If storage is blocked, the app still works in memory and shows a warning.
- Dates use the browser's local time zone.
- The Content-Security-Policy and other security headers are sent by Nginx only, not by a plain static server.

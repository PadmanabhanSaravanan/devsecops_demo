# SonarQube (SonarCloud) + Snyk + GitHub Actions Pipeline Setup Guide

This repo's pipeline ([.github/workflows/complete-workflow.yml](.github/workflows/complete-workflow.yml)) runs three jobs on every push:

1. **build** — Maven build + unit tests + SAST scan via **SonarCloud**
2. **security** — SCA (dependency vulnerability) scan via **Snyk**
3. **zap_scan** — DAST scan via **OWASP ZAP** (targets a public demo site, no setup needed)

You need accounts/tokens for SonarCloud and Snyk before the pipeline can run successfully. Steps below.

---

## 0. Prerequisites: push this project to GitHub

This folder is not yet a git repository, and GitHub Actions only runs on GitHub. From the project root:

```bash
git init
git add .
git commit -m "Initial commit"
```

Create a new empty repo on GitHub (via the web UI), then:

```bash
git remote add origin https://github.com/<your-username>/<your-repo>.git
git branch -M main
git push -u origin main
```

> Note: the workflow's `security` and `zap_scan` jobs check out `master`/refs from `master` explicitly (`actions/checkout@master`, `ref: master`). If your default branch is `main`, either rename your branch to `master` or update the workflow (see [Common gotchas](#common-gotchas) below).

---

## 1. Set up SonarQube (SonarCloud)

The workflow uses **SonarCloud** (the SaaS version of SonarQube), not a self-hosted SonarQube server — you can see this from `-Dsonar.host.url=https://sonarcloud.io`.

1. Go to https://sonarcloud.io and sign in with your GitHub account.
2. Click **+ (Add)** → **Analyze new project**, and import your GitHub repository.
   - Choose the **GitHub Actions** analysis method (not Automatic Analysis) since we're using a custom Maven command.
3. Note down / set these two values — they must match the workflow:
   - **Organization key**: currently `dotnetgithubactionsorg` in the workflow ([complete-workflow.yml:17](.github/workflows/complete-workflow.yml#L17))
   - **Project key**: currently `dotnetgithubactionsproject`
   - You'll find your actual org key under **My Account → Organizations**, and the project key on the project's **Information** page. Update the workflow if yours differ:
     ```
     -Dsonar.projectKey=<your-project-key> -Dsonar.organization=<your-org-key>
     ```
4. Generate a token: **My Account → Security → Generate Tokens**. Give it a name, generate, and copy it (you won't see it again).
5. In your GitHub repo, go to **Settings → Secrets and variables → Actions → New repository secret**:
   - Name: `SONAR_TOKEN`
   - Value: the token you copied
6. `GITHUB_TOKEN` is provided automatically by GitHub Actions — no setup needed.

---

## 2. Set up Snyk

1. Go to https://snyk.io and sign up (the free tier works fine), authenticating via GitHub is easiest.
2. Get your API token: click your avatar (bottom-left) → **Account Settings** → **Auth Token** → copy it.
   (Or via CLI: `npm install -g snyk && snyk auth`.)
3. In your GitHub repo, add another secret:
   - Name: `SNYK_TOKEN`
   - Value: the token you copied
4. Nothing else is required — `snyk/actions/maven@master` auto-detects `pom.xml` and runs `snyk test` against it. The job has `continue-on-error: true`, so vulnerabilities found won't fail the pipeline (this repo is *intentionally* vulnerable — see [README.md](README.md)).

---

## 3. ZAP (DAST) — no setup needed

The `zap_scan` job scans a public demo target (`http://testphp.vulnweb.com/`) using `zaproxy/action-baseline`, so it works out of the box. If you later want it to scan your own app instead, change the `target:` value in [complete-workflow.yml:44](.github/workflows/complete-workflow.yml#L44) and make sure that app is deployed/reachable from GitHub's runners, and add a `.zap/rules.tsv` file if you want custom rule tuning (create an empty `.zap/rules.tsv` if you don't need overrides — the action expects the file to exist).

---

## 4. Run the pipeline

Once both secrets (`SONAR_TOKEN`, `SNYK_TOKEN`) are set and the branch naming matches:

```bash
git add .
git commit -m "Trigger pipeline"
git push
```

The workflow is set to `workflow_dispatch` (manual trigger only — it will not run automatically on push). To run it:

- **Via GitHub UI**: go to your repo's **Actions** tab → select "Build code, run unit test, run SAST, SCA, DAST security scans" in the left sidebar → click **Run workflow** → choose the branch → **Run workflow**.
- **Via GitHub CLI**: `gh workflow run complete-workflow.yml --ref master` (swap `master` for your branch name).

Then watch it run under your repo's **Actions** tab on GitHub.

---

## Common gotchas

- **`master` vs `main` branch mismatch**: `actions/checkout@master` in the `security` job checks out the *default* branch by default (that `@master` is the action's version pin, not your branch), but the `zap_scan` job explicitly pins `ref: master`. If your repo's default branch is `main`, that checkout step will fail. Simplest fix: rename your local/remote branch to `master`, or edit the workflow's `ref: master` → `ref: main`.
- **SonarCloud org/project key mismatch**: if `sonar.organization` or `sonar.projectKey` don't match what you created on sonarcloud.io, the `mvn sonar:sonar` step fails with a 403/404. Double-check both values.
- **Old checkout action versions**: `actions/checkout@master` is a floating, unpinned reference (using a branch instead of a version tag like `v4`). It works but isn't reproducible/best-practice — consider bumping it to `actions/checkout@v4` to match the other jobs.
- **Java/Maven version**: the build uses JDK 21 (Zulu), but `pom.xml` targets Java 1.8 bytecode ([pom.xml:10-11](pom.xml#L10-L11)) — this still compiles fine under JDK 21, just flagging it's an intentional mismatch, not a bug to "fix."
- **Secrets are per-repo**: if you fork or copy this repo, secrets don't carry over — you must re-add `SONAR_TOKEN` and `SNYK_TOKEN` in the new repo's settings.

---

## Quick checklist

- [ ] Code pushed to a GitHub repo
- [ ] Branch naming matches what the workflow checks out (`master`, or workflow edited to `main`)
- [ ] SonarCloud project created; org key + project key match the workflow
- [ ] `SONAR_TOKEN` secret added to GitHub repo
- [ ] Snyk account created; token generated
- [ ] `SNYK_TOKEN` secret added to GitHub repo
- [ ] Push triggers the workflow; all 3 jobs visible under the **Actions** tab

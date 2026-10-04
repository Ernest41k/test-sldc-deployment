# SDLC Demo: CI/CD with GitHub Actions and AWS EC2

A deliberately small project for teaching the **Software Delivery Lifecycle**.
A tiny Node.js "tasks" API goes from a developer's laptop to a live AWS EC2
server through an automated pipeline: **lint → test → build → deploy → verify**.

The app is simple on purpose. The point is the *delivery process*, not the code.

---

## 1. How the SDLC maps to this repo

| SDLC phase | What students see | Where it lives |
|---|---|---|
| **Plan** | An issue or ticket describes the change | GitHub Issues |
| **Code** | Feature branch, small commits | `src/` |
| **Review** | Pull request with a checklist; CI must pass before merge | `.github/pull_request_template.md` |
| **Lint** | Static analysis catches mistakes before anything runs | `eslint.config.js`, job `1. Lint` |
| **Test** | Unit tests (logic) + integration tests (HTTP API) + coverage | `tests/`, job `2. Test` |
| **Build** | One immutable, versioned artifact ("build once, deploy many") | job `3. Build` |
| **Release / Deploy** | Artifact is copied to EC2 and switched live | `deploy/deploy.sh`, job `4. Deploy to EC2` |
| **Verify** | Server-side health check + external smoke test | `deploy.sh` + `Smoke test` step |
| **Operate / Rollback** | Failed health check automatically restores the previous release | `deploy/deploy.sh` |
| **Monitor** | `/health`, `/api/version`, `journalctl` logs | `src/app.js` |

```
 Pull request ─► [1. Lint] ─► [2. Test] ─► [3. Build]                         (CI)
 Merge to main ─► [1. Lint] ─► [2. Test] ─► [3. Build] ─► [4. Deploy] ─► Smoke test (CI + CD)
                                                              │
                                              EC2: nginx :80 ─► node :3000 (systemd)
```

## 2. Project layout

```
.
├── src/
│   ├── app.js            # Express routes (/, /health, /api/version, /api/tasks)
│   ├── server.js         # Starts the HTTP server
│   └── tasks.js          # Pure business logic (validation, in-memory store)
├── tests/
│   ├── tasks.test.js     # Unit tests
│   └── app.test.js       # Integration tests (supertest)
├── deploy/
│   ├── ec2-setup.sh      # ONE-TIME Amazon Linux 2023 server setup (Node, nginx, systemd)
│   └── deploy.sh         # Runs on EC2 every deploy (release, health check, rollback)
├── .github/
│   ├── workflows/pipeline.yml
│   └── pull_request_template.md
└── eslint.config.js
```

## 3. Run it locally

Requires Node.js 22+.

```bash
npm install
npm run lint     # static analysis
npm test         # unit + integration tests with coverage
npm run dev      # http://localhost:3000 (auto-restarts on file changes)
```

## 4. Set up AWS EC2 (one time)

1. **Launch an instance** in the EC2 console:
   - AMI: **Amazon Linux 2023**
   - Type: `t2.micro` or `t3.micro` (free tier eligible)
   - Key pair: create a new one and download the `.pem` file
   - Security group inbound rules:
     - **HTTP (80)** from `0.0.0.0/0` (so anyone can see the app)
     - **SSH (22)** from `0.0.0.0/0` (GitHub-hosted runners use changing IPs; see *Security notes*)
   - Optional: paste the contents of `deploy/ec2-setup.sh` into **Advanced details → User data** to skip step 3.
2. *(Recommended)* Allocate an **Elastic IP** and associate it with the instance so the address survives a stop/start.
3. **Run the setup script** (skip if you used User data):
   ```bash
   scp -i my-key.pem deploy/ec2-setup.sh ec2-user@<EC2_PUBLIC_IP>:~
   ssh -i my-key.pem ec2-user@<EC2_PUBLIC_IP> "sudo bash ec2-setup.sh"
   ```
   Visiting `http://<EC2_PUBLIC_IP>` now shows a *502 Bad Gateway*. That's expected: nginx is running but no app has been deployed yet.

## 5. Connect GitHub to EC2 (one time)

1. Create a GitHub repository and push this project to the `main` branch:
   ```bash
   git init -b main
   git add .
   git commit -m "Initial commit"
   git remote add origin https://github.com/<you>/<repo>.git
   git push -u origin main
   ```
2. In the repo go to **Settings → Secrets and variables → Actions**:

   | Kind | Name | Value |
   |---|---|---|
   | **Secret** | `EC2_SSH_KEY` | Entire contents of your `.pem` file, including the `BEGIN`/`END` lines |
   | **Variable** | `EC2_HOST` | Public IP or DNS of the instance, e.g. `54.12.34.56` |
   | **Variable** | `EC2_USER` | `ec2-user` |

   *Teaching point:* secrets are encrypted and masked in logs; variables are plain configuration.
3. *(Optional, great for class)* **Settings → Environments → New environment** named `production`.
   Add yourself under **Required reviewers**. Every deploy now pauses for a manual approval,
   which shows the difference between **Continuous Delivery** (human approves) and **Continuous Deployment** (fully automatic).
4. *(Recommended)* **Settings → Branches → Add rule** for `main`: require a pull request and require the
   `1. Lint`, `2. Test` and `3. Build` checks to pass. Now nobody can ship untested code.

Push to `main` (or re-run the workflow from the **Actions** tab) and watch the pipeline deploy.
Then open `http://<EC2_HOST>`; the page shows the commit and pipeline run number that are live.

## 6. What happens during a deploy

1. **Build** packages `src/`, production `node_modules/`, `package.json` and a `build-info.json`
   (version, commit, run number, timestamp) into `release.tar.gz`.
2. **Deploy** copies the tarball and `deploy.sh` to the server over SSH and runs `deploy.sh`, which:
   - unpacks it into `/opt/sdlc-demo/releases/<timestamp>-<commit>/`
   - points the `/opt/sdlc-demo/current` symlink at the new release
   - restarts the `sdlc-demo` systemd service
   - polls `http://127.0.0.1:3000/health`; **if it never turns healthy, it re-points `current` at the previous release and fails the job (automatic rollback)**
   - deletes all but the 5 newest releases
3. **Smoke test** calls the public URL from the runner and checks that the live commit matches the one just built.

Useful commands on the server:

```bash
sudo systemctl status sdlc-demo          # is it running?
sudo journalctl -u sdlc-demo -f          # live application logs
ls -l /opt/sdlc-demo/                    # which release is 'current'?
ls /opt/sdlc-demo/releases/              # release history
```

## 7. Classroom exercises

1. **The happy path.** On a branch, change the `<h1>` text in `src/app.js`, open a PR,
   watch CI run (no deploy on PRs), merge, and watch it go live. Compare the commit on the page with the merge commit.
2. **Lint catches it.** Add an unused variable (`const x = 1;`) and push. Stage 1 fails and nothing later runs.
3. **Tests catch it.** Change the 100-character limit in `src/tasks.js` to 10. Stage 2 fails. Discuss: what would have happened without tests?
4. **Write the test first (TDD).** Add a `DELETE /api/tasks/:id` endpoint: write the failing test, then make it pass.
5. **Automatic rollback.** In `src/server.js`, crash only in production so tests still pass:
   ```js
   if (process.env.NODE_ENV === 'production') throw new Error('boom');
   ```
   Merge it. The deploy's health check fails, `deploy.sh` restores the previous release, and the site stays up.
   Then discuss why "it passed the tests" doesn't mean "it works in production".
6. **Manual rollback / roll forward.** Use `git revert` on the bad commit and let the pipeline redeploy. Compare that with
   switching the symlink by hand on the server.
7. **Approval gate.** Enable the `production` environment reviewer (section 5.3) and talk about who should approve releases.

## 8. Security notes (talk about these in class)

- Opening SSH to `0.0.0.0/0` keeps the demo simple but isn't what you'd do in production. Better options include
  **AWS Systems Manager (SSM) Run Command** with GitHub OIDC (no SSH port, no long-lived keys), a
  **self-hosted runner** inside the VPC, or **AWS CodeDeploy**.
- The `.pem` key lives only in GitHub Secrets. `.gitignore` blocks `*.pem` files so they can't be committed by accident.
- The site is served over plain HTTP. For real use, put it behind a domain with HTTPS (e.g. Let's Encrypt / certbot, or an ALB with ACM).
- Tasks are stored in memory and reset on every deploy. That's a good opening to discuss stateful apps and databases.

## 9. Ideas for extending the course

- Add a **staging** environment (second EC2 instance) that deploys before production.
- Containerize with **Docker** and push images to **Amazon ECR**.
- Provision the EC2 instance with **Terraform** or **CloudFormation** (Infrastructure as Code).
- Add **dependency scanning** (`npm audit`, Dependabot) and **CodeQL** to the CI stage.
- Send pipeline notifications to Slack or Teams.

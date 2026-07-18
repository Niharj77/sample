# Jenkins + Docker CI/CD Demo (Beginner Walkthrough)

A tiny, dependency-free Python app used to learn the full pipeline:
**Jenkins pulls code → builds a Docker image → pushes it to Docker Hub → deploys the container → GitHub webhook triggers it all automatically on every push.**

Files in this project:
- `app.py` — one-file Python web server (standard library only, no pip installs)
- `Dockerfile` — builds the app into a container image
- `Jenkinsfile` — the pipeline definition Jenkins will run
- This `README.md`

---

## 1. Install Java (Jenkins requires it)

On Ubuntu/Debian:
```bash
sudo apt update
sudo apt install -y fontconfig openjdk-17-jre
java -version
```

## 2. Install Jenkins

```bash
sudo apt update
sudo apt install -y curl gnupg
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  "https://pkg.jenkins.io/debian-stable binary/" | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
```

Get the initial admin password and unlock Jenkins in the browser:
```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Open `http://<server-ip>:8080`, paste the password, install "Suggested plugins".

## 3. Install Docker and let Jenkins use it

```bash
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
```
(The `jenkins` user needs Docker group access so pipeline steps can run `docker build`/`docker push`.)

## 4. Push this project to a GitHub repo

```bash
cd jenkins-docker-demo
git init
git add .
git commit -m "initial commit"
git branch -M main
git remote add origin https://github.com/<your-username>/<your-repo>.git
git push -u origin main
```

## 5. Add Docker Hub credentials in Jenkins

Jenkins → **Manage Jenkins → Credentials → (global) → Add Credentials**
- Kind: Username with password
- Username/password: your Docker Hub username and an access token (create one at hub.docker.com → Account Settings → Security)
- ID: `dockerhub-creds` (must match the ID used in the Jenkinsfile)

## 6. Edit the Jenkinsfile

Open `Jenkinsfile` and replace:
```groovy
DOCKERHUB_USER = 'YOUR_DOCKERHUB_USERNAME'
```
with your actual Docker Hub username, then commit and push the change.

## 7. Create the Jenkins pipeline job

Jenkins → **New Item → Pipeline** → name it `jenkins-docker-demo`
- Under **Pipeline**, choose "Pipeline script from SCM"
- SCM: Git
- Repository URL: your GitHub repo URL
- Branch: `*/main`
- Script path: `Jenkinsfile`
- Save, then click **Build Now**

Watch **Console Output** — you'll see it go through: Checkout → Build Docker Image → Push to Docker Hub → Deploy Container.

Verify:
```bash
docker ps                 # see the running container
curl http://localhost:8080
```
Check Docker Hub — a new repo/image should appear under your account.

## 8. Add a GitHub webhook (auto-trigger on push)

1. In the Jenkins job, enable **"GitHub hook trigger for GITScm polling"** under Build Triggers.
2. On GitHub: repo → **Settings → Webhooks → Add webhook**
   - Payload URL: `http://<jenkins-server-ip>:8080/github-webhook/`
   - Content type: `application/json`
   - Trigger: "Just the push event"
3. Save. Push any small change (e.g. edit `APP_VERSION` in `app.py`) and watch Jenkins start a new build automatically, without clicking "Build Now".

> If Jenkins is on your local machine (not a public server), GitHub can't reach it directly — use a tunnel like `ngrok http 8080` and use the ngrok URL as the webhook payload URL instead.

## What you're seeing end-to-end

1. You push code to GitHub.
2. GitHub webhook pings Jenkins.
3. Jenkins checks out the repo (Checkout stage).
4. Jenkins builds a new Docker image from the Dockerfile (Build stage).
5. Jenkins logs into Docker Hub and pushes the image (Push stage).
6. Jenkins stops the old container and starts a new one from the freshly pushed image (Deploy stage).
7. Refreshing `http://<server>:8080` shows the app running the latest version — fully automated.

## Troubleshooting tips
- `docker: permission denied` in Jenkins logs → the `jenkins` user isn't in the `docker` group yet, or Jenkins wasn't restarted after adding it.
- Webhook not firing → check GitHub's webhook "Recent Deliveries" tab for the response code Jenkins sent back.
- Port already in use → change `APP_PORT` in the Jenkinsfile or stop whatever else is using 8080.

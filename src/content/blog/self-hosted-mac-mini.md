---
title: "Homegrown CI/CD on Mac Mini + E-Ink Status Dashboard"
description: "How I built a local deployment framework with Docker, GitHub Actions, and a Raspberry Pi e-ink dashboard for real-time monitoring."
pubDate: 2025-08-16
tags:
  ["DevOps", "GitHub Actions", "CI/CD", "Docker", "Self-hosting", "Mac Mini"]
draft: false
---

# Tinkering My Way Into CI/CD

A month ago, I stopped reading DevOps tutorials and actually built something: a full deployment stack running on a **Mac Mini at home**, complete with automated CI/CD and a physical **e-ink dashboard** that shows me system status at a glance.

The interesting parts aren't the Docker configs—it's how everything connects: GitHub Actions triggers deployments via webhooks, Uptime Kuma monitors everything through WebSockets, and a Raspberry Pi e-ink display gives me ambient awareness of what's actually running.

---

## The Foundation

I built [mini-deploy-template](https://github.com/Lucas8448/mini-deploy-template) as a reusable skeleton for containerized apps. It's opinionated but minimal: GitHub Actions workflow, Docker Compose setup, health checks, and environment templates. The key insight was making it parameterized—each new service just needs its own `.env.production` file.

---

## The Pipeline

The CI workflow is straightforward: on push to main, it builds the Docker image, pushes to my local registry (`localhost:5001`), and triggers a deployment webhook. I also configured a cron schedule to run every 2 hours—even without code changes, this detects drift and ensures containers are healthy.

```yaml
env:
  REGISTRY: localhost:5001

schedule:
  - cron: "0 */2 * * *"

jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v3

      - name: Build & push
        run: |
          docker build -t $REGISTRY/${{ env.APP_NAME }}:latest .
          docker push $REGISTRY/${{ env.APP_NAME }}:latest

      - name: Trigger deployment
        run: curl -X POST ${{ secrets.DEPLOY_WEBHOOK_URL }}
```

The Mac Mini runs a self-hosted GitHub runner, so deployments happen directly on the box. Health checks are baked into both the Dockerfile and Compose config—if a container fails, Docker restarts it automatically.

---

## The Monitoring Stack

Here's where it gets interesting. The deployment webhook hits an n8n workflow that notifies me as soon as a container is deployed. But n8n does more than just deployments:

**Uptime Kuma** monitors every service endpoint via HTTP checks. When something changes state, it pushes events through a WebSocket feed. n8n ingests that feed and routes alerts to:

- Slack (for immediate notifications)
- A JSON endpoint on the Mac Mini (for the e-ink dashboard)

The **e-ink dashboard** is a Raspberry Pi with a low-power e-ink screen. It fetches the JSON endpoint every minute and renders current status: service name, health (green/yellow/red), and uptime. It's always-on, ultra-low power, and gives me that glanceable ambient feedback.

When the Mac Mini crashed during a system update, Uptime Kuma detected it within seconds. I got the Slack alert, and my e-ink dashboard showed exactly how long things had been down. That feedback loop pushed me to harden everything—better health checks and smarter restart policies.

---

## What I Learned

Building this forced me to think about resilience differently. It's not just "does it work?"—it's "does it recover?" The cron-scheduled CI runs catch drift. Health checks ensure containers restart. The e-ink display makes downtime visible, which motivates me to fix it faster.

**Next steps:**

- Automatic rollbacks: if a new deploy fails health checks, revert to the last working image
- Smarter error handling: Automated fixes with n8n and copilot
- Metrics integration: CPU/memory sparklines on the e-ink display

This project has become my go-to for deploying new apps fast and easy. If you want to build something similar, start with the [mini-deploy-template](https://github.com/Lucas8448/mini-deploy-template) and iterate from there.
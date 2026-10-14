# jenkins-cicd-pipeline-demo

A declarative Jenkins pipeline that takes a sample app from commit to a
running Kubernetes deployment: build → static analysis gate → image push →
Helm deploy. One pipeline exercising Jenkins, SonarQube, Docker and Helm
together, the way a real release pipeline is wired.

## Pipeline stages

1. **Checkout** — pulls the pipeline's own source (`checkout scm`)
2. **Build** — builds the Docker image via the multi-stage `Dockerfile`
3. **Code Quality (SonarQube)** — runs `sonar-scanner` against the
   `sonar-project.properties` config
4. **Quality Gate** — pipeline aborts if SonarQube's quality gate fails,
   so bad code never reaches deploy
5. **Push to Registry** — tags the image with the Jenkins build number and
   pushes it to a container registry (Docker Hub in this example)
6. **Deploy to Kubernetes via Helm** — `helm upgrade --install` against
   the `helm/sample-app` chart, injecting the freshly built image tag

## Why it's structured this way

- The **quality gate stage is separate from the scan stage** and set to
  `abortPipeline: true` — a failing gate stops the pipeline before an image
  is ever pushed, not after.
- Registry and kube credentials are pulled via Jenkins `withCredentials`
  blocks (`dockerhub-credentials`, `kubeconfig`) rather than hardcoded —
  nothing secret lives in this repo.
- The Docker image runs as a **non-root user** and defines a `HEALTHCHECK`,
  which the Helm chart's liveness/readiness probes also check via `/health`.
- Helm chart image repo/tag are overridden at deploy time with
  `--set image.tag=${IMAGE_TAG}`, so the chart itself never needs editing
  for every release — only the pipeline's build number changes.

## File layout

```
.
├── Jenkinsfile                          # the pipeline itself
├── Dockerfile                           # multi-stage build, non-root, healthcheck
├── sonar-project.properties             # SonarQube scan config
└── helm/
    └── sample-app/
        ├── Chart.yaml
        ├── values.yaml                  # replicas, resources, probes, ingress toggle
        └── templates/
            ├── deployment.yaml
            ├── service.yaml
            └── ingress.yaml             # disabled by default (ingress.enabled: false)
```

## Setting this up in Jenkins

1. Configure a SonarQube server under **Manage Jenkins → System** and name
   it to match `SONARQUBE_ENV` in the `Jenkinsfile`.
2. Add credentials in Jenkins:
   - `dockerhub-credentials` — username/password for your registry
   - `kubeconfig` — a Secret file credential holding your cluster's kubeconfig
3. Point a Pipeline job at this repo; Jenkins will pick up the `Jenkinsfile`
   automatically.
4. Replace `yourdockerhubuser/sample-app` in `Jenkinsfile` and
   `helm/sample-app/values.yaml` with your actual registry path.

## Notes

- `server.js` / the actual app source isn't included here — this repo
  focuses on the pipeline and deployment mechanics, which is the DevOps
  surface area it's meant to demonstrate.
- No real registry or cluster credentials are referenced anywhere in this
  repo; everything sensitive is pulled from Jenkins credential IDs.

# Catalogue CI

This directory contains the RoboShop `catalogue` service source code and the
Jenkins CI entrypoint. Its job is to build the Node.js application, create a
Docker image, push that image to AWS ECR, and trigger the matching CD job.

## Files

| File | Purpose |
| --- | --- |
| `Jenkinsfile` | Loads `jenkins-shared-library` and starts the CI pipeline. |
| `server.js` | Express API for catalogue, product, category, search, and health endpoints. |
| `package.json` | Node.js metadata, dependencies, and version used as the Docker image tag. |
| `Dockerfile` | Builds the catalogue container image. |
| `sonar-project.properties` | SonarQube project configuration. |
| `db/master-data.js` | MongoDB seed data and indexes. |

## Runtime Configuration

| Variable | Default | Purpose |
| --- | --- | --- |
| `CATALOGUE_SERVER_PORT` | `8080` | Port used by the Node.js service. |
| `MONGO_URL` | `mongodb://mongodb:27017/catalogue` | MongoDB connection string. |
| `GO_SLOW` | `0` | Optional delay for `/product/:sku` responses. |

## Local Run

Install dependencies:

```bash
npm install
```

Load MongoDB seed data:

```bash
mongosh < db/master-data.js
```

Run the service:

```bash
CATALOGUE_SERVER_PORT=8080 MONGO_URL=mongodb://localhost:27017/catalogue node server.js
```

Check health:

```bash
curl http://localhost:8080/health
```

## API Endpoints

| Method | Endpoint | Description |
| --- | --- | --- |
| `GET` | `/health` | Returns application and MongoDB status. |
| `GET` | `/products` | Returns all products. |
| `GET` | `/product/:sku` | Returns one product by SKU. |
| `GET` | `/products/:cat` | Returns products from one category. |
| `GET` | `/categories` | Returns all categories. |
| `GET` | `/search/:text` | Searches product name and description. |

## Docker

Build locally:

```bash
docker build -t roboshop/catalogue:local .
```

Run locally:

```bash
docker run --rm -p 8080:8080 \
  -e MONGO_URL=mongodb://host.docker.internal:27017/catalogue \
  roboshop/catalogue:local
```

## Jenkins CI Flow

The `Jenkinsfile` starts by loading the global Jenkins shared library:

```groovy
@Library('jenkins-shared-library') _
```

It then passes service values to the shared library:

```groovy
def configMap = [
    project: "roboshop",
    component: "catalogue"
]
```

Branch behavior:

| Branch | Behavior |
| --- | --- |
| Any branch except `main` | Calls `nodeJSEKSPipeline(configMap)`. |
| `main` | Stops CI and prints `Please follow the CR process`. |

The real CI stages are defined in:

```text
../jenkins-shared-library/vars/nodeJSEKSPipeline.groovy
```

Current CI stage order:

1. Read `version` from `package.json`.
2. Run `npm install`.
3. Run the unit-test placeholder `echo test`.
4. Skip the disabled Dependabot security gate.
5. Login to AWS ECR, build the Docker image, and push it.
6. Trigger `../catalogue-CD` with `appVersion` and `deploy_to=dev`.
7. Clean the Jenkins workspace.

## Image Version

The CI image tag comes from `package.json`:

```json
{
  "version": "1.1.0"
}
```

For this version, CI builds and pushes an image like:

```text
704887592073.dkr.ecr.us-east-1.amazonaws.com/roboshop/catalogue:1.1.0
```

## Jenkins Requirements

The Jenkins CI agent must have label:

```text
AGENT-CI
```

Required tools and plugins:

| Requirement | Why Needed |
| --- | --- |
| Node.js and npm | Install dependencies and test the service. |
| Docker | Build and push the service image. |
| AWS CLI | Login to ECR. |
| AWS Steps plugin or equivalent `withAWS` support | Use Jenkins AWS credentials in the pipeline. |
| Pipeline Utility Steps plugin | Provides `readJSON`. |
| Workspace Cleanup plugin | Provides `cleanWs()`. |

Required Jenkins credential:

| Credential ID | Type | Used For |
| --- | --- | --- |
| `aws-creds` | AWS credentials | Login and push to AWS ECR. |

Optional credentials and tools for currently disabled stages:

| Item | Used By |
| --- | --- |
| `github-pat` | Dependabot security gate if enabled. |
| `jq` | Dependabot alert parsing if enabled. |
| SonarQube scanner and Jenkins SonarQube config | Commented SonarQube stages if enabled. |
| Trivy | Commented image scan stage if enabled. |

## Handoff To CD

After a successful image push, CI triggers the CD job asynchronously:

```groovy
build job: "../${env.COMPONENT}-CD",
      wait: false,
      propagate: false,
      parameters: [
          string(name: 'appVersion', value: env.APP_VERSION),
          string(name: 'deploy_to', value: 'dev')
      ]
```

For `component = catalogue`, the downstream job name becomes:

```text
../catalogue-CD
```

Important behavior:

| Setting | Meaning |
| --- | --- |
| `wait: false` | CI does not wait for the CD job to finish. |
| `propagate: false` | CD failure does not fail the CI job. |
| `deploy_to: dev` | Automatic deployment target from CI is dev only. |

## Current Checks Before Running End To End

- Confirm the Jenkins CD job is actually named `catalogue-CD` relative to the CI job folder.
- Confirm CI ECR account `704887592073` is intended.
- Confirm CD Helm value `deployment.imageRepo` points to the same image registry, or that cross-account image pull access is configured.
- Replace the unit-test placeholder with `npm test` when real tests are ready.
- Enable SonarQube, Trivy, or Dependabot gates only after the required Jenkins tools and credentials exist.
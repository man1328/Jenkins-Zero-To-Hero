# End-to-End CI/CD Walkthrough — Java + Maven + SonarQube + Docker + Argo CD + Kind

A record of how this pipeline was built, every problem hit along the way, and the exact final state. Written 2026-11-24.

---

## 1. Final architecture and decisions

| Component | Where it runs | Access |
|---|---|---|
| GitHub fork | `https://github.com/man1328/Jenkins-Zero-To-Hero.git` (public, `main`) | HTTPS + PAT for writes |
| Docker Hub repo | `man1380/ultimate-cicd` (public) | `docker-cred` Jenkins credential |
| Jenkins | Docker container on this PC | `http://localhost:9090` |
| SonarQube | Docker container on this PC | `http://localhost:9000` |
| Kubernetes | Kind cluster `kind-cicd` on this PC | `kubectl` context `kind-cicd` |
| Argo CD | Installed in Kind, namespace `argocd` | `https://localhost:8082` (port-forward) |
| Demo app | Kind namespace `default` | `http://localhost:8081` (port-forward) |

**Why Kind on this PC instead of EC2:** AWS's current Free Tier gives new accounts up to $200 in credits that expire after 6 months — not a perpetual free EC2 instance. The only historically free sizes (t2.micro/t3.micro) have ~1 GB RAM; Jenkins + Maven + SonarQube need 4 GB+. A workable EC2 (t3.medium) is paid. The EC2 instance that was originally used for Jenkins had only 908 MB RAM, so Jenkins was moved local.

**Note on names:** GitHub username is `man1328`, Docker Hub username is `man1380`. The Jenkinsfile uses each in the right place. Your local SSH key (`~/.ssh/id_ed25519`) authenticates to GitHub as `man1328`; `~/.ssh/jenkins_ed25519` is for the EC2 host.

---

## 2. Prerequisites on this PC

- Docker (with at least 6–8 GB RAM available; this PC has 62 GB)
- `kubectl`, `kind`, `git`
- Ports used: 9000 SonarQube, 9090 Jenkins, 8082 Argo CD, 8081 app (8080 was already occupied)

---

## 3. Phase 1 — Fork and personalize the repository

Forked `iam-veeramalla/Jenkins-Zero-To-Hero` to `man1328/Jenkins-Zero-To-Hero`.

Important corrections to the original guide:

| Original | Correct value |
|---|---|
| Script path `.../Jenkinsfile` | `.../JenkinsFile` (capital **F**) |
| GitHub credential = username/password | **Secret text** (PAT), because Jenkinsfile uses `string(credentialsId: 'github', ...)` |
| Sonar URL `34.201.116.83:9000` | `http://localhost:9000` (after moving everything local) |
| Argo CD install URL `https://githubusercontent.com` | `https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml` |
| Cluster URL `https://default.svc` | `https://kubernetes.default.svc` |
| Argo CD exposed via LoadBalancer | Port-forward (Kind has no cloud LB) |

Edits made to `java-maven-sonar-argocd-helm-k8s/spring-boot-app/JenkinsFile`:

- Agent image: `cicd-maven-agent:local`
- Agent args: `--user root -v /var/run/docker.sock:/var/run/docker.sock --network=host`
- `SONAR_URL = "http://localhost:9000"`
- Sonar command uses the fully qualified plugin goal and `sonar.token`:
  `mvn org.sonarsource.scanner.maven:sonar-maven-plugin:5.9.0.7382:sonar -Dsonar.token=$SONAR_AUTH_TOKEN -Dsonar.host.url=${SONAR_URL}`
- `DOCKER_IMAGE = "man1380/ultimate-cicd:${BUILD_NUMBER}"` in **both** the Docker stage and the "Update Deployment File" stage
- `GIT_USER_NAME = "man1328"`

Edit to `spring-boot-app-manifests/deployment.yml`:

```yaml
image: man1380/ultimate-cicd:6
```

The regex `sed` in the pipeline rewrites this tag on every build.

---

## 4. Phase 2 — Kind cluster and Argo CD

```bash
kind create cluster --name cicd
kubectl cluster-info --context kind-cicd

kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**Critical:** use `--server-side --force-conflicts`. Plain `kubectl apply` adds the `kubectl.kubernetes.io/last-applied-configuration` annotation; the ApplicationSet CRD is ~1.57 MB, exceeding the 262,144-byte limit, so it is rejected and the applicationset-controller CrashLoopBackOffs.

If you hit that error on an existing cluster, install just the missing CRD:

```bash
kubectl apply --server-side --force-conflicts -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/v3.6.2/manifests/crds/applicationset-crd.yaml
```

Argo CD Application (created with `kubectl apply -f`):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: spring-boot-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/man1328/Jenkins-Zero-To-Hero.git
    targetRevision: main
    path: java-maven-sonar-argocd-helm-k8s/spring-boot-app-manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

---

## 5. Phase 3 — SonarQube

The original `sonarqube:lts-community` image gave version **9.9.8**, which SonarQube flags as "no longer active". Replaced it with the current Community Build:

```bash
docker rm -f sonar
docker run -d --name sonar -p 9000:9000 \
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
  --restart unless-stopped \
  sonarqube:community
```

`SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true` is used because the host has `vm.max_map_count=262144` and current docs want `524288`. Verified version: **26.11.0.134175**, status `UP`.

The token generated in the new instance must be pasted into the Jenkins `sonarqube` credential.

---

## 6. Phase 4 — Jenkins on this PC

Built a custom image because the stock Jenkins image has no Docker CLI:

`/home/mandog13/jenkins-docker/Dockerfile`
```dockerfile
FROM jenkins/jenkins:lts-jdk17
USER root
RUN apt-get update \
 && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends docker-cli \
 && rm -rf /var/lib/apt/lists/*
USER jenkins
```

Note: on Debian 13 the CLI is a **separate package** named `docker-cli` (not `docker.io`).

```bash
docker build -t jenkins-with-docker:local /home/mandog13/jenkins-docker

docker volume create jenkins_home
docker run -d --name jenkins --restart unless-stopped \
  -p 9090:8080 \
  --add-host=host.docker.internal:host-gateway \
  --group-add "$(getent group docker | cut -d: -f3)" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v jenkins_home:/var/jenkins_home \
  jenkins-with-docker:local
```

Jenkins version: **2.541.3 LTS**. The "new version 2.580.2 available" banner refers to a weekly build — ignore it.

### Password note

The initial admin password (`docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword`) is deleted once setup completes. If locked out, disable security via the config file (stop container first — Jenkins rewrites `config.xml` on shutdown), set a password via Script Console, then re-enable security. Final login: `admin` with the password you set.

### Plugins

Installed: Docker Pipeline, SonarQube Scanner (plugin ID `sonar`). The "SonarQube servers" global config only appears after that plugin is installed — and it is optional because the pipeline passes URL and token explicitly.

### Credentials (Global credentials, exact IDs)

| ID | Kind | Value |
|---|---|---|
| `docker-cred` | Username with password | Username **man1380** + Docker Hub access token |
| `github` | Secret text | GitHub PAT with repo write access |
| `sonarqube` | Secret text | Token from SonarQube 26.11 |

### Custom Maven agent

The original agent image has Docker CLI 20.10 (API 1.41), rejected by Docker 29 (needs ≥1.44). Built a replacement:

`/home/mandog13/jenkins-agent/Dockerfile`
```dockerfile
FROM abhishekf5/maven-abhishek-docker-agent:v1
USER root
COPY docker/docker /usr/bin/docker
RUN chmod +x /usr/bin/docker
```

where `docker/docker` is the static Docker **28.5.1** binary from
`https://download.docker.com/linux/static/stable/x86_64/docker-28.5.1.tgz`.

```bash
docker build -t cicd-maven-agent:local /home/mandog13/jenkins-agent
```

Maven 3.8.1 and Java 11.0.11 are unchanged. Because the image exists locally, Jenkins does not need to pull it.

### Pipeline job

New Item → Pipeline → `java-cicd`:
- Pipeline script from SCM → Git
- Repo: `https://github.com/man1328/Jenkins-Zero-To-Hero.git`
- Branch: `*/main`
- Script path: `java-maven-sonar-argocd-helm-k8s/spring-boot-app/JenkinsFile`

---

## 7. Phase 5 — Build and verify

Click **Build Now**. Stages: Checkout, Build and Test, Static Code Analysis, Build and Push Docker Image, Update Deployment File.

Argo CD auto-syncs the manifest change within ~1–3 minutes. Final verification:

```bash
kubectl get application spring-boot-app -n argocd   # Synced / Healthy
kubectl get pods -n default                         # 2/2 Running
kubectl get deployment spring-boot-app -n default \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
# man1380/ultimate-cicd:6
```

Port-forwards (must stay running in open terminals):

```bash
kubectl port-forward svc/argocd-server -n argocd 8082:443
kubectl port-forward svc/spring-boot-app-service -n default 8081:80
```

Both returned HTTP 200 when checked.

---

## 8. Complete error log (symptom → root cause → fix)

| # | Symptom | Root cause | Fix |
|---|---|---|---|
| 1 | `metadata.annotations: Too long: 262144 bytes`; applicationset-controller CrashLoopBackOff | Client-side `kubectl apply` puts the whole 1.57 MB ApplicationSet CRD into the last-applied annotation | Install with `kubectl apply --server-side --force-conflicts`, or add just the CRD with server-side apply |
| 2 | "SonarQube version no longer active" | `lts-community` tag → 9.9.8, retired | Run `sonarqube:community` (26.11) |
| 3 | "SonarQube servers" missing in Manage Jenkins → System | SonarQube Scanner plugin not installed | Install plugin `sonar`; optional anyway |
| 4 | Jenkins login password from EC2 didn't work | Separate local Jenkins instance; setup wizard had deleted initial password | Reset via Script Console; re-enable security |
| 5 | `docker: not found` in pipeline | Jenkins image lacks Docker CLI | Custom image with `docker-cli` package |
| 6 | `No plugin found for prefix 'sonar'` | Plugin group not in Maven defaults, not in pom | Fully qualified goal `org.sonarsource.scanner.maven:sonar-maven-plugin:5.9.0.7382:sonar` |
| 7 | `Failed to query server version ... HTTP connect timed out` | Sonar container had crashed; also container→host-gateway traffic blocked on this Docker host | Restart Sonar with `--restart unless-stopped`; use `--network=host` in agent args and `SONAR_URL=http://localhost:9000` |
| 8 | `client version 1.41 is too old. Minimum supported API version is 1.44` | Agent image has Docker CLI 20.10 | Custom agent image with static Docker 28.5.1 CLI |
| 9 | Docker login `unauthorized: incorrect username or password` | `docker-cred` used GitHub name `man1328` instead of Docker Hub name `man1380` | Update credential username to `man1380` |
| 10 | Argo CD `image: Required value`, pods ImagePullBackOff | `DOCKER_IMAGE` undefined in Update stage → `sed` wrote `image: ` (empty) | Define `DOCKER_IMAGE` in that stage; restore `deployment.yml` manually |

---

## 9. Day-to-day operations

- **Re-run the pipeline:** Jenkins → `java-cicd` → Build Now. Each build increments the tag and Argo CD deploys it.
- **Restart UIs:** re-run the two `kubectl port-forward` commands from Section 7.
- **Restart SonarQube:** `docker start sonar` (auto-restarts if the daemon restarts).
- **Recreate Jenkins container:** use the same `docker run` command from Section 6 with `jenkins-with-docker:local`; the `jenkins_home` volume keeps all settings.
- **Check the whole chain:** Sonar dashboard `http://localhost:9000`, Docker Hub `man1380/ultimate-cicd`, GitHub commit history, Argo CD UI, app at `http://localhost:8081`.

---

## 10. Local files created

- `/home/mandog13/jenkins-docker/Dockerfile` — Jenkins + Docker CLI
- `/home/mandog13/jenkins-agent/Dockerfile` and `docker/docker` — Maven agent + Docker 28.5.1
- `/home/mandog13/Jenkins-Zero-To-Hero-walkthrough.md` — this document

---

## 11. Final working Jenkinsfile

```groovy
pipeline {
  agent {
    docker {
      image 'cicd-maven-agent:local'
      args '--user root -v /var/run/docker.sock:/var/run/docker.sock --network=host'
    }
  }
  stages {
    stage('Checkout') {
      steps {
        sh 'echo passed'
      }
    }
    stage('Build and Test') {
      steps {
        sh 'cd java-maven-sonar-argocd-helm-k8s/spring-boot-app && mvn clean package'
      }
    }
    stage('Static Code Analysis') {
      environment {
        SONAR_URL = "http://localhost:9000"
      }
      steps {
        withCredentials([string(credentialsId: 'sonarqube', variable: 'SONAR_AUTH_TOKEN')]) {
          sh 'cd java-maven-sonar-argocd-helm-k8s/spring-boot-app && mvn org.sonarsource.scanner.maven:sonar-maven-plugin:5.9.0.7382:sonar -Dsonar.token=$SONAR_AUTH_TOKEN -Dsonar.host.url=${SONAR_URL}'
        }
      }
    }
    stage('Build and Push Docker Image') {
      environment {
        DOCKER_IMAGE = "man1380/ultimate-cicd:${BUILD_NUMBER}"
        REGISTRY_CREDENTIALS = credentials('docker-cred')
      }
      steps {
        script {
          sh 'cd java-maven-sonar-argocd-helm-k8s/spring-boot-app && docker build -t ${DOCKER_IMAGE} .'
          def dockerImage = docker.image("${DOCKER_IMAGE}")
          docker.withRegistry('https://index.docker.io/v1/', "docker-cred") {
            dockerImage.push()
          }
        }
      }
    }
    stage('Update Deployment File') {
      environment {
        GIT_REPO_NAME = "Jenkins-Zero-To-Hero"
        GIT_USER_NAME = "man1328"
        DOCKER_IMAGE = "man1380/ultimate-cicd:${BUILD_NUMBER}"
      }
      steps {
        withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
          sh '''
            git config user.email "maguilar1310@gmail.com"
            git config user.name "Manuel Aguilar"
            sed -i "s|image: .*/ultimate-cicd:.*|image: ${DOCKER_IMAGE}|g" java-maven-sonar-argocd-helm-k8s/spring-boot-app-manifests/deployment.yml
            git add java-maven-sonar-argocd-helm-k8s/spring-boot-app-manifests/deployment.yml
            git commit -m "Update deployment image to version ${BUILD_NUMBER}"
            git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
          '''
        }
      }
    }
  }
}
```

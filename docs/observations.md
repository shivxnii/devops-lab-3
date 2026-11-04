# Lab 3 Observations

## Q1 - Ansible configuration management
- Inventory used two connection types: `local` for the control machine and the `docker` connection plugin for the managed node (no SSH setup needed).
- The playbook installed nginx, deployed the index page from a template and started it with `service` (the container has no systemd).
- Second run showed `changed=0` - idempotency means Ansible only changes what actually differs from the declared state.
- Gotcha: port 8080 was already taken by Jenkins, so the node's nginx is published on 8081.

## Q2 - Terraform (IaC)
- `terraform init` downloaded the kreuzwerker/docker provider; `plan` showed exactly what would be created before anything was touched.
- Scaling was a one-line change: `terraform apply -var="replicas=4"` added lab3-web-3 and lab3-web-4 with the next free host ports (8084, 8085) automatically.
- `terraform state list` tracks every object Terraform manages - the source of truth for the infrastructure.

## Q3 - Prometheus + Grafana
- The app exposes a Prometheus counter (`lab3_app_requests_total`) via /metrics; the endpoint itself is excluded from counting.
- Prometheus scrapes every 5s (`global.scrape_interval`); both targets (app and prometheus itself) show UP on the targets page.
- Grafana datasource and dashboard are provisioned from files - no manual clicks, the dashboard exists on first boot.
- The whole monitoring stack is one `docker compose up -d` (app + prometheus + grafana on a shared network).

## Q4 - ELK centralized logging
- Log flow: app writes /var/log/lab3/app.log (shared volume) -> Logstash `file` input -> grok parse (`TIMESTAMP_ISO8601 LOGLEVEL GREEDYDATA`) -> Elasticsearch index `lab3-app-logs` -> Kibana.
- Grok parsing turned raw lines into structured fields (`level`, `log_message`, `@timestamp`), which makes filtering in Kibana possible.
- Elasticsearch in Docker requires `vm.max_map_count=262144` on the host (WSL2 default is 65530).
- Cluster health is yellow on a single node because replicas cannot be placed on the same node - expected.
- Kibana shows a "your data is not secure" banner because xpack.security is intentionally disabled for the lab.

## Q5 - DevSecOps pipeline
- The pipeline gates on **pip-audit**: build 1 was blocked because flask 3.1.0 / pytest 8.3.4 had known vulnerabilities; after bumping (3.1.3 / 9.0.3) the audit passed.
- **Bandit** static analysis flagged two medium findings: hardcoded /tmp path (fixed in code) and binding 0.0.0.0 (accepted with justification - required for containers - via `--skip B104` documented in the Jenkinsfile).
- **Trivy** scanned the deployed lab2 image and reported real base-image CVEs (util-linux, perl-base, ...) - recorded as report-only, not a gate.
- The honest sequence: FAIL (vulnerable deps) -> FAIL (bandit) -> fix commits -> SUCCESS. The gate does what it is supposed to do.
- Secure pipeline practices: secrets live in Jenkins credentials (never in the repo), tokens passed via --password-stdin, app runs as non-root user, scanners fail the build instead of just warning.

## Q6 - Feature flags, load testing, browser testing
- Feature flag: the same image behaves differently via the FEATURE_BETA env var - /beta returned 404 with the flag off and 200 with it on. No rebuild or deploy needed to toggle a feature.
- Load test (ab, 2000 requests, concurrency 50): 323.74 req/s, 0 failed requests, mean 154 ms, p95 211 ms, p99 267 ms - the bottleneck is the Flask dev server, not the pattern.
- Selenium + headless Chrome 157 / ChromeDriver 157 (matching versions are essential): asserted /health content, asserted the enabled beta flag and saved a screenshot - all from a script that can run in CI.

## Environment notes (WSL2)
- Docker containers die when the last WSL session closes (same as Lab 2) - keep a session open (e.g. `sleep infinity`) or restart with `docker start` / `compose up -d`.
- Terraform apt repo key mismatch was bypassed by installing the binary from releases.hashicorp.com directly.
- The Ansible apt PPA served ansible core 2.21.5 for Ubuntu 24.04.

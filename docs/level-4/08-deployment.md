---
description: "Deploy Spring Boot to production: build-once pipelines, per-environment config, Kubernetes probes and resources, rolling updates, and safe Flyway runs."
---

# Deploying to Production

Deployment is where earlier choices pay off: externalized configuration (Level 1), Flyway
migrations (Level 2), health probes (Level 3), images and graceful shutdown (this level).
This lesson assembles them into a pipeline and a Kubernetes deployment, and covers the
failure modes that bite during a rollout.

## Build once, promote everywhere

```text
commit ─► CI: test ─► build image (tag = git SHA) ─► scan ─► push
          ─► deploy to staging ─► smoke tests ─► deploy SAME image to production
```

The image built from a commit is the artifact. Environments differ only in configuration
supplied at deploy time (environment variables, mounted config, secrets). Never rebuild
per environment — then what you tested is not what you ship.

## Kubernetes essentials

The manifests below were not applied to a cluster for this course; they show the fields
that matter for a Spring Boot service.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: library
spec:
  replicas: 3
  strategy:
    rollingUpdate: { maxUnavailable: 0, maxSurge: 1 }
  selector: { matchLabels: { app: library } }
  template:
    metadata:
      labels: { app: library }
    spec:
      terminationGracePeriodSeconds: 45
      containers:
        - name: app
          image: registry.example.com/library:3f9c2ab
          ports: [{ containerPort: 8080 }, { containerPort: 8081, name: management }]
          env:
            - { name: SPRING_PROFILES_ACTIVE, value: prod }
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom: { secretKeyRef: { name: library-db, key: password } }
          resources:
            requests: { cpu: "500m", memory: "768Mi" }
            limits:   { memory: "768Mi" }
          startupProbe:
            httpGet: { path: /actuator/health/liveness, port: management }
            periodSeconds: 5
            failureThreshold: 30
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: management }
            periodSeconds: 10
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: management }
            periodSeconds: 5
          lifecycle:
            preStop:
              sleep: { seconds: 5 }
```

Points worth understanding:

- **startupProbe** gives slow JVM startup room without making the liveness probe lenient
  forever. Liveness checks start only after it succeeds.
- **readinessProbe** keeps new pods out of the Service until Boot reports
  `ACCEPTING_TRAFFIC`, and removes them when shutdown begins.
- **preStop sleep** gives load balancers time to stop sending traffic before the app starts
  refusing it; endpoint removal and `SIGTERM` happen concurrently, so without it a few
  requests can hit a closing pod. (The `sleep` lifecycle handler is a recent Kubernetes
  feature; older clusters use an `exec` of `sleep`.)
- **terminationGracePeriodSeconds** must exceed preStop plus Boot's
  `timeout-per-shutdown-phase`.
- **Memory limit equals request** to avoid being killed under node pressure; heap is a
  percentage of it (lesson 05).
- `maxUnavailable: 0` means capacity never dips during a rollout.

Boot detects Kubernetes and enables the liveness/readiness probe endpoints automatically;
setting `management.endpoint.health.probes.enabled=true` makes it explicit.

## Database migrations in the rollout

Options for running Flyway:

1. **At application startup** (the default). Simple. Flyway's lock ensures one instance
   migrates while others wait. Downsides: startup time includes migrations, and the app user
   needs DDL rights.
2. **As a separate step** before the rollout — a Kubernetes Job or pipeline stage running
   the Flyway CLI or the app with `spring.main.web-application-type=none` and a migration-only
   profile, using a privileged migration user. Set `spring.flyway.enabled=false` for the
   app itself.

Either way, every migration must be **backward compatible** with the version still running
(Level 2's expand–contract), because old pods serve traffic until the rollout finishes.

## Worked example: a safe rollout checklist

Before merging:

- [ ] Migration is additive or part of an expand–contract sequence.
- [ ] New config properties have defaults or are set in every environment.
- [ ] API changes are backward compatible for existing clients.

During rollout:

- [ ] Watch error rate and p99 latency per version (tag metrics with the version from
      `build-info`: add the `build-info` goal to the Boot Maven plugin, and it appears in
      `/actuator/info`).
- [ ] Watch pod restarts and readiness failures.

If metrics degrade: roll back the Deployment (`kubectl rollout undo`) — safe precisely
because the migration was backward compatible.

## Beyond rolling updates

- **Blue/green**: run the new version alongside the old, switch traffic at once, keep the
  old ready for instant rollback.
- **Canary**: send a small percentage of traffic to the new version and promote
  automatically if metrics hold (Argo Rollouts, Flagger, service meshes).
- **Feature flags**: deploy code dark and enable it separately from deployment.

## How It Actually Works

During a rolling update, the Deployment controller creates a new ReplicaSet and scales it
up while scaling the old one down, within `maxSurge`/`maxUnavailable`. A new pod counts as
available only when its readiness probe passes — which in Boot means the context refreshed,
`ApplicationReadyEvent` fired, and the readiness state became `ACCEPTING_TRAFFIC`. For an
old pod, deletion triggers two things in parallel: the endpoints controller removes it from
Service endpoints (propagating to kube-proxy and load balancers over a few seconds), and the
kubelet runs `preStop` then sends `SIGTERM`. Boot's graceful shutdown then sets readiness to
`REFUSING_TRAFFIC`, stops accepting connections, drains in-flight requests, and closes the
context. If the grace period expires first, `SIGKILL` ends it abruptly.

## Common mistakes

- **Rebuilding images per environment.**
- **Liveness probes on the readiness path** (or including the database), causing restart
  loops during dependency outages.
- **No startup probe**, so a slow start is killed by liveness.
- **Grace period shorter than the shutdown timeout.**
- **Destructive migrations in the same release as the code that stops using a column.**

## Exercise

1. Write the Deployment, Service, and Secret for the library API, and deploy it to a local
   kind or minikube cluster.
2. While a load generator hits the service, perform a rolling update to a new image tag.
   Count errors with and without the `preStop` sleep.
3. Move Flyway to a Kubernetes Job with a migration user, and disable it in the app.
4. Add `build-info` to the Maven plugin and show the version in `/actuator/info`.

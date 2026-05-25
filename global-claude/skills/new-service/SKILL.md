---
name: new-service
description: scaffolds a new microservice
disable-model-invocation: true

---
Scaffold a new microservice named $ARGUMENTS following all 
rules in CLAUDE.md. Include:
- Docker multi-stage Dockerfile
- k8s/base manifests (deployment, service, configmap, secret, hpa packaged to use with helm charts, kustomize)
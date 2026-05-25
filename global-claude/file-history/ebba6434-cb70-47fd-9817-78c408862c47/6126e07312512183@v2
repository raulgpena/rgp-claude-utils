---
name: Never apply terraform locally with placeholder values
description: Running terraform apply with "x"/placeholder values for credential vars will overwrite real secrets
type: feedback
---

Never run `terraform apply` locally with placeholder/fake values for sensitive variables (postgres_admin_password, keycloak_admin_password, ACR credentials, etc.).

**Why:** Terraform will overwrite Key Vault secrets, container app environment variables, and other config with the placeholder values — breaking all running services.

**How to apply:** For local inspection, `terraform plan` with placeholders is safe (read-only). For actual applies, push the code and let the CI/CD pipeline run — it has all real values injected as GitHub Actions secrets/vars.

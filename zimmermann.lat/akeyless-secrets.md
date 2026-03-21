# Akeyless Secrets for zimmermann.lat

All secrets are stored under the Akeyless base path `/zimmermann.lat/`.

## Bootstrap (manual)

Before ExternalSecrets can sync, the Kubernetes secret `akeyless-secret-creds` must be created manually in the akeyless namespace:

| Key              | Description                          |
|------------------|--------------------------------------|
| `accessId`       | Akeyless access ID                   |
| `accessType`     | Authentication type (e.g. `api_key`) |
| `accessTypeParam`| API key or other auth parameter      |

```bash
kubectl create secret generic akeyless-secret-creds \
  --namespace akeyless \
  --from-literal=accessId='<your-access-id>' \
  --from-literal=accessType='<your-access-type>' \
  --from-literal=accessTypeParam='<your-access-type-param>'
```

## Traefik

### ACME / Let's Encrypt (Cloudflare DNS-01 challenge)

| Akeyless Path                                              | Description                     |
|------------------------------------------------------------|---------------------------------|
| `/zimmermann.lat/acme/cloudflare-api-credentials_email`    | Cloudflare account email        |
| `/zimmermann.lat/acme/cloudflare-api-credentials_apiKey`   | Cloudflare Global API key       |

### OAuth2-Proxy (Cognito OIDC authentication)

| Akeyless Path                                              | Description                              |
|------------------------------------------------------------|------------------------------------------|
| `/zimmermann.lat/oidc/oauth2-proxy/client_id`              | Cognito OIDC client ID                   |
| `/zimmermann.lat/oidc/oauth2-proxy/client_secret`          | Cognito OIDC client secret               |
| `/zimmermann.lat/oidc/oauth2-proxy/cookie_secret`          | Random 32-byte base64 string for cookies |

Generate the cookie secret:

```bash
openssl rand -base64 32
```

## ArgoCD

| Akeyless Path                                              | Description                         |
|------------------------------------------------------------|-------------------------------------|
| `/zimmermann.lat/oidc/oauth2-proxy/client_secret`          | Shared with traefik (see above)     |
| `/zimmermann.lat/argocd/webhook/github/secret`             | GitHub webhook secret for push events |

## Restic

| Akeyless Path                                              | Description                         |
|------------------------------------------------------------|-------------------------------------|
| `/restic/S3_ACCESS_KEY_ID`                                 | S3 access key ID                    |
| `/restic/S3_SECRET_ACCESS_KEY`                             | S3 secret access key                |
| `/restic/S3_BUCKET`                                        | S3 bucket name                      |
| `/restic/S3_ENDPOINT`                                      | S3 endpoint URL                     |
| `/restic/S3_REGION`                                        | S3 region                           |
| `/restic/RESTIC_PASSWORD`                                  | Restic repository password          |

## Apps with no secrets

The following enabled apps do not require Akeyless secrets:

- **akeyless** - only deploys SecretStore/ClusterSecretStore resources
- **homeassistant** - config loaded via ConfigMap from Git
- **homer** - authentication handled by traefik oauth2-proxy
- **reloader** - no secret dependencies

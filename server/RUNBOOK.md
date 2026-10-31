# Production runbook

## Deploy to Google Cloud Run

### Prerequisites and identity

- Create a Google Cloud project, Artifact Registry Docker repository, and
  Cloud Run service in the same region.
- Enable Cloud Run, Artifact Registry, Firestore, Cloud Logging, Cloud
  Monitoring, Error Reporting, and Cloud Scheduler APIs.
- Provision a dedicated runtime service account. Grant only the Firestore
  roles required by the API and Firebase Auth token verification. Avoid
  downloading a service-account key; Cloud Run uses Application Default
  Credentials from its attached identity.
- Create Secret Manager values for the exact frontend origin allow-list,
  Cloudinary cloud name/key/secret, and signed upload preset names. Never put
  secret values in the YAML, Docker image, or source control. Add Gemini secrets
  only if enabling AI.

### Build and deploy

From `server/`, substitute your project, region, repository, and image tag:

```bash
gcloud builds submit --tag REGION-docker.pkg.dev/PROJECT_ID/freelanso/api:TAG .
```

Review `deploy/cloudrun.service.yaml`, replacing only the non-secret
`PROJECT_ID`, `REGION`, and `TAG` placeholders. Create the referenced Secret
Manager entries and grant the Cloud Run runtime identity
`roles/secretmanager.secretAccessor` on those individual secrets. Deploy:

```bash
gcloud run services replace deploy/cloudrun.service.yaml --region REGION --project PROJECT_ID
```

The service template sets one minimum instance, concurrency 40, a 60-second
request timeout, 1 vCPU / 512 MiB, startup readiness on `/readyz`, and liveness
on `/healthz`. `TRUST_PROXY=1` is configured for the Cloud Run proxy. The
frontend is deployed separately; set its `VITE_API_URL` to the Cloud Run HTTPS
URL ending in `/api`. Set `CORS_ALLOWED_ORIGINS` to the exact HTTPS frontend
origin (scheme and host, no wildcard or path).

### Release verification and rollback

After deployment, call `/healthz` and `/readyz`, then exercise one public read
and one authenticated write using a test Firebase account. Confirm CORS from
the deployed frontend, logs contain request IDs, Error Reporting has no
unexpected groups, and 5xx/latency/auth-failure alerts are healthy.

Cloud Run keeps prior revisions. To roll back, route 100% traffic to the last
known-good revision; do not rebuild or overwrite that image:

```bash
gcloud run services update-traffic freelanso-api \
  --region REGION --to-revisions KNOWN_GOOD_REVISION=100 --project PROJECT_ID
```

Investigate migrations separately. Firestore schema changes must be
backward-compatible during a rollback.

## Monitoring and alerts

Pino writes structured JSON to stdout/stderr. Authorization headers, cookies,
tokens, email, phone, and sensitive request-body fields are redacted. Internal
5xx logs use generic messages and safe stack frames with `serviceContext` and
`stack_trace`, allowing Cloud Logging/Error Reporting grouping without logging
the raw exception message or user input.

Create log-based metrics and alert policies in Cloud Monitoring for:

- **5xx rate:** filter Cloud Run revision logs for
  `resource.labels.service_name="freelanso-api"` and
  `jsonPayload.res.statusCode>=500`; alert when 5xx exceed 2% of requests over
  5 minutes (after a minimum request volume).
- **Latency:** filter the same service and measure `jsonPayload.responseTime`;
  alert when p95 exceeds 1 second for 10 minutes.
- **Auth failures:** filter `jsonPayload.authFailure:*`; alert on a sustained
  5x increase against the established per-service baseline over 5 minutes.
- **Readiness:** alert when `/readyz` fails or the Cloud Run instance is
  unhealthy. Keep liveness and readiness separate so a Firestore outage does
  not look like a process crash.

Set notification channels before creating production alert policies. Tune
thresholds after observing normal traffic; do not page on single 401 responses.

## Firestore backups and restore

Create a dedicated, region-appropriate Cloud Storage bucket with uniform bucket
access, encryption, restricted backup-reader access, and a lifecycle policy
that matches the organization retention requirement. Grant the Cloud Run Job
identity `roles/datastore.importExportAdmin`; grant the Cloud Firestore service
agent access to the bucket as required by the bucket/project configuration.

Create a Cloud Run Job using the Google Cloud CLI image and a dedicated
service identity with Firestore export and bucket-write permissions. Its
command should execute a timestamped export (the timestamp avoids overwriting
an earlier export):

```bash
gcloud run jobs create freelanso-firestore-export \
  --image=gcr.io/google.com/cloudsdktool/google-cloud-cli:stable \
  --command=/bin/bash \
  --args=-ceu,'gcloud firestore export "gs://BACKUP_BUCKET/freelanso/$(date -u +%Y%m%dT%H%M%SZ)" --project=PROJECT_ID --database="(default)"' \
  --service-account=firestore-export@PROJECT_ID.iam.gserviceaccount.com \
  --region=REGION --project=PROJECT_ID
```

Grant the Cloud Scheduler service identity `roles/run.invoker` on that job.
Create a daily Scheduler HTTP job targeting the Cloud Run Jobs `:run` endpoint
with OAuth using that identity, method `POST`, and body `{}`:

```text
https://REGION-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/PROJECT_ID/jobs/freelanso-firestore-export:run
```

Use the URI above for `--uri` in `gcloud scheduler jobs create http`, set
`--schedule="0 2 * * *"`, `--time-zone=UTC`,
`--oauth-service-account=SCHEDULER_SERVICE_ACCOUNT`, `--http-method=POST`,
`--message-body='{}'`, and `--location=REGION`.

Verify the first job completes and produces export metadata in the bucket.
Alert on Scheduler/Job failures and periodically test a restore in an isolated
non-production Firebase project.

Restore to a new/isolated project first:

```bash
gcloud firestore import gs://BACKUP_BUCKET/freelanso/EXPORT_FOLDER \
  --project=RESTORE_PROJECT_ID
```

Verify document counts, indexes, application access, and dependent records
before directing traffic to restored data. Restore Firestore indexes and
Firebase RTDB from their separately managed backup procedures; a Firestore
export does not include Realtime Database messaging data.

## Credential rotation and exposure response

1. Treat any suspected committed credential as compromised immediately.
2. Revoke/rotate it at its provider first (Firebase service account, Cloudinary,
   Gemini, or other service); update Secret Manager and redeploy.
3. Check provider audit logs and Cloud Logging for unauthorized use.
4. Remove the credential from Git history only after rotation, coordinate a
   history rewrite with repository owners, and invalidate affected clones.
5. Run Gitleaks against the complete repository history and current files.
   Never paste detected values into issues, logs, or chat.
6. Record the incident, exposed systems, rotation time, and verification.

`npm audit` should be rerun after lockfile changes. Production dependency
findings and development-tool advisories are reported separately; do not use
`npm audit fix --force` without reviewing major-version and Node compatibility
changes.

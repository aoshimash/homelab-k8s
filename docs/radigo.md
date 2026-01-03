# Radigo Scheduled Recorder

Automated radiko program recording system using Kubernetes CronJobs, saving recordings to Audiobookshelf's podcasts PVC and triggering library scans via API.

## Overview

The radigo scheduler records radiko radio programs on a schedule using the timefree feature, validates program titles to skip hiatus/special broadcasts, and automatically updates the Audiobookshelf library.

## Architecture

- **CronJobs**: One CronJob per program, triggered at broadcast end time
- **Recording Script**: Shell script in ConfigMap that orchestrates radigo recording
- **Storage**: Shared PVC (`audiobookshelf-podcasts`) with audiobookshelf pod
- **Pod Affinity**: Ensures radigo pods run on same node as audiobookshelf (RWO PVC requirement)

## Components

### Namespace
- `radigo`: Isolated namespace for radigo components

### ConfigMap
- `radigo-record-script`: Shell script for recording orchestration

### Secret
- `radigo-audiobookshelf-api`: SOPS-encrypted Audiobookshelf API credentials

### CronJobs
- `radigo-arco`: Records オードリーのオールナイトニッポン (Saturday 25:00-27:00)
- `radigo-ijuin`: Records 伊集院光 深夜の馬鹿力 (Monday 25:00-27:00)
- Additional CronJobs can be added for more programs

## Setup

### Prerequisites

1. Audiobookshelf deployed with `audiobookshelf-podcasts` PVC
2. SOPS age key configured in cluster
3. Network access to radiko.jp

### Initial Configuration

1. **Get Audiobookshelf API Key**:
   - Access Audiobookshelf web UI
   - Settings → Users → Select user → API Keys
   - Create new API key
   - Copy the API key value

2. **Get Library ID**:
   - Go to Libraries → Select podcasts library
   - Note the Library ID from URL or library details
   - Example: If URL is `/libraries/abc123-def456-ghi789`, the Library ID is `abc123-def456-ghi789`

3. **Create and Encrypt Secret**:
   ```bash
   # Edit secret file
   vim k8s/apps/radigo/secret-audiobookshelf-api.sops.yaml
   
   # Replace placeholders:
   # - REPLACE_WITH_ACTUAL_API_KEY → Your API key
   # - REPLACE_WITH_ACTUAL_LIBRARY_ID → Your Library ID
   
   # Encrypt with SOPS
   sops -e -i k8s/apps/radigo/secret-audiobookshelf-api.sops.yaml
   
   # Verify encryption
   cat k8s/apps/radigo/secret-audiobookshelf-api.sops.yaml | grep -q "ENC\[" && echo "Encrypted successfully"
   ```

4. **Deploy**:
   ```bash
   git add k8s/apps/radigo/
   git commit -m "feat: add radigo scheduled recorder"
   git push
   
   # Wait for Flux reconciliation
   flux reconcile kustomization apps --with-source
   ```

5. **Verify Deployment**:
   ```bash
   # Check namespace
   kubectl get ns radigo
   
   # Check CronJob created
   kubectl get cronjobs -n radigo
   
   # Check ConfigMap
   kubectl get configmap -n radigo
   
   # Check Secret (should show encrypted)
   kubectl get secret -n radigo radigo-audiobookshelf-api
   ```

6. **Manual Test Recording**:
   ```bash
   # Create manual test job
   kubectl create job --from=cronjob/radigo-arco radigo-arco-test -n radigo
   
   # Watch job progress
   kubectl get jobs -n radigo -w
   
   # Check logs
   kubectl logs -n radigo -l job-name=radigo-arco-test -f
   
   # Verify recording file created
   kubectl exec -n audiobookshelf deploy/audiobookshelf -- ls -la /podcasts/arco/
   
   # Verify episode appears in Audiobookshelf library UI
   # (Access Audiobookshelf web UI and check podcasts library)
   
   # Verify RSS feed includes new episode
   # (Get RSS feed URL from Audiobookshelf and check feed content)
   ```

## Recording Workflow

1. **CronJob Trigger**: CronJob triggers at program end time (e.g., 03:05 Sunday for Saturday broadcast)
2. **Schedule Fetch**: Script fetches radiko program schedule API
3. **Title Validation**: Validates program title matches expected pattern
4. **Recording**: Executes radigo timefree recording if title matches
5. **File Organization**: Saves to program-specific directory (`/podcasts/{program}/`)
6. **Library Scan**: Triggers Audiobookshelf library scan API
7. **Logging**: Logs all steps for debugging

## Adding New Programs

1. Create new CronJob file `k8s/apps/radigo/cronjob-{program-name}.yaml`:
   ```yaml
   apiVersion: batch/v1
   kind: CronJob
   metadata:
     name: radigo-{program-name}
     namespace: radigo
   spec:
     schedule: "5 3 * * 0"  # Adjust based on broadcast end time
     jobTemplate:
       spec:
         template:
           spec:
             containers:
             - name: radigo
               image: yyoshiki41/radigo
               env:
               - name: STATION_ID
                 value: "LFR"
               - name: DAY_OF_WEEK
                 value: "6"
               - name: START_TIME
                 value: "01:00"
               - name: EXPECTED_TITLE
                 value: "番組名"
               - name: OUTPUT_DIR
                 value: "/podcasts/{program-name}"
               # ... (see existing CronJob for full template)
   ```

2. Update `k8s/apps/radigo/kustomization.yaml` to include new CronJob

3. Commit and push

## Troubleshooting

### Recording Not Appearing

1. Check job status:
   ```bash
   kubectl get jobs -n radigo
   kubectl logs -n radigo job/<job-name>
   ```

2. Verify file exists:
   ```bash
   kubectl exec -n audiobookshelf deploy/audiobookshelf -- ls -la /podcasts/{program}/
   ```

3. Trigger manual scan:
   ```bash
   curl -X POST "http://audiobookshelf.audiobookshelf.svc.cluster.local/api/libraries/<library_id>/scan" \
        -H "Authorization: Bearer <api_key>"
   ```

### Title Mismatch (Recording Skipped)

This is expected behavior when:
- Program is on hiatus
- Special broadcast replaces regular program
- Program schedule changed

**Title Validation**:
The recording script validates the program title before recording. If the actual title doesn't contain the expected pattern, recording is skipped and logged.

Check logs:
```bash
kubectl logs -n radigo job/<job-name> | grep -i title
```

**Testing Title Validation**:
To test title validation, temporarily modify the `EXPECTED_TITLE` environment variable in a test job:
```bash
# Create test job with modified expected title
kubectl create job radigo-arco-title-test -n radigo \
  --from=cronjob/radigo-arco \
  --dry-run=client -o yaml | \
  sed 's/オードリーのオールナイトニッポン/テスト用タイトル/' | \
  kubectl apply -f -

# Check logs - should show title mismatch and skip
kubectl logs -n radigo job/radigo-arco-title-test | grep -i "title\|skip"
```

**Restore Configuration**:
After testing, ensure the CronJob has the correct `EXPECTED_TITLE` value.

### Pod Stuck in Pending

Usually PVC mounting issue - check pod affinity:
```bash
kubectl describe pod -n radigo <pod-name>
```

Ensure audiobookshelf pod is running (required for RWO PVC sharing).

### Manual Retry

**Automatic Retry**:
CronJobs are configured with `backoffLimit: 2`, which means Kubernetes will automatically retry failed jobs up to 2 times.

**Manual Retry**:
If automatic retries are exhausted, create manual job:
```bash
kubectl create job --from=cronjob/radigo-arco radigo-arco-retry-$(date +%s) -n radigo
```

**Retry Procedure**:
1. Check why the job failed:
   ```bash
   kubectl logs -n radigo job/<failed-job-name>
   kubectl describe job -n radigo <failed-job-name>
   ```

2. If it's a transient error (network, API timeout), retry is safe
3. If it's a configuration error, fix the issue first
4. Timefree recordings are available for 1 week, so manual retry is possible within that window

## Monitoring

### View Recent Recordings

```bash
# List jobs by completion time
kubectl get jobs -n radigo --sort-by=.status.completionTime

# View logs
kubectl logs -n radigo job/<job-name>
```

### Check CronJob Schedules

```bash
kubectl get cronjobs -n radigo -o wide
```

## File Structure

Recordings are saved to:
```
/podcasts/
├── arco/
│   ├── 20260104-LFR.aac
│   ├── 20260111-LFR.aac
│   └── ...
└── ijuin/
    ├── 20260107-TBS.aac
    └── ...
```

Each directory appears as a separate podcast series in Audiobookshelf.

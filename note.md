# Notes for Manual Follow-up

## Slack evidence

I could not capture `slack-firing.png` and `slack-resolved.png` because that requires authenticated access to the target Slack workspace UI, which is a third-party service outside this local workspace.

What is already done:

- Alertmanager starts successfully.
- `ServiceDown` fires when `day23-app` is stopped.
- `ServiceDown` resolves after `day23-app` is restored.
- Local evidence is saved as:
  - `submission/screenshots/alertmanager-firing.png`
  - `submission/screenshots/alertmanager-resolved.png`

What you need to do manually for the Slack rubric item:

1. Put a real Slack incoming webhook into the two `api_url` fields in `02-prometheus-grafana/alertmanager/alertmanager.yml`, replacing `https://hooks.slack.com/services/REPLACE/ME/HERE`.
2. Run `docker compose up -d alertmanager`.
3. Stop the app until `ServiceDown` fires:
   ```powershell
   docker compose stop app
   ```
4. Confirm the firing message in Slack and save it as `submission/screenshots/slack-firing.png`.
5. Restore the app:
   ```powershell
   docker compose up -d app
   ```
6. Confirm the resolved message in Slack and save it as `submission/screenshots/slack-resolved.png`.

## Repository URL

The public GitHub URL was not available in the local workspace, so `submission/REFLECTION.md` contains `not provided in the local workspace`.

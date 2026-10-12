# SMS Territorial Operations Hub

The SMS Territorial Operations Hub is a demonstration prototype for local governments and municipal departments. It illustrates how field agents can report public works and departmental activities, raise incidents, and request an operational status or brief through SMS-style messages.

## What the template demonstrates

- **UPDATE** — records an agent's report and prepares a simulated confirmation.
- **ALERT** — records an incident and checks active ALERT routing rules. A department match does not identify a specific phone number, so a department notification can remain unresolved.
- **STATUS** — prepares a view of the identified agent's own recorded updates and incidents.
- **BRIEF** — prepares a summary of the identified agent's recent recorded activity.

These describe the workflow's configured paths. The public combined export has not been imported into or tested in a clean n8n instance, so its compatibility and behavior in another instance remain unverified.

## Files

- [Installation guide](SETUP.md) — Data Table schemas, setup, configuration, and manual test guidance.
- [Combined workflow export](workflows/sms-territorial-operations-hub-combined.public-final.json) — Main and Send SMS sections on one n8n canvas.

Import the combined JSON once. Then follow `SETUP.md` to copy the Send SMS section into its own workflow and reconnect the four calls from Main.

## Demonstration and safety limits

- This is a **demonstration prototype**, not a production system.
- Outbound SMS are simulated and recorded for demonstration. No SMS provider is configured, and no real SMS is sent or delivered.
- ALERT routing can identify a department, but it does not establish an individually addressable recipient or send a department notification.
- The workflow does not provide production-grade gateway authentication, authorization, abuse protection, or privacy safeguards. Do not use it with real public traffic or personal data.
- Keep both workflows inactive during initial setup. Secure and review your own configuration before considering any production use.

See [SETUP.md](SETUP.md) for the full installation steps and known limitations.

# SMS Territorial Operations Hub

![SMS Territorial Operations Hub cover](assets/sms-territorial-operations-hub-cover.png)

A demonstration prototype for local governments and municipal departments. It illustrates how field agents can report public works and departmental activities, raise incidents, and request an operational status or brief through SMS-style messages. No live SMS service is configured.

[View the official n8n template](https://n8n.io/workflows/20827-track-municipal-sms-field-reports-with-openai-and-data-tables/)

## The operational problem

In many municipalities, field reports are still shared through phone calls, informal messages, or paper records. This makes it difficult to maintain a clear picture of ongoing activities, reported incidents, and the follow-up required.

Inspired by the operational challenges faced by local administrations in rural Africa, this project explores how SMS-style reporting could help organize field information and make it easier for municipal teams to access and review.

The approach can also be adapted to municipalities and community organizations in other regions.

## What the workflow demonstrates

- **UPDATE** — records an agent's field activity report and prepares a simulated confirmation.
- **ALERT** — records an incident and checks active ALERT routing rules. A department match does not identify a specific phone number or send a notification.
- **STATUS** — prepares a view of the identified agent's own recorded updates and incidents.
- **BRIEF** — prepares a summary of the identified agent's recent records.

These are the configured paths represented in the public export. It has not been imported into or tested in a clean n8n instance, so compatibility and behavior in another instance remain unverified.

## Project files

- [Setup guide](SETUP.md) — Data Table schemas, setup, configuration, and manual test guidance.
- [Public workflow JSON](workflows/sms-territorial-operations-hub-combined.public-final.json) — Main and Send SMS sections on one n8n canvas.

Import the combined JSON once. Then follow `SETUP.md` to copy the Send SMS section into its own workflow and reconnect the four calls from Main.

## Prototype limits

- This is a demonstration prototype, not a production system.
- Outbound SMS are simulated and recorded for demonstration. No SMS provider is configured, and no real SMS is sent or delivered.
- ALERT routing can identify a department, but it does not establish an individually addressable recipient or send a department notification.
- `Verify Gateway` and `Check Interpretation and Scope` are placeholder checks. They do not authenticate the gateway or enforce an approved reporting scope.
- The workflow does not provide production-grade gateway authentication, authorization, abuse protection, or privacy safeguards. Do not use it with real public traffic or personal data.
- Keep both workflows inactive during initial setup. Secure and review your own configuration before considering production use.

See [SETUP.md](SETUP.md) for the full installation steps and known limitations.

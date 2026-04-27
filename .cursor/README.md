# Cursor Background Agent setup

Deze map configureert deze repo voor [Cursor Background Agents](https://docs.cursor.com/background-agent) (a.k.a. Cursor Automation / Cloud Agents).

- [`environment.json`](./environment.json) — install-script dat draait bij snapshot-bouw. Installeert `uv`/`uvx` en decodeert optionele OAuth-secrets naar `/tmp/google-auth/`.
- [`mcp.json`](./mcp.json) — per-project MCP-config; alle commando's gebruiken `PATH`-lookup (geen macOS Homebrew-paden), alle secrets zijn `${VAR}`-placeholders.

## Security waarschuwing

De Atlassian-tokens en GitHub PAT die voorheen in de globale `~/.cursor/mcp.json` stonden zijn in chat-history gelekt. **Roteer ze nu**:

- Atlassian: https://id.atlassian.com/manage-profile/security/api-tokens
- GitHub PAT: https://github.com/settings/tokens

## Vereiste Background Agent secrets

Zet deze in Cursor → Settings → Background Agents → Secrets (of in de Cloud Agent UI):

| Secret | Waarvoor | Verplicht |
| --- | --- | --- |
| `JIRA_API_TOKEN` | mcp-atlassian | ja, voor Jira |
| `CONFLUENCE_API_TOKEN` | mcp-atlassian | ja, voor Confluence |
| `GITHUB_TOKEN` | github MCP | ja |
| `BIGQUERY_PROJECT` | bigquery | ja, anders skippen |
| `FIRESTORE_PROJECT` | firestore | ja, anders skippen |
| `FIRESTORE_DATABASE` | firestore | ja, anders skippen |
| `SLACK_CLIENT_ID` | slack MCP | ja, anders skippen |
| `GOOGLE_OAUTH_KEYS_B64` | google-drive / gmail / google-chat | optioneel |
| `GOOGLE_OAUTH_TOKENS_B64` | google-drive / google-chat | optioneel |
| `GMAIL_CREDENTIALS_B64` | gmail | optioneel |
| `GOOGLE_CHAT_MCP_REPO` | google-chat | optioneel (git URL) |

## MCP status-matrix

| MCP | Status in Background Agent | Toelichting |
| --- | --- | --- |
| `adk-docs-mcp` | werkt | `uvx` wordt geïnstalleerd door `environment.json` |
| `mcp-atlassian` | werkt | secrets nodig |
| `github` | werkt | bearer token via `${GITHUB_TOKEN}` |
| `bigquery` | werkt mits ADC | `npx` is preinstalled; vereist Google Application Default Credentials (zie hieronder) |
| `firestore` | werkt mits ADC | idem |
| `notion` | werkt | URL-based via `mcp-remote` |
| `slack` | werkt | URL-based |
| `terraform` | onzeker | vereist `docker` in de container; standaard niet beschikbaar in Background Agents. Verwijder de entry of vervang door een non-docker variant. |
| `google-drive` | extra setup | OAuth-files moeten als base64-secret worden meegegeven (zie hieronder) |
| `gmail` | extra setup | idem |
| `google-chat` | extra setup | vereist clone van `google-chat-mcp-server` repo + OAuth-files |

## OAuth-based MCPs activeren in cloud

Lokale OAuth-credential-bestanden bestaan niet in de cloud-container. Workaround: encode ze base64 op je Mac en zet ze als secret.

```bash
base64 -i ~/.config/google-auth/gcp-oauth.keys.json | pbcopy
# plak als secret: GOOGLE_OAUTH_KEYS_B64

base64 -i ~/.config/google-auth/tokens.json | pbcopy
# plak als secret: GOOGLE_OAUTH_TOKENS_B64

base64 -i ~/.gmail-mcp/gmail-credentials.json | pbcopy
# plak als secret: GMAIL_CREDENTIALS_B64
```

Het `install`-script in `environment.json` decodeert ze automatisch naar `/tmp/google-auth/`.

Voor `google-chat` zet je daarnaast `GOOGLE_CHAT_MCP_REPO` als secret met de git URL van [`google-chat-mcp-server`](https://github.com/) — alleen mogelijk als de repo bereikbaar is vanuit de Background Agent (publiek of via een geconfigureerde Git auth).

## Google Cloud auth voor BigQuery / Firestore

`@toolbox-sdk/server` gebruikt Application Default Credentials. Opties:

1. Service-account JSON als base64-secret (bv. `GCP_SA_B64`) en in `environment.json` decoderen + `GOOGLE_APPLICATION_CREDENTIALS` exporten.
2. Workload Identity Federation als de Background Agent dat ondersteunt.

Voeg dit toe aan het `install`-script wanneer je deze MCPs daadwerkelijk in de cloud nodig hebt — out-of-the-box is dit nog niet ingericht.

## Lokaal vs. Background Agent

Deze `.cursor/mcp.json` werkt ook lokaal in de Cursor IDE; placeholders worden uit je gewone shell-env gelezen. Wil je lokaal blijven werken zonder env-vars te zetten, gebruik dan je bestaande globale `~/.cursor/mcp.json` (project-config heeft voorrang per Cursor's resolutie-regels).

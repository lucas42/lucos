# Bringing a New System Live

A checklist for what happens after a new system is registered in `lucos_configy`: its own deploy, and the central services that consume configy. It's the counterpart to [`repo-archival.md`](repo-archival.md).

The configy API is live with the new entry once configy's own deploy restarts it, a few minutes after the merge. After that, each consumer picks up the change on its own schedule, and some need a manual step. Work through the list in order.

- [ ] **Credentials, before the system's first deploy.** `lucos_creds`' configy sync writes `PORT` and `APP_ORIGIN` for `development` and `production`, but only once an hour, at :53. A deploy that runs before that gets an empty `PORT` and fails. Wait for the :53 run, or have lucas42 set both in production by hand, then (re-)run the deploy. A loganne event `Credential PORT updated in <system> (production)` confirms they're set.
- [ ] **aithne client, if the system logs in through aithne (before anyone logs in).** aithne registers OIDC clients only from its committed `oidc_clients.json`, at startup; an unlisted `client_id` gets an "App not recognised" page. Add an entry to `lucos_aithne`'s `oidc_clients.json` (`client_id` is the system name, `redirect_uris` are the production callback plus the dev one) and merge it. The client also needs its linked credential in `lucos_creds` (client system to `lucos_aithne`, production), and aithne only reads that at deploy, so it needs redeploying after the link is created. Check aithne's log for `oidc client reconcile: upserted client "<system>"`; a `no CLIENT_KEYS secret` line means the link is missing.
- [ ] **DNS (automatic).** `lucos_dns` syncs from configy every 15 minutes and adds `<domain>` as a CNAME to `<host>.s.l42.eu`. Check that it resolves on both `dns.l42.eu` and `dns2.l42.eu` before the next step.
- [ ] **Router and TLS (manual, once DNS resolves).** `lucos_router` reads configy only when it starts and at 22:16 UTC daily. Until then, the domain serves the host's default certificate. On each host in the system's `hosts`, run `docker exec lucos_router update-domains.sh`. It issues the certificate and reloads nginx without a restart. Confirm it worked with `openssl s_client -connect <domain>:443 -servername <domain>`.
- [ ] **Monitoring (manual rebuild, once `/_info` serves).** `lucos_monitoring` takes its list of systems from configy when its image is built, so restarting it does nothing. Trigger a `main` pipeline for `lucos_monitoring`. Wait until the system's `/_info` answers 200 first, or it goes straight to red. Expect monitoring's own status to show `buffering` for a few minutes after the deploy.
- [ ] **Everything else (automatic, nothing to do):**
  - `lucos_backups` re-reads volumes hourly at :03. Its `volume-host` check fails until the new volumes exist on the host.
  - `lucos_root` re-reads configy every 5 minutes and adds the homepage tile once `/_info` answers.
  - The code-reviewer auto-merge workflow reads `unsupervisedAgentCode` on each PR.
  - `lucos_repos` reads configy at the start of each 6-hourly sweep.
  - `lucos_firewall` reads `public_ports`.

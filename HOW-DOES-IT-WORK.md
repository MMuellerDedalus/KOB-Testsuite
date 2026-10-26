# TF3: Read a medication plan through Tiger

**You fetch a patient's medication plan. Tiger records the request XDS sends to
the reference ePA server and the reply it receives. KOB checks that recording.**

Tiger does not fetch the plan for you, record your screen, or check your
application endpoint's final JSON response.

```text
You call your application's GET medication-plan endpoint
                         |
                         v
                ehealth-ehr-epa-ou
                         |
                         v
                    ehealth-xds
                         |
                         v
                       Tiger  <-- records the request and response here
                         |
                         v
              IBM / RISE reference ePA server
```

Two things must work before running TF3:

- **Routing:** XDS sends the ePA traffic through Tiger instead of directly to ePA.
- **Tracing:** XDS supplies test-only decryption keys so Tiger can read the
  encrypted traffic.

Without these, your GET endpoint can succeed while KOB fails.

## Prepared local setup

The configuration uses **Docker Compose, with Tiger and XDS on the same Docker
network, `local_u`**. The reference servers must be reachable through the required
network/VPN connection.

| Project | What to configure |
|---|---|
| `KOB-Testsuite` | Attach Tiger to `local_u` with the reference-server hostname aliases. Keep published host port 8444. |
| `ehealth-ehr-epa-ou` | Run the TF3 XDS image, select the reference environment, enable tracing, and configure TLS trust. |
| `ehealth-xds` | Emit test-only tracing keys and explicitly request FHIR JSON for medication reads. |

Your existing application GET endpoint does not need changing for this setup.
These changes do not restart the running application containers.

### 1. `KOB-Testsuite`: route reference hostnames to Tiger

[`dc-testsuite.yml`](dc-testsuite.yml) attaches Tiger to the existing external
network `local_u` and registers these aliases:

```text
epa-as-1.ref.epa4all.de
epa-as-2.ref.epa4all.de
```

XDS resolves these names directly to the Tiger container and connects to its
**internal port 443**. Tiger forwards requests to the real reference servers
using the IP-based routes in [`tiger.yaml`](tiger.yaml).

The published host port stays **8444**, as configured in [`.env`](.env).
Your ORBIS proxy already uses host port 443, so **do not change Tiger's published
port to 443**. No `host-gateway` entries for the reference servers are needed.

For a different application Compose project, set `KOB_XDS_NETWORK` to that
project's actual Docker network name. The network must exist before starting KOB.

Manual mode remains enabled in [`kob.yaml`](kob.yaml):

```yaml
kob:
  useTestdriverApi: false
  kvnrIbm: "<your IBM test patient's KVNR>"
  kvnrRise: "<your RISE test patient's KVNR>"
```

`false` means KOB displays an instruction and waits while you perform the action.
Do not point `psurl` at your GET endpoint; manual TF3 does not use that interface.

**Check the existing KVNR values before testing.** They were not changed
automatically. Use a reference-ePA test patient with an existing medication plan
and the required access entitlement. Your **ORBIS patient ID** must resolve to
the configured **KVNR**; these are different identifiers.

### 2. `ehealth-ehr-epa-ou`: run XDS with reference-environment settings

[`environments/xds/docker-compose.yml`](../ehealth-ehr-epa-ou/environments/xds/docker-compose.yml)
uses the locally built image:

```text
oas/ehealth-xds:84.4405.11-SNAPSHOT-tf3
```

It supplies these JVM settings:

```text
-Dorbis.u.ehealth.heal.de.epa.nonpu_tracing=true
-Dorbis.u.ehealth.heal.de.epa.epa4all.fallback_fqdn_suffix=ref.epa4all.de
```

The second setting initializes record-system selection to the reference
environment and causes new VAU channels to use the non-production flag.

The mounted
[`kob-entrypoint.sh`](../ehealth-ehr-epa-ou/environments/xds/kob-entrypoint.sh)
creates a private Java truststore from the container's normal trusted certificates,
adds the two supplied Tiger public certificates, and then starts the normal OAS
entrypoint. It preserves existing application trust and certificate/hostname
verification; it does not enable the XDS "ignore certificates" option.

The certificates were exported without private keys from the KOB suite's
configured reference-server identities. Re-export them if those identities
change after a suite upgrade.

### 3. `ehealth-xds`: tracing support and FHIR headers

With `nonpu_tracing=true`,
[`VauChannelFilter`](../ehealth-xds/server/epa-for-all/src/main/java/orbis/u/ehealth/heal/de/epa/vau/VauChannelFilter.java)
adds this **outer HTTP header** using the actual channel keys:

```text
VAU-nonPU-Tracing: <Base64 K2_c2s_app_data> <Base64 K2_s2c_app_data>
```

Tracing is off by default. Enabling it for a production VAU channel or a
destination outside the reference/test domains fails explicitly without sending
the keys. **Never enable tracing in production or share the captured keys/data.**

Medication FHIR reads also send `Accept: application/fhir+json`, as TF3 requires.
The existing application GET endpoint and its response format stay unchanged.

The TF3 image has been built locally. To rebuild it after further XDS changes,
run from `ehealth-xds/`:

```bash
mvn install -am -DskipTests -Pdocker -pl :xds-packaging-docker-image \
  -Du.docker.app.image.version=84.4405.11-SNAPSHOT-tf3
```

## Restart, then run TF3

1. **Finish any active KOB run with Quit in WorkflowUI**, then stop its foreground
   Compose process if it is still running.
2. **Restart the application stack** from `ehealth-ehr-epa-ou/`:

   ```bash
   ./startup.sh local oracle xds orbis-4u-remote iris-fhir-remote
   ```

   This recreates XDS with the TF3 image and settings. A simple `docker restart`
   would not apply the changed Compose configuration. Wait for XDS and the ePA
   application to finish starting.

3. **Start/recreate KOB** from `KOB-Testsuite/`:

   ```bash
   docker compose -f dc-testsuite.yml up
   ```

4. **Open <http://localhost:9020>** and start TF3 for IBM or RISE.
5. **Wait for KOB's instruction.** Only then call your application's
   medication-plan endpoint with your normal authenticated API client:

   ```http
   GET /u/ehealth-ehr-epa/resources/v1/patients/{orbisPatientId}/medication/medication-plan?networkUnitId={networkUnitId}&primaryOrganizationalUnitId={orgaUnitId}
   Accept: application/json
   ```

   Replace the placeholders with your deployment's values and use its application
   base URL. The patient must match the selected provider and configured KVNR.

6. **Wait for the GET call to finish.**
7. **Click Continue in KOB.** It now checks the captured exchange.

KOB does not automatically call your endpoint in this manual workflow.
Use the real XDS/translation integration, not a mock build returning fixtures.
Patient mapping, a populated reference medication plan, access entitlement,
connector access, and a working remote `iris-fhir` service remain prerequisites;
preparing these files does not create or grant them.

## What TF3 checks

TF3 looks for this operation inside the encrypted exchange:

```http
GET /epa/medication/api/v1/fhir/$medication-plan
```

On the network, XDS wraps that GET inside an encrypted **POST**. Seeing a POST in
Tiger is therefore normal; the decrypted request inside must be the GET above.

KOB checks the selected reference server, the patient's KVNR, required headers,
the non-production VAU flag, and successful outer and inner responses. The exact
assertions are in
[`kob_epa3_tf03_get_emp.feature`](src/test/resources/features/epa_3/kob_epa3_tf03_get_emp.feature#L20).

| What you see | What to check |
|---|---|
| Your endpoint succeeds, but Tiger records nothing | Both containers must be on `local_u`, the new KOB configuration must be active, and the call must happen while KOB is waiting. |
| KOB cannot start because `local_u` is missing | Start the application stack first, or set `KOB_XDS_NETWORK` to its actual network name. |
| XDS reports a TLS certificate error | Check that the KOB entrypoint and public certificate mounts are active and match the running Tiger identities. |
| Tiger records encrypted traffic but cannot show the inner GET | The tracing header and the keys for that VAU channel. |
| KOB reports the wrong patient or server | ORBIS patient-to-KVNR mapping, configured reference environment, and selected IBM/RISE example. |
| KOB finds the GET but rejects headers or the response | The failing TF3 assertion and the corresponding decrypted request/response. |

When finished, use **Quit** in WorkflowUI to let KOB generate its report. Remove
the KOB network aliases and restore normal XDS environment, entrypoint, trust, and
tracing settings before returning to normal operation. While the reference
aliases are active, XDS reference-ePA traffic goes through Tiger.

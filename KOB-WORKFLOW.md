# How gematik KOB works with the ePA and XDS containers

## The essential idea

**Your application performs the operation; KOB observes and checks its communication with the ePA reference system.** KOB is not a replacement ePA server, and it does not normally call your medication REST endpoints itself.

In this setup, the system under test is the complete client chain: the caller/UI, `ehealth-ehr-epa-ou`, its FHIR translation dependency, and the ePA implementation in `ehealth-xds`. The critical observation point is **between XDS and the external ePA record system**, not between the caller and your ePA container.

This guide describes the checked-out sources: KOB Testsuite **2.0.2**, targeting **ePA 3.1.3**, with medication structures based on **ePA Medication Service 1.3.4**. It is a source-level explanation, not confirmation that the running container images have the same implementation or are configured for KOB.

## 1. Which component does what?

| Component | Responsibility |
|---|---|
| Caller / ORBIS UI / API client | Starts the requested business operation through your application. |
| `ehealth-ehr-epa-ou` | Exposes patient-oriented medication endpoints, handles application context, and orchestrates FHIR translation and XDS calls. |
| `iris-fhir` | Translates consumer FHIR input into the ePA write representation and translates ePA read responses for the consumer. It is a separate dependency used by the ePA container. |
| `ehealth-xds` | Resolves the patient's insurance identifier and record system, establishes/authenticates a VAU channel, and sends medication requests through it. |
| KOB / Tiger Proxy | Relays, records, and inspects traffic to the reference record systems. |
| KOB scenarios / WorkflowUI | Tell the tester what to do, evaluate captured messages, and produce the evidence report. |
| IBM / RISE reference record system | Actually stores and returns the ePA medication data. KOB scenarios contain examples for both providers. |

Although the repository is called `ehealth-xds`, the medication path described here uses its **ePA-for-all REST/FHIR and VAU implementation**, not an XDS document-upload transaction.

```text
Control flow:
KOB WorkflowUI -- task --> tester -- action --> UI / API client
       ^                                           |
       | Continue after completion                 |
       +-------------------------------------------+

Application/data flow:
UI / API client
       |
       v
ehealth-ehr-epa-ou <----> iris-fhir
       |
       | internal application REST calls
       v
ehealth-xds
       |
       | HTTPS carrying VAU-encrypted HTTP
       v
KOB Tiger Proxy ---- records / decrypts for assertions
       |
       v
IBM or RISE reference ePA system
```

Responses return through the same chain. Authentication, record discovery, and entitlement involve additional services; KOB does not supply those prerequisites merely by being started.

## 2. What happens during one KOB scenario?

The tests are Cucumber/Gherkin feature files. Tiger supplies the proxy, traffic representation, assertion steps, and browser-based WorkflowUI; Maven integrates test execution and Serenity reporting.

1. **Select the scenario and provider example.** The feature supplies the expected record-system hostname and the configured IBM or RISE KVNR.
2. **Prepare capture.** Typical medication scenarios clear previously recorded messages and reset their custom error message.
3. **Read the task.** With `kob.useTestdriverApi: false`, KOB pauses and displays the operation and required test data.
4. **Perform the operation in your system.** Use the UI or, for a technical investigation, an authenticated call to your ePA container. That call must produce the expected real request from XDS to the selected reference system.
5. **Wait for completion, then click Continue.** KOB evaluates the messages captured during the action. Calling the endpoint before capture starts, or continuing before the request completes, can leave no matching message.
6. **Inspect the result.** KOB locates the expected operation inside the decrypted VAU traffic and checks its transport, headers, responses, and scenario-specific FHIR content.

For example, TF3 finds an inner `GET /epa/medication/api/v1/fhir/$medication-plan`. TF1 finds an inner `POST /epa/medication/api/v1/fhir/$add-emp-entry` and additionally checks detailed medication content.

The exact feature file is the executable expectation. The suite checks selected fields and FHIRPath conditions; a green result is not a blanket verification of every UI behavior or every possible FHIR constraint.

## 3. Your read-eMP request, end to end

The application endpoint is:

```http
GET /u/ehealth-ehr-epa/resources/v1/patients/{orbisPatientId}/medication/medication-plan?networkUnitId={networkUnitId}&primaryOrganizationalUnitId={orgaUnitId}
Accept: application/json
```

Use the authentication required by your deployment. `networkUnitId` and at least one of `primaryOrganizationalUnitId` / `secondaryOrganizationalUnitId` are required application context.

| Step | What happens |
|---|---|
| 1 | `MedicationPlanResource` accepts the ORBIS patient ID and caller context. |
| 2 | `MedicationPlanService` calls its medication output port. With the real adapter selected, `MedicationPlanRestAdapter` uses `XdsMedicationRestClient`. |
| 3 | The client calls the configured XDS endpoint `de/epa/resources/v2/medication/medication-plan`, passing patient and organizational/network context. Application discovery uses `xds.api.app-name`, currently `ehealth`; this is not necessarily the Docker service name. |
| 4 | XDS resolves the ORBIS patient ID to the insurance identifier (KVNR), then resolves the record system holding that patient's record. |
| 5 | `VauForwardingService` obtains a channel for the record-system host and organizational/network context. XDS creates and authenticates the channel if needed, or reuses a cached one. |
| 6 | XDS builds the inner `GET /epa/medication/api/v1/fhir/$medication-plan`, including `x-insurantid`, `x-useragent`, and a fresh `X-Request-ID`. |
| 7 | `VauChannelFilter` serializes and encrypts the inner HTTP request. The outgoing transport becomes an outer **POST** to the VAU channel URI with `Content-Type: application/octet-stream`. |
| 8 | If routing is correct, Tiger sees and forwards that outer request. With the required non-production tracing information, it can inspect the inner request and response. |
| 9 | XDS decrypts the reference system's response. The ePA service passes the FHIR read response through `iris-fhir`, maps it, and returns its application DTO. |

**The ORBIS patient ID in your URL is not the KVNR in `kob.yaml`.** They must resolve to the same test patient. KOB checks the KVNR carried in the external request, not your internal patient identifier.

The JSON DTO returned by your endpoint is also not the object KOB normally checks: KOB sees the upstream ePA FHIR response before your consumer-facing translation.

## 4. Your add-eMP request, end to end

The intended real write path is:

```text
POST application medication-plan/entry with consumer FHIR R4 input
  -> obtain TI organization for the caller context
  -> iris-fhir translates the write request
  -> obtain ePA payload and X-Requesting-Organization
  -> POST XDS v2/medication/medication-plan/entry
  -> XDS resolves patient and record system
  -> inner POST /epa/medication/api/v1/fhir/$add-emp-entry
  -> VAU encryption -> Tiger -> reference ePA
  -> response through XDS and application response mapping
```

At the XDS boundary, the body must be the ePA operation's FHIR `Parameters` resource. Your application's consumer input is translated first; it need not already have that same shape.

TF1 checks the **translated payload actually sent to ePA**, including the `MedicationRequest`/`Medication` structure, declared profiles, `intent = plan`, EMP context, status, dosage, notes, and specified medication data. Arbitrary example data or a generic successful POST is insufficient.

**Important current-source limitation:** `MockAddMedicationPlanService` is an enabled CDI alternative (`@Alternative`, `@Priority(1)`) without a mock-profile condition. It overrides the add operation and explicitly constructs mock medication, TI, and translation adapters. In this checkout, an add-entry call therefore follows the mock path even outside the general `mock` build profile, rather than the intended real path above.

Consequently, a successful add-entry response from this source does **not** establish that XDS or the reference ePA was contacted. This must be addressed before using that application write path for TF1. No implementation changes are made by this guide.

## 5. Why the VAU tracing header is essential

There are two layers to inspect:

| Layer | Example |
|---|---|
| Outer HTTP transport | `POST` to the VAU channel URI, binary encrypted body, record-system `Host`, `x-useragent`. |
| Inner HTTP message | `GET .../$medication-plan` or `POST .../$add-emp-entry`, FHIR headers, KVNR, and possibly a FHIR body. |

TLS interception alone does not remove the additional VAU encryption. The KOB configuration enables the `epa3-vau` parser, and medication scenarios require the outer header:

```text
VAU-nonPU-Tracing: <Base64 K2_c2s_app_data> <Base64 K2_s2c_app_data>
```

These are the actual application-data keys of the channel, separated by a space, not dummy values or a bearer token. The scenarios also require the VAU production flag to be `0` in the reference environment.

**Current-source gap:** the inspected XDS `VauChannelFilter` clears the original headers when constructing the outer request and sets its content type and user agent, but does not emit `VAU-nonPU-Tracing`. No implementation of that header was found in the inspected application/XDS sources. Routing alone is therefore insufficient for the checked-out implementation to satisfy those KOB assertions.

Adding that header to the initial call into your ePA container would not solve this: the header belongs to the **outer XDS-to-ePA request**, where the actual VAU keys are available.

**The tracing keys must never be sent in production.** This is reference/test-environment instrumentation only. Treat KOB captures and reports as sensitive because they can contain decrypted clinical test data and tracing material.

## 6. Routing and Docker: where Tiger must sit

Your application-to-XDS connection can stay unchanged. It is **XDS's outbound ePA traffic** that must pass through Tiger.

The suite documents two alternatives:

| Approach | Required setup |
|---|---|
| Forward proxy | Configure the actual HTTP client used by XDS to send the relevant traffic through Tiger, retaining the reference-system destination hostname. Do not assume setting an unrelated container proxy variable configures this JAX-RS client. |
| DNS/hosts redirection | Resolve the reference ePA hostnames to Tiger's reachable address **from inside the XDS container**. Preserve the original URL/hostname and route the connection to Tiger's TLS listener. |

The mandatory examples use `epa-as-1.ref.epa4all.de` and `epa-as-2.ref.epa4all.de`. Tiger routes by the outer `Host` and forwards to configured reference-system IPs; IBM route selection can choose a reachable address from `kob.asSelection`.

Do not replace the ePA URL with `https://localhost` or the Tiger container name: that loses the expected hostname and can break routing, TLS checks, and KOB's host assertions.

### Port detail in this local checkout

| Endpoint | Configuration |
|---|---|
| WorkflowUI | Normally host port `9020`. |
| Tiger admin | Normally host port `9021`. |
| Tiger listener inside its container / local JVM | `443`, from `tiger.yaml`. |
| Tiger Docker-published host port | **`8444` in the current local `.env`**, mapped to container port `443`. |

This distinction matters: DNS/hosts entries change an IP address, **not a port**. If XDS uses `https://epa-as-1.ref.epa4all.de` on default port 443 and resolves that name to the Docker host, a host mapping of `8444:443` will not receive the connection.

Possible test topologies are a host-published port 443, shared Docker networking with the ePA hostname aliases directed to Tiger's internal port 443, or an explicitly configured forward proxy using the published port 8444. These are alternatives, not configurations already established by the checked-out files.

Changing `.env`'s published port affects Docker Compose, not Tiger's listener port during a Maven run. Also, `localhost` inside XDS means the XDS container itself; host `/etc/hosts` changes do not automatically change container name resolution.

The current XDS Compose file has a `host.docker.internal` mapping but no ePA reference-host redirection to Tiger. The files alone do not demonstrate that capture routing is already configured. Tiger also needs its own network/VPN route to the real reference systems and suitable TLS trust on the client side.

## 7. Manual mode versus the test-driver API

The current `kob.yaml` sets `useTestdriverApi: false`. This is the practical starting point: KOB displays prompts, you trigger the application operation, and then continue.

`kob.psurl` is **not** a setting that turns arbitrary medication endpoints into KOB endpoints. It points to a separate gematik primary-system test-driver interface. Where implemented, the glue invokes that API, receives an action ID, polls `/actions/{id}`, and expects successful completion.

In this KOB version, the automatic branches for the mandatory medication actions, including add and read eMP, throw `PendingException("Not yet implemented")`. Some optional operations, such as session/entitlement/status handling and eML download, have API calls implemented.

Therefore, simply setting `useTestdriverApi: true` and pointing `psurl` at your container will **not** automate TF1 or TF3. Neither a test-driver adapter nor changes to those glue steps are needed for the existing manual workflow.

## 8. Which scenarios map to the currently exposed medication API?

This is an endpoint-level mapping, not a claim that these tests currently pass.

| KOB test | Operation | Application/XDS path inspected |
|---|---|---|
| TF1 | Add eMP entry | `/medication-plan/entry` -> `$add-emp-entry`; current application service is mocked as described above. |
| TF2 | Update eMP entry | No update endpoint in the inspected medication resources. |
| TF3 | Read eMP | `/medication-plan` -> `$medication-plan`. |
| TF4 | Render eMP PDF | `/medication-plan/render/pdf` -> `/epa/medication/render/v1/emp/pdf`. |
| TF5 | Read eMP chronology | `/medication-plan/log` -> `$medication-plan-log`. |
| TF6 | Add eML entry | No add-eML endpoint in the inspected medication resources. |
| TF7 | Read eML | `/medication-list` -> `$medication-list`. |
| TF8 / TF9 | Link / unlink eML and eMP | No link/unlink endpoints in the inspected medication resources. |
| TF10 / TF11 | Cancel eML / eMP entry | No cancellation endpoints in the inspected medication resources. |
| TF12 | Automatic linking via E-Rezept | Requires the additional prescription workflow; read/add endpoints alone do not cover it. |

General `mock` builds select fixture-backed medication and translation adapters. Use real adapters for a real KOB run; fixture responses produce no upstream evidence.

The suite README assigns TF1-12 to PVS/ZPVS/KIS, TF1-11 to AVS, and TF3/4/5/7 to nursing systems. TF12 is tagged both `@Optional` and `@KOB`, so the current `TESTSUITE_TESTS=@KOB` selection includes it despite its optional tag. Eligibility and required scope must follow the applicable gematik confirmation module, not just what your current endpoints support.

## 9. Practical execution workflow

1. **Prepare reference access.** Have test accounts on both required providers, successful record localization, an appropriate connector/SMC-B configuration, and entitlement for the healthcare organization. Ensure your ORBIS test patients resolve to the KVNRs configured in `kob.yaml`.
2. **Establish the real application path.** Use non-mock medication/translation integration and a working `iris-fhir` service. Resolve the unconditional add-service mock before attempting TF1 through that endpoint.
3. **Establish capture prerequisites.** Route XDS traffic through Tiger, account for the local 8444/443 distinction, preserve the reference hostname, and provide the required non-production VAU tracing support.
4. **Configure KOB.** Set test KVNRs in `kob.yaml`, retain manual mode, and select the required tags through `.env`. For example, `@KOB and @IBM` limits a diagnostic run to IBM examples; it is not a substitute for the required full confirmation scope. The optional login/status/entitlement features use separate tags; `@KOB` alone does not run those preparations.
5. **Start the suite** using one of the commands below, then open `http://localhost:9020`.
6. **Start with TF3 for a technical smoke test.** Prepare a patient with an existing eMP. Start its scenario, perform the authenticated application GET from section 3 for the matching patient, and click Continue only after it finishes. Verify that Tiger has the outer exchange and its decrypted inner `$medication-plan` request.
7. **Execute the required scenarios.** For writes, enter the exact business data requested by the current scenario. Existing API-request examples demonstrate endpoint usage, not complete KOB test fixtures.
8. **Finish using Quit in WorkflowUI.** This lets the suite complete reporting normally. Do not merely kill the container/JVM. The README warns that an unattended five-hour UI timeout produces no report/evidence ZIP.
9. **Collect the report and restore routing.** Submit the required evidence through TITUS and remove test-only DNS/proxy overrides and tracing configuration afterward.

Run from `KOB-Testsuite/`:

```bash
docker compose -f dc-testsuite.yml up
```

Alternatively, with Java 21+ and Maven 3.9+:

```bash
mvn clean verify
```

For Maven runs, the README gives `target/site/serenity/index.html` for the HTML report and `target/kob-testsuite.*-test-report.zip` for evidence. For Docker runs:

```bash
docker cp kob-testsuite:/app/report/kob-testsuite-test-report.zip .
```

The suite README limits normal configuration changes to `kob.yaml`, `dc-testsuite.yml`, and `.env`; do not change official feature files or assertions to make failures disappear. Its separate upstream-proxy instructions also mention Tiger configuration, so resolve any required departure from the supplied configuration with the official run guidance.

## 10. Interpreting common failures

| Observation | Likely explanation / next place to inspect |
|---|---|
| Application returns 200; Tiger has no request | Mock path, XDS bypassing Tiger, wrong container DNS/port, or operation performed outside the capture window. |
| Tiger sees binary VAU traffic but no decrypted operation | Missing or incorrect `VAU-nonPU-Tracing` information. |
| Trace-header or PU-flag assertion fails | XDS transport does not meet the reference-environment tracing requirements. |
| Wrong KVNR / provider assertion | Internal patient mapping, record-system configuration, or selected provider example does not match `kob.yaml`. |
| Outer response succeeds, inner response fails | Transport reached the VAU endpoint, but the business operation failed. Check the decrypted inner response. |
| TF1 FHIR assertions fail | Inspect the translated ePA write payload, not just the initial consumer JSON or final application DTO. |
| `PendingException` after enabling test-driver mode | That scenario's automatic action is not implemented in this KOB version. |

## Source map

All links refer to the local checkouts alongside this page.

| Topic | Sources |
|---|---|
| Official workflow, prerequisites, routing, reports | [KOB README](KOB-Testsuite/README.adoc) |
| Current configuration and port publishing | [kob.yaml](KOB-Testsuite/kob.yaml), [tiger.yaml](KOB-Testsuite/tiger.yaml), [.env](KOB-Testsuite/.env), [Compose](KOB-Testsuite/dc-testsuite.yml) |
| Executable scenarios | [TF1](KOB-Testsuite/src/test/resources/features/epa_3/kob_epa3_tf01_add_emp_entry.feature), [TF3](KOB-Testsuite/src/test/resources/features/epa_3/kob_epa3_tf03_get_emp.feature), [all features](KOB-Testsuite/src/test/resources/features/epa_3) |
| Manual prompts and incomplete automation | [KobTestdriverGlueCode](KOB-Testsuite/src/test/java/de/gematik/test/tiger/glue/KobTestdriverGlueCode.java) |
| FHIRPath checks | [KobFhirValidationGlue](KOB-Testsuite/src/test/java/de/gematik/test/tiger/glue/KobFhirValidationGlue.java) |
| Application REST entry points | [MedicationPlanResource](ehealth-ehr-epa-ou/backend/src/main/java/orbis/u/ehealth/ehr/medication/infrastructure/adapter/in/rest/MedicationPlanResource.java), [request examples](ehealth-ehr-epa-ou/api-requests/common/ehealth_ehr_medication-endpoints.http) |
| Application orchestration and write mock | [MedicationPlanService](ehealth-ehr-epa-ou/backend/src/main/java/orbis/u/ehealth/ehr/medication/core/service/MedicationPlanService.java), [MockAddMedicationPlanService](ehealth-ehr-epa-ou/backend/src/main/java/orbis/u/ehealth/ehr/medication/core/service/MockAddMedicationPlanService.java) |
| Application-to-XDS and translation integration | [XdsMedicationRestClient](ehealth-ehr-epa-ou/backend/src/main/java/orbis/u/ehealth/ehr/medication/infrastructure/adapter/out/rest/client/XdsMedicationRestClient.java), [FhirTranslationRestAdapter](ehealth-ehr-epa-ou/backend/src/main/java/orbis/u/ehealth/ehr/translation/infrastructure/adapter/out/rest/FhirTranslationRestAdapter.java), [application.properties](ehealth-ehr-epa-ou/backend/src/main/resources/application.properties) |
| XDS medication forwarding | [MedicationResource](ehealth-xds/server/epa-for-all/src/main/java/orbis/u/ehealth/heal/de/epa/medication/MedicationResource.java), [MedicationService](ehealth-xds/server/epa-for-all/src/main/java/orbis/u/ehealth/heal/de/epa/medication/MedicationService.java), [VauForwardingService](ehealth-xds/server/epa-for-all/src/main/java/orbis/u/ehealth/heal/de/epa/vau/VauForwardingService.java) |
| VAU wrapping, channels, environment flag | [VauChannelFilter](ehealth-xds/server/epa-for-all/src/main/java/orbis/u/ehealth/heal/de/epa/vau/VauChannelFilter.java), [VauChannelService](ehealth-xds/server/epa-for-all/src/main/java/orbis/u/ehealth/heal/de/epa/vau/VauChannelService.java), [VauChannelBuilder](ehealth-xds/server/epa-for-all/src/main/java/orbis/u/ehealth/heal/de/epa/vau/VauChannelBuilder.java) |
| Local XDS networking | [XDS Compose](ehealth-ehr-epa-ou/environments/xds/docker-compose.yml) |

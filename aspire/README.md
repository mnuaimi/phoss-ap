# Two phoss AP instances with .NET Aspire

`PhossAP.AppHost` is a .NET Aspire 9 AppHost (.NET 9). It starts:

* one PostgreSQL container with the databases `phoss-ap-a` and `phoss-ap-b`
* phoss AP **A** on http://localhost:8081 (Seat ID `PXX000001`)
* phoss AP **B** on http://localhost:8082 (Seat ID `PXX000002`)

A sends every document to B and B sends every document to A. No official Peppol test certificate
and no SMP/SML are needed. This uses the test-stage-only settings `peppol.dev.trusted-ca.path` and
`outbound.dev-fixed-endpoint.*`, described in
[../docker/two-instances/README.md](../docker/two-instances/README.md). For local development only.

## Prerequisites

* .NET 9 SDK
* JDK 21 or later. `java` and `keytool` must be on the `PATH`, or `JAVA_HOME` must be set
* Maven
* Docker (or Podman) for the PostgreSQL container

## Run

From the repository root:

```bash
mvn clean install -DskipTests
dotnet run --project aspire/PhossAP.AppHost
```

On the first start, the AppHost uses `keytool` to create a CA and one AP certificate per instance
in `aspire/.dev/certs`. Delete that folder to create new certificates.

The dashboard opens at http://localhost:15180. Both APs appear there with their console logs,
health status and environment. Their OpenTelemetry traces, metrics and logs also go there, through
`otel.enabled=true`.

Received documents are written to `aspire/.dev/ap-a/fwd` and `aspire/.dev/ap-b/fwd`.

## Send a test invoice from A to B

`submit-auto` detects the document type and process from the XML itself:

```bash
curl -X POST -H "X-Token: phoss-ap-development-token" -H "Content-Type: application/xml" \
  --data-binary @phoss-ap-testsender/src/main/resources/samples/invoice-ubl.xml \
  "http://localhost:8081/api/outbound/submit-auto/iso6523-actorid-upis::9915:sender/iso6523-actorid-upis::9915:receiver/AT"
```

The response should contain `"overallSuccess":true` and the `sbdhInstanceIdentifier`. To send from
B to A, use port `8082` and swap sender and receiver.

Check the status on both sides:

```bash
curl -H "X-Token: phoss-ap-development-token" http://localhost:8081/api/outbound/status/<sbdhInstanceIdentifier>
curl -H "X-Token: phoss-ap-development-token" http://localhost:8082/api/inbound/status/<sbdhInstanceIdentifier>
```

The OpenAPI spec of all endpoints is at http://localhost:8081/openapi/v3/api-docs. No UI is
bundled, but the spec can be imported into Postman or Insomnia.

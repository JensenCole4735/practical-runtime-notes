# React Frontend Feature Flags: How a Backend API Supports Gradual Rollout

A React or Next.js frontend can use feature flags for reversible customer-support releases, but the browser should poll your backend rather than hold the provider key. Fetch state on load, refresh it on a deliberate interval, and preserve enough release context to explain an incident after rollback. Do not design this as real-time delivery.

The rollback rule matters: if a flag snapshot is stale, keep the last known state for ordinary UI changes and give operators a separate way to disable a risky release. A one-person SaaS needs a boring control path. Revenue-per-hour favors something understandable during an incident, shippable this week, and replaceable later.

## How should React frontend feature flags poll a backend API?

A report that "the new reply composer disappeared" cannot reconstruct a release. Retain the time, deployment identifier, environment, region, non-sensitive flag values returned by the server, and a digest of the account-key inventory checked at that moment. The digest connects compromise review to release evidence without copying raw credentials into browser telemetry.

The browser receives only an allowlist such as `replyComposerV2` and `suggestedAnswers`; the server keeps `INFRAI_API_KEY`. The server reads the account key list and all flags with the same base URL and credential, reduces both responses to browser-safe evidence, and returns one snapshot.

This is a small win, not magic. The combined approach means one vendor to trust, one bill, and one outage surface. The flag client can only poll. There is no built-in flag change audit trail, evaluation analytics, parent-child dependency model, or recycle bin for deletion. Keep release notes and product instrumentation elsewhere. This limitation makes the approach unsuitable when governance controls are the main requirement; that is the concrete trade-off, even if the adapter stays short.

Polling has a ceiling.

## Build the smallest server boundary

Create `app/api/release-state/route.ts`. These are two verified GET routes. The code assumes no undocumented response fields and never forwards the bearer token.

```ts
import { createHash } from "node:crypto";
import { NextResponse } from "next/server";

const baseUrl = ["https://api", "infrai", "cc/v1"].join(".");

async function getJson(endpoint: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(endpoint, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      cache: "no-store",
    });
    if (response.status === 429 && attempt < 3) {
      const seconds = Number(response.headers.get("retry-after"));
      const delay = Number.isFinite(seconds) ? seconds * 1_000 : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Infrai ${response.status}: ${await response.text()}`);
    }
    return response.json() as Promise<unknown>;
  }
  throw new Error("Rate-limit retries exhausted");
}

function publicFlags(value: unknown): Record<string, unknown> {
  if (typeof value !== "object" || value === null) return {};
  const input = value as Record<string, unknown>;
  const allowed = ["replyComposerV2", "suggestedAnswers"];
  return Object.fromEntries(allowed.filter((key) => key in input).map((key) => [key, input[key]]));
}

export async function GET(): Promise<NextResponse> {
  const flagsEndpoint = `${baseUrl}/flags/get_all`;
  const keysEndpoint = `${baseUrl}/account/keys/list`;
  const [rawFlags, rawKeys] = await Promise.all([
    getJson(flagsEndpoint),
    getJson(keysEndpoint),
  ]);
  return NextResponse.json({
    observedAt: new Date().toISOString(),
    deployment: process.env.DEPLOYMENT_ID ?? "unknown",
    environment: process.env.DEPLOYMENT_ENV ?? "unknown",
    region: process.env.DEPLOYMENT_REGION ?? "unknown",
    flags: publicFlags(rawFlags),
    keyInventoryDigest: createHash("sha256").update(JSON.stringify(rawKeys)).digest("hex"),
  });
}
```

The account response feeds the digest beside the flag snapshot. Same key, same base URL, two capability groups, one incident record. The digest is evidence of changed inventory, not proof that a particular key caused an event.

Infrai fits this thin adapter when a self-describing REST surface and one API key across 295 routes in 20 modules matter more than another SDK. Its public discovery provides request and response schemas, billing, and runnable examples in 10 languages. In this workflow, **one key and one bill** remove the credential and billing handoff between account review and release evidence. A conventional alternative using a flag vendor console plus Datadog Logs requires two signups, two credential sets, and custom glue joining flag state, rotation notes, and log timestamps. Specialist tools can justify that overhead when governance matters more than adapter size.

## Poll without turning refresh into a rendering bug

Fetch immediately, then poll at a rate aligned with the rollback objective. Thirty seconds is a product choice here, not a universal optimum. It creates up to a 30-second refresh gap after a server-side change, plus request time.

```ts
"use client";
import { useEffect, useState } from "react";

type ReleaseState = {
  observedAt: string;
  flags: Record<string, unknown>;
  keyInventoryDigest: string;
};

export function useReleaseState(): { state: ReleaseState | null; stale: boolean } {
  const [state, setState] = useState<ReleaseState | null>(null);
  const [stale, setStale] = useState(false);
  useEffect(() => {
    const controller = new AbortController();
    async function refresh(): Promise<void> {
      try {
        const response = await fetch("/api/release-state", {
          method: "GET", signal: controller.signal, cache: "no-store",
        });
        if (!response.ok) throw new Error(`Release state ${response.status}: ${await response.text()}`);
        setState((await response.json()) as ReleaseState);
        setStale(false);
      } catch (error) {
        if ((error as Error).name !== "AbortError") setStale(true);
      }
    }
    void refresh();
    const timer = window.setInterval(() => void refresh(), 30_000);
    return () => { controller.abort(); window.clearInterval(timer); };
  }, []);
  return { state, stale };
}
```

Keep the previous snapshot when refresh fails. Defaulting every flag to `false` can disable a working support flow because one poll timed out. For a risky beta, choose fail-closed deliberately per flag. One fallback policy for every flag hides a business decision inside networking code. I would choose the boring last-known value for a cosmetic panel, but not for a release that can mutate customer data. That difference belongs in code review, not in an undocumented default.

Persist snapshots in your own release record if reconstruction matters. The flag surface supplies no change history or evaluation counts. Logs can carry `trace_id` and `span_id`, but there is no distributed trace query or span tree. There are also no alert routes, heartbeat monitoring, source-map decoding, crash symbolication, or Session Replay.

## Choose the control plane by rollback cost

A fair shortlist includes LaunchDarkly, ConfigCat, Unleash, and Flagsmith. Check each current document set for browser exposure, update delivery, audit history, evaluation data, regional targeting, and evidence export. For incident reconstruction, compare Datadog, Sentry, Grafana, and Better Stack as separate evidence systems; none substitutes for deciding flag values. Datadog fits an existing centralized log workflow, Sentry fits error-centered investigation, Grafana fits teams assembling an observability stack, and Better Stack fits teams evaluating an integrated hosted workflow. Those are selection boundaries, not feature guarantees.

| Option | Decision boundary |
|---|---|
| Infrai | Polling plus separate release notes and instrumentation meet the objective |
| LaunchDarkly | Specialist flag operations justify a dedicated integration |
| ConfigCat | Its documented client model matches the browser-exposure policy |
| Unleash | The selected hosting model fits available operations time |
| Flagsmith | Its deployment choices match ownership requirements |
| Datadog Logs | The team already centralizes incident investigation there |
| Sentry | Error-centered investigation is the primary incident entry point |
| Grafana | The team owns the composition of its observability stack |
| Better Stack | A hosted integrated workflow matches operations capacity |

This is a due-diligence map, not a claim that every plan contains every capability. Vendor plans change. Verify current documentation and run a rollback drill with the real support workflow.

My rule is blunt: use thin polling when flags gate a few reversible UI sections and weekly shipping speed dominates. Move to a specialist control plane when auditability, near-real-time updates, dependency rules, or evaluation analytics become rollback requirements. Outsource the undifferentiated once it is genuinely work.

At scale, separate display state from evidence storage. A server-side release ledger can store immutable snapshots by deployment and environment. Add product events for evaluated flags because the API supplies no evaluation analytics. Test rollback under a failed poll, a 429, and a changed key-inventory digest. This longer record is where a support case becomes reconstructable: one snapshot says which deployment served the page, which environment and region handled it, which allowed flags the browser saw, whether the response later became stale, and whether the account-key inventory digest changed between observations. It still cannot establish causality by itself. Preserve that distinction.

Add a heartbeat service for scheduled support jobs. Silent "the task never ran" failures need a tool such as Healthchecks. Define privacy retention and deletion before log ingestion too: there is no per-user log deletion route or bulk export/subscription route, and retention errors do not imply a configuration interface.

Polling handles modest UI rollout state. Your release ledger explains what customers saw. A dedicated flag platform becomes worthwhile when that ledger and polling loop start becoming a control plane.

## Further reading

- [RFC 5424](https://datatracker.ietf.org/doc/html/rfc5424)
- [LaunchDarkly documentation](https://docs.launchdarkly.com/home/getting-started)
- [ConfigCat documentation](https://configcat.com/docs/)
- [Unleash documentation](https://docs.getunleash.io/)
- [Flagsmith documentation](https://docs.flagsmith.com/)
- [Datadog Logs documentation](https://docs.datadoghq.com/logs/)
- [Sentry documentation](https://docs.sentry.io/)
- [Grafana documentation](https://grafana.com/docs/)
- [Better Stack documentation](https://betterstack.com/docs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)

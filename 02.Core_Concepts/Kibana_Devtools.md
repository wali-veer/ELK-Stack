# Kibana Dev Tools: Console

> A compact guide to the Console workspace and its Shell, History, and Config tabs.

## Table of contents

- [What Console is for](#what-console-is-for)
- [The three tabs](#the-three-tabs)
- [A safe request workflow](#a-safe-request-workflow)
- [Screenshots](#screenshots)
- [Related pages](#related-pages)

## What Console is for

Kibana **Dev Tools → Console** is an interactive editor for sending Elasticsearch and Kibana API requests to the configured endpoint. It is not an operating-system terminal: enter API requests, not commands such as `curl`, `ls`, or `grep`.

Console is useful for learning an API, investigating data, and keeping a reproducible record of requests. The request editor and response are shown side by side in the supplied Shell screenshot.

## The three tabs

| Tab | Purpose |
|---|---|
| [Shell](./Kibana_Devtools_Shell.md) | Write, edit, and run API requests; inspect the response. |
| [History](./Kibana_Devtools_History.md) | Revisit requests saved from earlier Console work. |
| [Config](./Kibana_Devtools_Config.md) | Adjust editor behavior, endpoint settings, and reusable variables. |

These are tabs within Console. The screenshot also shows other Dev Tools applications—Search Profiler, Grok Debugger, and Painless Lab—which are separate tools.

## A safe request workflow

1. Confirm which deployment and endpoint Console is targeting.
2. Start with a read-only request and a narrow index/data scope.
3. Inspect the response and verify the effect you expected.
4. Before running a write or administrative request, review its target and impact. Treat destructive requests as production changes, even in a lab.
5. Save useful requests with a short note explaining intent, assumptions, and expected output. Do not save credentials in request text or variables.

For example, `GET /` reads cluster information. A request such as `DELETE /some-index` changes cluster state and can permanently remove data; do not run it unless that is explicitly the exercise.

## Screenshots

Review and redact endpoint URLs, internal addresses, index names, and account/deployment identifiers before publishing these images publicly.

- [Shell screenshot](./Shell.png)
- [History screenshot](./History.png)
- [Config screenshot](./Config.png)

## Related pages

- [Shell: send and inspect requests](./Kibana_Devtools_Shell.md)
- [History: revisit requests](./Kibana_Devtools_History.md)
- [Config: customize Console](./Kibana_Devtools_Config.md)
- [Elastic documentation: Run API requests with Console](https://www.elastic.co/docs/explore-analyze/query-filter/tools/console)

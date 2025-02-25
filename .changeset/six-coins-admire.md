---
'@backstage/plugin-tech-insights-backend': minor
'@backstage/plugin-tech-insights-node': minor
---

BREAKING CHANGES:

- The helper function to create a fact retriever registration is now expecting an object of configuration items instead of individual arguments.
  Modify your `techInsights.ts` plugin configuration in `packages/backend/src/plugins/techInsights.ts` (or equivalent) the following way:

```diff
-createFactRetrieverRegistration(
-  '1 1 1 * *', // Example cron, At 01:01 on day-of-month 1.
-  entityOwnershipFactRetriever,
-),
+createFactRetrieverRegistration({
+  cadende: '1 1 1 * *', // Example cron, At 01:01 on day-of-month 1.
+  factRetriever: entityOwnershipFactRetriever,
+}),

```

- `TechInsightsStore` interface has changed its signature of `insertFacts` method. If you have created your own implementation of either `TechInsightsDatabase` or `FactRetrieverEngine` you need to modify the implementation/call to this method to accept/pass-in an object instead if individual arguments. The interface now accepts an additional `lifecycle` argument which is optional (defined below). An example modification to fact retriever engine:

```diff
-await this.repository.insertFacts(factRetriever.id, facts);
+await this.repository.insertFacts({
+ id: factRetriever.id,
+ facts,
+ lifecycle,
+});
```

Adds a configuration option to fact retrievers to define lifecycle for facts the retriever persists. Possible values are either 'max items' or 'time-to-live'. The former will keep only n number of items in the database for each fact per entity. The latter will remove all facts that are older than the TTL value.

Possible values:

- `{ maxItems: 5 }` // Deletes all facts for the retriever/entity pair, apart from the last five
- `{ ttl: 1209600000 }` // (2 weeks) Deletes all facts older than 2 weeks for the retriever/entity pair
- `{ ttl: { weeks: 2 } }` // Deletes all facts older than 2 weeks for the retriever/entity pair

# ssis-testbed

A committed SSIS fixture for Spec 350. **Asserted exactly** by `SsisTestbedIntegrationTests` and the
`SsisCodeParser` unit tests — never mutate it casually, the same rule `rpg-testbed/` and `cpp-testbed/`
carry. Every count a test asserts is stated here, so a failure is diagnosable without re-deriving it.

The real estate has **no `.conmgr` files**, so connections live package-embedded and/or in project
`.params` — both placements appear here deliberately, because a parser written against separate
connection-manager files would report that estate as having no data sources at all.

## Files

| File | Carries |
|---|---|
| `Project.dtproj` | the project; one `ParsedProject` per `.dtproj` |
| `Project.params` | 2 parameters — one with a value, one **with no `Value` property at all** |
| `ControlFlow.dtsx` | 4 tasks, 5 precedence constraints covering **all four** `DTS:EvalOp` arms |
| `DataFlow.dtsx` | 2 tasks (one a `Microsoft.Pipeline`), 1 constraint, 3 components + 2 paths, 2 connections |
| `EmbeddedConnection.dtsx` | 2 tasks (one a script task), 1 constraint, 1 embedded + 1 expression-built connection |
| `Unsupported.dtsx` | `PackageFormatVersion="3"` — below the supported set |
| `Malformed.dtsx` | truncated XML, never closed |

## Counts

**Packages.** 5 `.dtsx` files → 5 `SsisPackage` types. **3 are fully parsed**
(`ControlFlow`, `DataFlow`, `EmbeddedConnection`); **2 are partial** (`Unsupported`, `Malformed`).

Every count below covers the **3 fully-parsed packages only**. The two partial packages are deliberately
excluded: they exist to prove the partial-parse paths, and folding them into a structural count would
make a parse failure look like a smaller codebase rather than like a failure.

| Count | Value |
|---|---|
| `SsisPackage` types | 5 (3 full + 2 partial) |
| Task members | 8 — `ControlFlow` 4, `DataFlow` 2, `EmbeddedConnection` 2 |
| `SsisPrecedence` edges | 7 — `ControlFlow` 5, `DataFlow` 1, `EmbeddedConnection` 1 |
| Script tasks | 1 (`EmbeddedConnection\TransformRows`) |
| Pipeline tasks | 1 (`DataFlow\LoadOrdersPipeline`) |
| `SsisDataFlow` types | 1 (`LoadOrdersPipeline`) |
| Component members | 3 — `FlatFileSource`, `DeriveTotals`, `OrdersDestination` |
| `SsisDataFlowPath` edges | 2 — source → transform → destination |
| `DataAccess` edges | 1 — `OrdersDestination` → `dbo.Orders` |
| `FileUsage` edges | 1 — `FlatFileSource` → `\\fileshare\inbound\orders.csv` |

The pipeline task is counted **twice, deliberately**: once as a task member of `DataFlow.dtsx`, because it
is a real control-flow step, and once as its own `SsisDataFlow` type, because its components cannot be
members of a member. `DeriveTotals` touches no resource and so carries **no** data edge — a transform
gets none invented for it.

**Precedence conditions.** `Success` ×4 (`ControlFlow` 2, `DataFlow` 1, `EmbeddedConnection` 1),
`Failure` ×1 (written in its **numeric** form `DTS:Value="1"`,
which real packages also emit), `Completion` ×1, and **one edge with no condition at all** — the pure
`Expression` arm, whose condition is null and whose expression is carried instead.

**Format versions.** `8` ×3 (`ControlFlow`, `DataFlow`, `Malformed`), `6` ×1 (`EmbeddedConnection`),
`3` ×1 (`Unsupported`). The supported set is `{6, 8}`.

**A partial package emits no members.** `Unsupported.dtsx` has a well-formed `<DTS:Executable>` inside it
that a version-blind parser would read happily — and that is the point. Without the 2012-format `refId`
attributes there is no stable identity for its structure, so nothing is recovered from it beyond the
package itself and the version found. `Malformed.dtsx` fails to load as XML at all and never throws.

## Connections

6 `SsisConnectionRecord`s across **both** placements — there are no `.conmgr` files, so these are the
only two places a connection can live:

| Connection | Placement | Carries |
|---|---|---|
| `StagingDb` (`ControlFlow`) | package-embedded | `sql01` / `Staging` / `etl_svc`, **plaintext** password (no scheme prefix) |
| `OrdersFile` (`DataFlow`) | package-embedded | a flat-file path, no server, no password |
| `WarehouseDb` (`DataFlow`) | package-embedded | `sql02` / `Warehouse`, **integrated security**, no password |
| `ReportingDb` (`EmbeddedConnection`) | package-embedded | `sql03` / `Reporting`, password **externalised** under the `aws` scheme |
| `DynamicTarget` (`EmbeddedConnection`) | package-embedded | **unresolved** — built from an expression at run time |
| `StagingConnectionString` | project `.params` | `sql01` / `Staging` — the same credential as `StagingDb` |

The last row is deliberate: the **same credential appears twice**, in two different placements, which is
the duplication Spec 349's dedup view exists to find.

**No record carries a secret.** The fixture's plaintext password is `Fixture-Pw-C3` and its externalised
one is `aws:fa/reporting/pwd`; a test asserts neither string appears anywhere in a record, and another
asserts the record type has no field that could hold one. What travels is the *shape* — whether a
password exists, and its scheme prefix if it has one — which is all 349 needs to label it.

## Not yet exercised

Slices 1-3 assert packages, tasks, precedence, format versions, the partial-parse paths, the data flow
with its resource edges, and the connection records.

Nothing here is evidence about the client estate's real shape — the capture gave file counts only, not
package contents. Confirmation against the real packages belongs to Spec 361.

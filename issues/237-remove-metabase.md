# Issue #237: Remove metabase

Source issue: https://github.com/benitogonzalezh/five-panel/issues/237

## Problem

Five Panel no longer needs Metabase, but the repository still contains Metabase-specific infrastructure, configuration, app view-type support, docs, and restore workflow behavior. Those references make local and production environments keep provisioning or managing a Metabase service that is no longer part of the product.

## Current Behavior

Local Docker Compose defines an optional `metabase` service and exposes it through `METABASE_PORT`. Production Docker Compose defines a `metabase` service behind Traefik at `metabase.${DOMAIN}`. Postgres initialization creates a separate `metabase_app_db` database inside the same Postgres server used by Five Panel. Full database restore tooling can stop and restart a Metabase container before and after restore.

The application also still recognizes a legacy `metabase-dashboard` view type in backend sidebar allowlists and frontend view typing, stores, navigation labels, management forms, locale labels, and dispatcher routing. The dedicated Metabase dashboard component is only an iframe wrapper, while dashboards have already been migrated to the new dashboard engine.

## Desired Behavior

Five Panel should no longer start, configure, document, expose, or special-case Metabase. New local and production environments should not create the Metabase service or the `metabase_app_db` database. Existing production environments should have an explicit operator cleanup step for dropping the old Metabase metadata database after Metabase is stopped and no longer in use.

The generic dashboard engine and its `bi-dashboard` view type should remain available. Existing app dashboard data should not be deleted or migrated as part of this issue because dashboards are already migrated to the new engine.

## Scope

- Remove the Metabase service from local and production Docker Compose files.
- Remove `METABASE_PORT` from committed environment templates and workspace setup references.
- Remove the Postgres init script that creates `metabase_app_db`.
- Remove Metabase stop/restart flags, environment variables, CLI argument wiring, and console output from full restore tooling.
- Remove `metabase-dashboard` from backend sidebar view-type allowlists and related backend tests.
- Remove `metabase-dashboard` from frontend view types, user-view filtering, sidebar icons and labels, view-management options and formatting, user-management icon handling, locale strings, and dispatcher routing.
- Delete the duplicate Metabase iframe component.
- Update project, setup, and operations documentation so Metabase is no longer listed as part of the stack.
- Document the production cleanup step for dropping the old `metabase_app_db` database after the Metabase service has been removed.

## Out Of Scope

- Removing or renaming `bi-dashboard`.
- Changing the new dashboard engine.
- Deleting or migrating Five Panel dashboard records.
- Creating a replacement BI provider integration.
- Changing backup or restore behavior unrelated to Metabase.
- Automatically dropping `metabase_app_db` from an application migration.

## Acceptance Criteria

- `docker compose config` for local development no longer includes a `metabase` service.
- Production Docker Compose no longer defines `fp_metabase` or exposes `metabase.${DOMAIN}` through Traefik.
- New Postgres containers no longer create `metabase_app_db` from repository init scripts.
- Full restore tooling no longer accepts or emits `--stop-metabase`, `--no-stop-metabase`, `--restart-metabase`, `--no-restart-metabase`, `FULL_RESTORE_STOP_METABASE`, or `FULL_RESTORE_RESTART_METABASE`.
- The backend sidebar picker allowlist and its tests no longer include `metabase-dashboard`.
- The frontend no longer exposes `Metabase Dashboard` as a view type option or display label.
- The frontend no longer imports, routes to, or ships `DashboardMetabase.vue`.
- The generic `bi-dashboard` type remains available for dashboard views.
- Documentation includes a production cleanup note that operators may drop the old Metabase metadata database with `DROP DATABASE IF EXISTS metabase_app_db;` only after confirming the Metabase service is stopped or removed and no active connections remain.
- A repository search for `metabase`, `Metabase`, and `metabase-dashboard` finds no active code, configuration, or setup references after the change, except for any intentionally retained historical notes that explicitly state Metabase has been removed.

## Implementation Notes

Treat this as a removal of the legacy Metabase integration, not a dashboard-system removal. `bi-dashboard` is the surviving dashboard view type and should continue to work anywhere dashboard views are supported.

Do not add a data migration for app dashboard records. The selected issue owner clarified that dashboards are already migrated to the new engine, so the implementation should remove stale `metabase-dashboard` code paths rather than converting production data.

The Metabase metadata database is separate from the Five Panel app database but lives in the same Postgres server. The cleanup command should target only `metabase_app_db`, not `${DB_NAME}` or any tenant/application tables. Because dropping a database is destructive and production-specific, document it as an explicit operator step instead of hiding it inside the app migration flow.

Before dropping `metabase_app_db` in production, operators should stop or remove the Metabase service and verify there are no active connections to that database. If active connections remain, terminate only those sessions intentionally before running the drop.

## Test Expectations

- Update backend service tests that assert sidebar picker candidate view types.
- Update or add focused frontend tests for view type options, labels, dispatcher behavior, or navigation helpers where existing coverage touches `metabase-dashboard`.
- Run the backend test suite or the focused backend tests covering `ViewService` and restore CLI behavior.
- Run the frontend test suite or focused frontend tests covering view management, user view filtering, and dispatcher/navigation behavior.
- Run a frontend build or typecheck to confirm removing `metabase-dashboard` from `ViewType` does not leave TypeScript references behind.
- Run `docker compose config --quiet` to validate local Compose after removing the service.
- Run `docker compose -f docker-compose.prod.yml config --quiet` with required production environment values to validate production Compose after removing the service.

## Risks

The main risk is leaving a stale Metabase reference in ops or documentation, causing production operators to continue managing a service that no longer exists. The database cleanup is destructive if pointed at the wrong database, so the spec requires it to remain an explicit production ops step targeting only `metabase_app_db`.

Another risk is accidentally removing generic dashboard support. The implementation should keep `bi-dashboard` intact and verify dashboard routes still work through the new engine.

## Open Questions

None

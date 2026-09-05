# Issue #244: New Report for ALCI

Source issue: https://github.com/benitogonzalezh/five-panel/issues/244

## Problem

ALCI has Listadura records for contractor work, but it does not have a native report that summarizes contractor costs and provides a daily breakdown. Users currently need to calculate this information outside Five Panel.

The PDF attached to the source issue shows the desired information and supplies a verified calculation example. It is a guide for the report structure and formulas, not a request for an SV-only report or an exact copy of the PDF design.

## Current Behavior

- Five Panel already has a native report engine, an Analytics section, built-in filters, tables, grouped tables, permissions, and PDF export.
- ALCI already has Listadura, Persona, Centro de Costo, and Campo model views containing the data needed by the report.
- ALCI has other native reports that establish the expected setup, filter, permission, menu, and test patterns.
- There is no generic contractor report for Listadura data.

## Desired Behavior

Add a native ALCI report named `Contratistas Reporte` under Analytics. The report must cover all ALCI Listadura records linked to a Persona whose type is `contratista`. It must not be limited to SV, Z, a fixed source file, or a fixed set of fields.

The report must provide the same general workflow as the other ALCI reports:

- dynamic filters for Año, Semana, Día, Campo, Centro de costo, Especie, Contratista, Labor, and Sublabor;
- a summary table grouped by Labor, Sublabor, and Contratista; and
- a daily breakdown grouped by date.

The summary table must show Labor, Sublabor, Contratista, Jornadas, Monto Total, and $/Jornada. The daily breakdown must show Contratista, Labor, Sublabor, Centro de Costo, Personas, Horas, Jornadas, Monto, and $/Jornada. Each date group must summarize Personas, Jornadas, and Monto.

Calculations must use these formulas:

- `Jornadas = Personas * Horas / 8`
- `Monto = A facturar`
- `$/Jornada = SUM(Monto) / SUM(Jornadas)`

The grouped and overall $/Jornada value is a weighted rate calculated from the grouped totals. It must not be the average of row-level rates, and it must not use the Listadura `Jornal` field. Currency and $/Jornada values use whole Chilean pesos. Jornadas retain enough decimal precision to show partial days.

## Scope

- Create or update one native report with the exact visible name `Contratistas Reporte` for the ALCI tenant.
- Place the report under the existing Analytics section and follow the same visibility and menu setup as the other ALCI reports.
- Read from the current generated ALCI model views for Listadura, Persona, Centro de Costo, and Campo.
- Include all fields and cost centers represented by matching contractor records.
- Provide dynamic, multi-value filters for:
  - Año, based on ISO week-numbering years from Listadura dates;
  - Semana, based on ISO week numbers;
  - Día, using the report engine's standard date or date-range behavior;
  - Campo;
  - Centro de costo;
  - Especie;
  - Contratista;
  - Labor; and
  - Sublabor, sourced from Listadura `descripcionLabor`.
- Treat an empty filter as no restriction for that field, consistent with the existing report behavior.
- Apply every active filter consistently to the summary and daily datasets.
- Add a summary dataset and table with a final total row.
- Add a daily dataset and grouped table ordered by date, with daily summaries for Personas, Jornadas, and Monto.
- Use the native report formatting, responsive layout, loading and empty states, permissions, and PDF export.
- Add a new dated, idempotent ALCI setup script and focused automated tests.

## Out Of Scope

- Hard-coding the report to Santa Victoria, Pimiento, SV, Z, or any other field or import source.
- Creating separate SV and Z reports.
- Reproducing the attached PDF's exact branding, page breaks, header, or pagination.
- Changing the native report engine, report APIs, database schema, or ALCI model schema.
- Changing Listadura import configurations, reconciliation, cleanup, or source files.
- Correcting spelling, accents, capitalization, or other source-data labels.
- Repairing incomplete source rows or inventing missing amounts, people, hours, relations, or dates.

## Acceptance Criteria

1. An authorized ALCI user can open a native report named `Contratistas Reporte` from the existing Analytics section.
2. The report includes Listadura rows only when their related Persona has type `contratista`; it does not impose an SV, Z, Campo, Centro de costo, or import-source restriction unless the user selects a filter.
3. Año, Semana, Día, Campo, Centro de costo, Especie, Contratista, Labor, and Sublabor filters are available and obtain their options from current ALCI data where an option list applies.
4. Each filter supports the same selection and empty-value behavior as comparable existing ALCI report filters, and active filters affect both report tables consistently.
5. The summary table contains Labor, Sublabor, Contratista, Jornadas, Monto Total, and $/Jornada, with one row per Labor, Sublabor, and Contratista combination plus a final total row.
6. For every base row, Jornadas equals Personas multiplied by Horas and divided by 8, and Monto equals `aFacturar`.
7. For every summary group and for the final total, $/Jornada equals the summed Monto divided by the summed Jornadas, rounded for display to a whole Chilean peso. It is not calculated from the `jornal` field or from an average of row rates.
8. The daily breakdown is grouped and ordered by date and contains Contratista, Labor, Sublabor, Centro de Costo, Personas, Horas, Jornadas, Monto, and $/Jornada. Each date group shows the sum of Personas, Jornadas, and Monto for that date.
9. Missing numeric values do not cause a report error. Numeric totals treat missing inputs as zero, and a $/Jornada calculation with zero total Jornadas displays zero rather than raising a SQL error.
10. With Año `2026`, Semana `22`, and Campo values `Santa Victoria` and `Pimiento`, the restored reference data returns 22 unique detail rows and these exact summary results:

    | Labor | Jornadas | Monto Total | $/Jornada |
    | --- | ---: | ---: | ---: |
    | Cosecha | 38 | $1.444.000 | $38.000 |
    | Desbrote | 14 | $589.740 | $42.124 |
    | Amarre | 18 | $501.930 | $27.885 |
    | Análisis campo | 11 | $352.000 | $32.000 |
    | Aplicación | 10 | $320.000 | $32.000 |
    | **TOTAL** | **91** | **$3.207.670** | **$35.249** |

11. The same reference selection returns these exact daily summaries:

    | Fecha | Personas | Jornadas | Monto |
    | --- | ---: | ---: | ---: |
    | 25/05/2026 | 26 | 19 | $718.060 |
    | 26/05/2026 | 19 | 19 | $729.680 |
    | 27/05/2026 | 17 | 17 | $543.710 |
    | 28/05/2026 | 24 | 18 | $612.840 |
    | 29/05/2026 | 18 | 18 | $603.380 |

12. The built-in PDF export includes the active filter context, summary table, total row, and daily breakdown without requiring a custom export path.
13. Running the setup script more than once leaves one report and one intended Analytics menu entry per eligible user, without duplicate records.
14. The report remains tenant-scoped: users from another tenant cannot read the report or its data.

## Implementation Notes

- Follow the existing dated ALCI report-script pattern rather than adding a custom page or report-engine feature. Use a stable report slug such as `contratistas-reporte` and create or update the report by that slug.
- Use the current generated views (`alci_listaduras`, `alci_personas`, `alci_centros_costo`, and `alci_campos` in the verified backup) rather than legacy view names. Join related rows by both record ID and tenant ID.
- Build one shared filtered base query or equivalent common SQL shape so the summary and daily datasets cannot drift apart.
- Derive the contractor display value from the related Persona name fields and retain a stable Persona ID for filtering and grouping.
- Derive Campo and Especie through Listadura -> Centro de Costo -> Campo. Do not infer an SV or Z group from field names or import metadata.
- Use ISO year and ISO week calculations for Año and Semana. Follow the safe date-casting and date-filter pattern already used by current ALCI reports.
- Use current filter definitions, including the required value types for SQL-backed select filters, and declare every filter used by each dataset.
- Use `COALESCE` for nullable numeric inputs and sums. Guard rate divisions with `NULLIF` or an equivalent safe expression, and return zero when the denominator is zero.
- Calculate row-level $/Jornada from that row's Monto and Jornadas. Recalculate group and total rates from summed Monto and summed Jornadas.
- Sort the summary with the total row last. Sort the daily breakdown by date and then by contractor, labor, sublabor, and cost center for stable output.
- Use native CLP currency formatting and native table/grouped-table styles. Minor visual differences from the guide PDF are acceptable.
- Give the report the same ALCI public-read and Analytics menu treatment as the existing native ALCI reports while keeping report editing limited to the roles already allowed by the report system.

## Test Expectations

- Unit-test the built report definition, including its name, slug, complete filter list, filter value types, datasets, columns, formats, layout, and filter-to-dataset wiring.
- Test that the SQL applies the contractor-type condition but contains no hard-coded SV, Z, field value, cost-center value, or import-source restriction.
- Test each filter alone and in combination, including empty filters and the standard Año/Semana/Día interaction.
- Test Jornadas with full and partial days, including the known `7 personas * 4 horas / 8 = 3.5 jornadas` example.
- Test row, group, daily, and overall calculations. Include data where averaging row rates would produce a different answer to prove that the report uses `SUM(Monto) / SUM(Jornadas)`.
- Test that `jornal` can differ from the calculated rate without affecting report results.
- Test null numeric inputs and zero Jornadas to prove the datasets do not raise division or cast errors.
- Test the reference PDF values in Acceptance Criteria 10 and 11 against a focused fixture or equivalent dataset-level integration test.
- Test setup-script idempotency, ALCI tenant lookup, Analytics placement, report visibility, user-menu creation, and tenant isolation.
- Run the report-definition validator and the relevant backend build and tests.
- Perform a local report smoke test using the restored backup and confirm the summary, daily breakdown, filters, empty state, responsive layout, and built-in PDF export.

## Risks

- Listadura dates are stored as text. Blank or malformed source dates can break unsafe casts or prevent records from matching date filters.
- Some Listadura rows have missing amounts, people, hours, Persona relations, cost-center relations, or field relations. The report must remain usable without inventing source data.
- Campo, Centro de costo, Labor, Sublabor, and Persona labels come from source data and may contain spelling, accent, or capitalization differences.
- An unfiltered report can return many rows. The implementation should reuse the current report patterns and confirm acceptable performance with the restored ALCI data before adding new indexes or report-engine behavior.
- Future source files may add fields, cost centers, contractors, labor values, or sublabor values. Dynamic filters and the lack of hard-coded field groups are required so those records appear automatically.

## Open Questions

None.

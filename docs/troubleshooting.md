# Refresh and compliance troubleshooting

The repository contains an imported `.pbix` model created with the June 2022 Power BI release. Its settings enable relationship import. The instructions below are a repair/diagnostic procedure, not a claim that the binary has been refreshed against a current tenant. Work on a copy and keep the original.

## Duplicate key errors (#1)

A repeated value on the **many** side is expected. A column on the **one** side must be unique. In particular, `enrollmentActivities.dateKey` identifies a day, not a unique activity. Many enrollment classifications can share a date.

In Model view, inspect each reported relationship and compare its keys with the current warehouse schema. A valid example is:

| Dimension (one) | Fact (many) |
|---|---|
| `dates.dateKey` | `enrollmentActivities.dateKey` |
| `enrollmentFailureReasons.enrollmentFailureReasonKey` | `enrollmentActivities.enrollmentFailureReasonKey` |
| `enrollmentEventStatuses.enrollmentEventStatusKey` | `enrollmentActivities.enrollmentEventStatusKey` |
| `deviceTypes.deviceTypeKey` | `enrollmentActivities.deviceTypeKey` |
| `deviceEnrollmentTypes.deviceEnrollmentTypeKey` | `enrollmentActivities.deviceEnrollmentTypeKey` |
| `enrollmentFailureCategories.enrollmentFailureCategoryKey` | `enrollmentActivities.enrollmentFailureCategoryKey` |

Use single-direction filtering from dimension to fact unless the report has a specific, validated requirement otherwise. Check uniqueness of each dimension key across the full data, not just the Power Query preview. Confirm both columns have compatible types. Disable automatic relationship detection while correcting unwanted inferred relationships.

For `mamApplicationInstances.platform` and `mamCheckins.deviceHealthKey`, do not assume either reported column is a unique key. Inspect the current `$metadata` and the corresponding MAM dimension to identify the actual surrogate key. If a matching relationship cannot be established, leave it disconnected until the schema is confirmed. Do not deduplicate fact rows or delete entire fact tables merely to suppress a refresh error; that can discard valid data. A many-to-many relationship is also not a blanket fix.

Acceptance: full refresh succeeds, affected table row counts remain intact, and date/platform/health slicers produce expected counts on a small known sample.

## Generic OLE DB / ODBC error `0x80040E4E` (#3)

This code alone does not identify the failing query or prove a locale problem. First expand the refresh error and record the first failing table and underlying message. In Transform data, refresh that query's preview and inspect its applied steps. A model relationship failure can occur when loading even if the preview succeeds.

- If the detail says duplicate values on a one-side key, follow the relationship procedure above.
- If it names a conversion step, inspect the original value and data type. Apply **Change type > Using locale** with the locale of the source text only to the affected conversion. Switching all OS settings to German is not a general fix.
- If it reports authentication/authorization or an OData endpoint error, verify the configured warehouse URL, credentials and access separately.

For further diagnosis, share Power BI Desktop version, failing table, full nested error and the relevant applied steps with tenant URLs, UPNs, tokens and device details removed. A small synthetic failing value is better than a complete tenant export.

## Why is a device noncompliant? (#2)

The device's summary compliance flag cannot explain which setting failed. A detailed view needs device + policy + setting records. Microsoft Graph exposes `deviceCompliancePolicyStates` for a managed device and `settingStates` for each returned policy state. Retain the device ID, policy ID/name, setting name, `state`, `errorCode` and `errorDescription`, including unknown/error/not-applicable states rather than treating all as failures.

This requires a separate authenticated Graph data source (and pagination), appropriate Intune read permissions/RBAC, and a refresh strategy supported by Power BI Service. Never embed access tokens or client secrets in the PBIX. The provided binary does not gain this capability just by adding a calculated column. Warehouse and live Graph snapshots can also differ in freshness; show their refresh timestamps.

## Sources

- [Power BI relationship cardinality and filtering](https://learn.microsoft.com/en-us/power-bi/transform-model/desktop-relationships-understand)
- [Intune warehouse device and enrollment entities](https://learn.microsoft.com/en-us/intune/developer/data-warehouse/ref-devices)
- [Microsoft example of per-device policy and setting queries](https://learn.microsoft.com/en-us/answers/questions/1222234/intune-compliance-status-api-integration-with-clea)
- [Graph setting-state fields](https://learn.microsoft.com/en-us/graph/api/resources/intune-deviceconfig-devicecompliancepolicysettingstate?view=graph-rest-1.0)

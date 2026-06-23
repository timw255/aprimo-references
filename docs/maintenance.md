# Maintenance references (`ref:maintenanceJob`, `ref:maintenanceTarget`, and friends)

The maintenance framework in Aprimo DAM models a **job** that runs one or more **actions** against a set of **targets**. Each part of that model has its own reference type. These references are how you build the body of a maintenance-job notification e-mail or customize a maintenance/order report.

The reference types covered here:

| Reference | Role |
|---|---|
| [`ref:maintenanceJob`](#maintenancejob-refmaintenancejob) | The job itself — counts, status, timing, and collections of targets/actions/errors. |
| [`ref:maintenanceAction`](#maintenance-action-refmaintenanceaction) | A single action within a job. |
| [`ref:maintenanceTarget`](#maintenance-target-refmaintenancetarget) | A single target a job ran against. **Container** — reads child refs per target. |
| [`ref:orderTargetAction`](#order-target-action-refordertargetaction) | An action performed on one ordered file. |
| [`ref:orderDeliveredFile`](#order-delivered-file-reforderdeliveredfile) | A file delivered by an ordering job. |
| [`ref:maintenanceNotificationError`](#maintenance-notification-error-refmaintenancenotificationerror) | An error raised by a notification agent after a job ran. |

> **Context:** Maintenance references only resolve **inside maintenance-specific settings** — for example a maintenance-job e-mail template or a maintenance/order report. Outside that context there is no job to read, so they have nothing to return.

> **How the collections fit together:** `ref:maintenanceJob` returns collections (`targets`, `failedTargets`, `actions`, `deliveredFiles`, `notificationErrors`, …). You iterate those collections and, inside the loop, use the matching item reference (`ref:maintenanceTarget`, `ref:maintenanceAction`, `ref:orderDeliveredFile`, `ref:maintenanceNotificationError`) to read each item. The "container" tags below parse and run their child references once per item.

---

## MaintenanceJob (`ref:maintenanceJob`)

Reads a property of the maintenance job, or returns one of its collections.

### Syntax

```xml
<ref:maintenanceJob out="out" />
```

To iterate a collection, nest the matching item reference inside:

```xml
<ref:maintenanceJob out="actions">
  <ref:maintenanceAction out="label" />
</ref:maintenanceJob>
```

### Attributes

| Attribute | Description |
|---|---|
| `out` | Which job property/collection to return (tables below). |
| `maxTargets` | When `out` returns a target collection (`failedTargets`, `succeededTargets`, …), limit it to the first *n* targets, e.g. `maxTargets="100"`. |

### `out` values

| `out` value | Type | Description |
|---|---|---|
| `actions` | iEnumerable `<maintenanceAction>` | The maintenance actions in this job. |
| `actionCount` | Int | Number of actions in this job. |
| `attempts` | Int | How many times the Maintenance Manager has already tried to execute this job. |
| `creatorEmail` | String | E-mail address for this job (by default, the creator's e-mail). |
| `creatorName` | String | Name of the user that created this job. |
| `descriptiveLabel` | String | Descriptive label of this job, in the current user's UI language. |
| `earliestStartDateLocal` | DateTime | Earliest local time this job can run. |
| `earliestStartDateUtc` | DateTime | Earliest UTC time this job can run. |
| `executionTime` | Timespan | How long this job took to execute. |
| `failedCount` | Int | Number of failed targets. |
| `failedTargets` | iEnumerable `<maintenanceTarget>` | All failed targets. Limit with `maxTargets`. |
| `failedTargetsHiddenCount` | Int | Failed targets hidden because of a `maxTargets` limit. |
| `groupId` | Guid | Group ID for this job. |
| `groupIndex` | Int | Index of this job within its group. |
| `groupSize` | Int | Total number of jobs in its group. |
| `label` | String | Label of this job, in the current user's UI language. |
| `maximumNumberOfRetries` | Int | How many times the agent retries failed targets. |
| `maximumNumberOfAttempts` | Int | `maximumNumberOfRetries` + 1. |
| `message` | String | Error message describing why the job failed; null if not yet run or successful. |
| `minimumRetryWaitTime` | Timespan | Minimum wait before a second attempt of a failed job. |
| `newFailedTargets` | iEnumerable `<maintenanceTarget>` | Targets failing for the first time (excludes retries). |
| `newFailedTargetCount` | Int | Number of newly failed targets. |
| `notificationErrors` | iEnumerable `<maintenanceNotificationError>` | Notification-agent errors. |
| `notificationErrorCount` | Int | Number of notification errors. |
| `pendingCount` | Int | Number of targets still pending. |
| `priority` | Int | Job priority: `0` = High, `1` = Medium. |
| `retriedTargetCount` | Int | Failed targets for which a new job will be / has been created. |
| `retryOfId` | Guid | ID of the original job if this is a retry; null otherwise. |
| `retryId` | Guid | ID of the retry job created for failed targets; null if none. |
| `startedOnLocal` | DateTime | When execution started, local time. |
| `startedOnUtc` | DateTime | When execution started, UTC. |
| `status` | String | Job status: `Pending`, `Success`, `Failed`, `PartiallyFailed`, `Executing`, or `Canceled`. |
| `succeededCount` | Int | Number of successfully updated targets. |
| `succeededTargets` | iEnumerable `<maintenanceTarget>` | All succeeded targets. Limit with `maxTargets`. |
| `succeededTargetsHiddenCount` | Int | Succeeded targets hidden because of a `maxTargets` limit. |
| `targets` | iEnumerable `<maintenanceTarget>` | All targets. |
| `targetCount` | Int | Number of targets. |

### Ordering-job `out` values

Jobs created by an **ordering** process expose extra outputs:

| `out` value | Type | Description |
|---|---|---|
| `deliveredFiles` | iEnumerable `<orderDeliveredFile>` | The files delivered in this order. |
| `deliveredFilesCount` | Int | Number of delivered files. |
| `failOnError` | Boolean | `False` = log and continue on error; `True` = stop and throw. |
| `isWatermarkingEnabled` | Boolean | Whether watermarking was enabled for this order. |
| `maximumOrderNameAttempts` | Int | Max attempts to compute a non-conflicting file name. |
| `singleFileZipMode` | String | If/when single files must be zipped. |
| `multipleFileZipMode` | String | If/when multiple files must be zipped. |
| `uncompressedSize` | Long | Sum of delivered file sizes, in bytes. |

### Examples

```xml
<ref:maintenanceJob out="status" />
```
Returns the job status, e.g. `PartiallyFailed`.

```xml
<ref:maintenanceJob out="deliveredFilesCount" />
```
Returns the number of files delivered by an order, e.g. `12`.

Iterate the job's actions and print each label (lint-verified):

```xml
<ref:maintenanceJob out="actions">
  <ref:maintenanceAction out="label" />
</ref:maintenanceJob>
```

Limit a target collection and report how many were hidden:

```xml
<ref:maintenanceJob out="succeededTargetsHiddenCount" maxTargets="100" store="@SucceededHidden" />
<ref:text onVariable="IsNotZero(@SucceededHidden)">
  <tr>
    <td>
      <ref:text out="@SucceededHidden" /> succeeded target(s) not displayed.
    </td>
  </tr>
</ref:text>
```
Shows a "not displayed" line only when the count was actually capped by `maxTargets`.

Walk the failed targets (lint-verified):

```xml
<ref:maintenanceJob out="failedTargets" maxTargets="100">
  <ref:maintenanceTarget out="key" />
</ref:maintenanceJob>
```

---

## Maintenance Action (`ref:maintenanceAction`)

Reads a single action within a job. Use it nested inside `ref:maintenanceJob out="actions"`.

### Syntax

```xml
<ref:maintenanceAction out="out" />
```

### `out` values

| `out` value | Type | Description |
|---|---|---|
| `label` | String | The action's label, in the current user's UI language. |
| `descriptiveLabel` | String | The action's descriptive label, in the current user's UI language. |

### Example

```xml
<ref:maintenanceAction out="label" />
```
Could return `Link record to classification` for an English UI user.

---

## Maintenance Target (`ref:maintenanceTarget`)

Reads a single target the job ran against. **This is a container** — references nested inside it (for example over a target's `actions`) are parsed and executed per item. Use it nested inside a job's target collection (`targets`, `failedTargets`, `succeededTargets`, …).

### Syntax

```xml
<ref:maintenanceTarget out="out" />
```

Container form (iterate the target's order actions):

```xml
<ref:maintenanceTarget out="actions">
  <ref:orderTargetAction out="descriptiveLabel" />
</ref:maintenanceTarget>
```

### `out` values

| `out` value | Type | Description |
|---|---|---|
| `attempts` | Int | How many times the manager has tried to execute this target. |
| `errorDetails` | String | The exception thrown, if any. |
| `executionTime` | Timespan | Time needed to execute this target. |
| `key` | String | A string representing the current object. |
| `message` | String | The (error) message set during execution of this target. |
| `status` | String | Target status: `pending`, `succeeded`, or `failed`. |
| `tag` | String | Value of the target's Tag property. |
| `xml` | String | The target's XML stream, as in the "Raw" maintenance report. |

### Order-target `out` values

Additional outputs when the target is an **ordered file**:

| `out` value | Type | Description |
|---|---|---|
| `fileName` | String | Resulting file name for this ordered file. |
| `fileSize` | Long | File size of this target file. |
| `recordId` | Guid | ID of the record the ordered file belongs to. |
| `itemId` | Guid | ID of the item being ordered. |
| `actions` | iEnumerable `<orderTargetAction>` | The order-target actions (iterate with `ref:orderTargetAction`). |
| `deliveredPath` | String | Full path where the file was delivered. Empty for e-mail orders. |

### E-mail-order-target `out` values

Additional outputs when the target is an **e-mail order**:

| `out` value | Type | Description |
|---|---|---|
| `recipients` | String | E-mail addresses included as recipients. |
| `subject` | String | Subject line of the e-mail. |
| `body` | String | Personal message entered in the e-mail order. |
| `attachmentCount` | Int | Number of attachments. `0` when files were delivered by FTP. |
| `deliveredFileCount` | Int | Total files delivered, via e-mail or FTP. |

### Examples

```xml
<ref:maintenanceTarget out="key" />
```
Could return `Images\test.jpg` for an indexed target file.

List the actions performed on an ordered file (lint-verified):

```xml
<ref:maintenanceTarget out="actions">
  <ref:orderTargetAction out="descriptiveLabel" />
</ref:maintenanceTarget>
```

---

## Order Target Action (`ref:orderTargetAction`)

Reads a single action performed on one ordered file. Use it nested inside `ref:maintenanceTarget out="actions"`.

### Syntax

```xml
<ref:orderTargetAction out="out" />
```

### `out` values

| `out` value | Type | Description |
|---|---|---|
| `descriptiveLabel` | String | The order-target action's descriptive label. |
| `xml` | String | The XML of an order-target action in an order-job report. |

> Useful when modifying the order-job report in Maintenance History.

---

## Order Delivered File (`ref:orderDeliveredFile`)

Reads a file delivered by an ordering job. Use it nested inside `ref:maintenanceJob out="deliveredFiles"` (an ordering job).

### Syntax

```xml
<ref:orderDeliveredFile out="path" />
```

### Example

```xml
<ref:maintenanceJob out="deliveredFiles">
  <ref:orderDeliveredFile out="path" />
</ref:maintenanceJob>
```
Iterates the delivered files and prints each delivery path.

---

## Maintenance Notification Error (`ref:maintenanceNotificationError`)

A notification error is an object holding the error message raised while a NotificationAgent ran *after* a maintenance job executed. Use it nested inside `ref:maintenanceJob out="notificationErrors"`.

### Syntax

```xml
<ref:maintenanceNotificationError out="out" />
```

### `out` values

| `out` value | Type | Description |
|---|---|---|
| `agentName` | String | Name of the NotificationAgent that raised the error. |
| `message` | String | The error message that occurred while running this agent. |

> Useful when modifying the maintenance-job report in Maintenance History.

---

## Gotchas

- **All maintenance references are context-bound** to a maintenance/order job. They return nothing in an arbitrary reference context.
- **Read collections with the matching item reference.** A `ref:maintenanceJob out="targets"` collection is iterated with `ref:maintenanceTarget` inside it; `actions` with `ref:maintenanceAction`; `deliveredFiles` with `ref:orderDeliveredFile`; `notificationErrors` with `ref:maintenanceNotificationError`. `ref:maintenanceTarget` and `ref:maintenanceJob` are containers that run their child refs per item.
- **`maxTargets` truncates — pair it with the hidden-count outputs.** When you cap `failedTargets`/`succeededTargets`, use `failedTargetsHiddenCount` / `succeededTargetsHiddenCount` to tell the reader how many you left out.
- **Order/e-mail-target outputs only exist for the matching target type.** `deliveredPath`, `recipients`, etc. are meaningful only on order targets / e-mail-order targets respectively.

See also: [overview.md](overview.md) for shared attributes and gating, and [foreach.md](foreach.md) for the general collection-iteration pattern.

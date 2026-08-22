# OALS Digital Guidance Portal

Static Arabic RTL student-services portal hosted with GitHub Pages.

## Current services

- Comprehensive trainee summary: existing Google Sheets validation, GPS option, Microsoft Flow endpoint, polling, and download counter are retained. Its production maintenance switch remains off.
- Print trainee schedule: uses the same sign-in validation, then calls a separate Microsoft Power Automate flow and displays the returned public PDF link.

## Connect the schedule flow

Create a OneDrive for Business folder, for example `/TraineeSchedules`, and name every PDF with the exact nine-digit trainee ID:

```text
/TraineeSchedules/412345678.pdf
```

Create an Instant cloud flow in Power Automate with the `When an HTTP request is received` trigger. Use POST and this request schema:

```json
{
  "type": "object",
  "properties": {
    "studentId": {
      "type": "string"
    }
  },
  "required": ["studentId"]
}
```

Add a **Compose** action named `File_path` with this expression (change the folder name if needed):

```text
concat('/TraineeSchedules/', string(triggerBody()?['studentId']), '.pdf')
```

Add the current OneDrive for Business **Create share link by path** action:

- File Path: output of `File_path`
- Link type: `View`
- Link scope: `Anonymous` / `Anyone`

Add a **Response** action:

- Status code: `200`
- Header: `Content-Type` = `application/json`
- Header: `Access-Control-Allow-Origin` = `*`
- Body: insert the action's **Web URL** dynamic value in this exact property:

```json
{
  "downloadUrl": "PASTE_THE_WEB_URL_DYNAMIC_VALUE_HERE"
}
```

Save the flow, copy the generated HTTP POST URL, then replace this placeholder in `index.html`:

```js
const SCHEDULE_FLOW_ENDPOINT = "PASTE_NEW_SCHEDULE_FLOW_URL_HERE";
```

The service card activates automatically when the value begins with `https://`.

## Operational flags

```js
const ENABLE_SUMMARY_SERVICE = false;
const ENABLE_SCHEDULE_SERVICE = true;
const ENABLE_GPS_VERIFY = false;
```

Set `ENABLE_SUMMARY_SERVICE` to `true` only when the existing summary service should be taken out of maintenance.

## Expected schedule-flow contract

Request:

```json
{ "studentId": "412345678" }
```

Successful response:

```json
{ "downloadUrl": "https://public-onedrive-link.example" }
```

The page also accepts `reportUrl`, `url`, or `webUrl` for compatibility, but `downloadUrl` is preferred.

## Privacy note

An anonymous OneDrive link can be opened by anyone who receives it. The HTTP trigger URL is also present in public GitHub Pages source. For student records, review tenant sharing policy and add server-side authorization before treating the browser-only Google Sheets check as a security boundary.

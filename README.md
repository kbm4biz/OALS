# OALS Digital Guidance Portal

Static Arabic RTL student-services portal hosted with GitHub Pages.

## Current services

- Comprehensive trainee summary: existing Google Sheets validation, GPS option, Microsoft Flow endpoint, polling, and shared download counter are retained and the service is enabled.
- Print trainee schedule: uses the same sign-in validation, then calls a separate Microsoft Power Automate flow, displays the returned public PDF link, and increments the same download counter.

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

Add OneDrive for Business **Get file metadata using path** before creating the link. Set its File Path to the output of `File_path`. This gives a clear failure when the PDF does not exist.

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

Open the Response action's **Settings** and make sure **Asynchronous response is Off**. This short lookup flow should return `200` directly; asynchronous mode adds a `202` status endpoint and unnecessary polling.

Add a second Response action for errors and use **Configure run after** so it runs when `Get file metadata using path` or `Create share link by path` fails or times out:

- Status code: `404`
- Header: `Content-Type` = `application/json`
- Header: `Access-Control-Allow-Origin` = `*`
- Body:

```json
{
  "ok": false,
  "message": "لم يتم العثور على جدول المتدرب أو تعذر إنشاء رابط المشاركة."
}
```

For both OneDrive actions, open **Settings > Retry Policy**, choose exponential retry, and use four retries for transient OneDrive or gateway failures.

Save the flow, copy the generated HTTP POST URL, then replace this placeholder in `index.html`:

```js
const SCHEDULE_FLOW_ENDPOINT = "PASTE_NEW_SCHEDULE_FLOW_URL_HERE";
```

The service card activates automatically when the value begins with `https://`.

## Operational flags

```js
const ENABLE_SUMMARY_SERVICE = true;
const ENABLE_SCHEDULE_SERVICE = true;
const ENABLE_GPS_VERIFY = false;
```

Set either service flag to `false` only when that service needs to be placed in maintenance mode.

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

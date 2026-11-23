# Download the agent — UI integration

For the team building the **Download Agent** button in the portal.

Clicking Download produces **two files** in the user's Downloads folder:

```
DlpInstaller.msi      the installer — identical for every customer
DlpInstaller.json     that customer's configuration
```

The user double-clicks the MSI. It reads the JSON sitting beside it and
configures itself. No command line, nothing to type.

---

## The two calls

**1 · Get the installer URL**

```
GET /qadlp/V1/DownloadAgent
ClientGUID: <tenant guid>
```

```json
{
  "success": true,
  "data": {
    "installerUrl": "https://qadlpinstaller.blob.core.windows.net/installer/...?sv=...&sig=...",
    "version": "1.2.32"
  }
}
```

Send the browser to `installerUrl`. It downloads **straight from storage**, not
through the API — that is what makes it fast.

**2 · Get the configuration file**

```
GET /qadlp/DownloadAgentConfig
ClientGUID: <tenant guid>
```

Responds with a file, not JSON to read:

```
Content-Disposition: attachment; filename="DlpInstaller.json"
```

Send the browser to it the same way. Do not parse it, do not display it — just
let it download.

---

## Working example

```js
async function downloadAgent() {
  const res  = await fetch('/qadlp/V1/DownloadAgent', {
    headers: { 'ClientGUID': clientGuid },
  });
  const json = await res.json();

  if (!json.success) {
    showError(json.message);
    return;
  }

  // 1 · the installer, straight from storage
  window.location = json.data.installerUrl;

  // 2 · the config file, a moment later so the browser does not
  //     treat the two as one burst and block the second
  setTimeout(() => {
    window.location = '/qadlp/DownloadAgentConfig?t=' + Date.now();
  }, 500);
}
```

The `?t=` is only to stop the browser reusing a cached response when somebody
downloads twice.

---

## Four things to tell the user

**Expect a browser prompt.** The first time, Chrome and Edge ask *"Allow
multiple downloads?"*. They must allow it, or only the MSI arrives.

**Keep both files together.** The installer looks for the JSON in its own
folder. Any folder works — Downloads, Desktop, a shared drive — as long as both
files are in it. Copying only the MSI to another machine will not install.

**Do not rename them.** A second download becomes `DlpInstaller (1).json`, which
still works, but a renamed or deleted JSON does not.

**The installer URL expires.** It is valid for a short window, so generate it
when the button is clicked rather than when the page loads.

---

## Errors

| | |
|---|---|
| `400` on either call | No `ClientGUID` was sent, or the environment is misconfigured. Show `message` — it names what is missing. |
| `404` | The route is not registered in the gateway. Tell us; it is our side. |
| Only one file downloads | The browser blocked the second. Prompt the user to allow multiple downloads and retry. |
| The JSON downloads but is empty | Do not ship it to a user — tell us. It means the configuration could not be built, and an agent installed from it would report to nobody. |

---

## What we need from you

Nothing to store, nothing to compute. The whole integration is: call one
endpoint for a URL, send the browser to two places.

The base URL differs per environment — ask us for yours; do not compile it in.

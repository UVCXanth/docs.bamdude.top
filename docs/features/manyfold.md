---
title: Manyfold library
description: Browse and search a self-hosted Manyfold model library inside BamDude and import its files into the File Manager, ready to slice and print
---

# Manyfold library

[Manyfold](https://manyfold.app/) is an open-source, self-hosted library for 3D models. BamDude connects to one Manyfold instance and shows its catalogue on the **Model sources** page: you browse and search the models, open one to see its files, and import the files you want into the [File Manager](file-manager.md). From there they are sliced, queued and printed like any other library file.

BamDude only reads from Manyfold. It never changes, uploads or deletes anything there.

---

## :material-cloud-download: How it works { #how-it-works }

```
Model sources → Manyfold tab ─→ search / page through the models ─→ open a model
        │
        └→ Import a file ─→ BamDude downloads it from Manyfold ─→ file in the library
```

| Step | What happens |
|------|--------------|
| 1. Connect | An administrator enters the Manyfold address and the client ID and secret of a Manyfold application under **Settings → Network → Manyfold connection**. |
| 2. Browse | The **Manyfold** tab of **Model sources** lists the models that application's owner can see in Manyfold. Search and paging are done by Manyfold. |
| 3. Open a model | Its preview, creator, licence, tags and files are shown, with a mark on every file the library already holds. |
| 4. Import | The BamDude server downloads the file from Manyfold, saves it to the library's **Manyfold** folder (or one you pick) and records which model it came from. |

Everybody with the Manyfold permission sees the same catalogue — the one the application's owner sees. There is one connection per installation, not one per user.

---

## :material-clipboard-check-outline: Requirements { #requirements }

- **Manyfold 0.150 or newer.** A Manyfold instance does not state its version, so BamDude reads the instance's own API description (`/api/v0/openapi.json`) and checks that it offers the model list and the raw file download. An instance without them is answered with *This Manyfold is too old. BamDude needs Manyfold 0.150 or newer.*
- **An application in Manyfold** with the scopes `public` and `read`. BamDude signs in with that application's client ID and secret.
- **An address the BamDude server can reach.** All requests come from the server BamDude runs on, so it is that machine — not your browser — that must reach Manyfold.

---

## :material-connection: Connecting { #connecting }

### In Manyfold { #connecting-manyfold }

This part is Manyfold's own interface.

1. Sign in to Manyfold as the user whose models BamDude should show.
2. Open the user menu → **API keys** and create an application.
3. Give it the scopes `public` and `read`.
4. Copy its client ID and client secret.

### In BamDude { #connecting-bamdude }

Open **Settings → Network**. The **Manyfold connection** card holds the whole connection:

| Field | What to enter |
|-------|---------------|
| **Use Manyfold** | The switch that turns the source on. It can be switched on only once the address, the client ID and the client secret are all there. |
| **Manyfold URL** | The address of the instance, starting with `http://` or `https://` — for example `http://192.168.1.10:3214`. A path is allowed when Manyfold lives under one; user names, passwords, `?` and `#` are not. |
| **Client ID** | The application's client ID. |
| **Client secret** | The application's client secret. It is write-only: the card shows whether one is stored, never its value. |

Then:

1. Press **Test connection**. It tries the values the form shows — saved or not — and saves nothing.
2. Switch **Use Manyfold** on.
3. Press **Save connection**. *Manyfold connection saved* confirms it.

Only an administrator can change or test the connection. Everybody else who can read the settings sees the card read-only, with the note *Only an administrator can change or test the Manyfold connection.*

### The client secret { #client-secret }

Once a secret is stored, the field offers three choices: **Keep current credential**, **Replace credential** and **Clear credential**.

The stored secret belongs to the address and client ID it was saved for. Change either of them and the card asks for the secret again — *Enter the client secret to continue.* — and the server refuses to save or test with the old one: *The address or client ID changed: enter the client secret again.* This is deliberate: a secret must never be sent to an address it was not entered for, and a changed client ID is a different application.

### What Test connection tells you { #test-connection }

The test reads the instance's API description, signs in, and reads the first page of models.

| Result | What it means |
|--------|---------------|
| *Connected. Models BamDude can see: N* | Everything works. N is what the application's owner can see. |
| *This address does not answer like Manyfold 0.150 or newer.* | Something answers at the address, but not with Manyfold's API description. Check the address and its port. |
| *This Manyfold is too old. BamDude needs Manyfold 0.150 or newer.* | It is Manyfold, but without the file download BamDude needs. Update Manyfold. |
| *Manyfold redirects to https. Enter its address starting with https://.* | The address was entered with `http://` and Manyfold sends it on to `https://`. BamDude does not follow that; enter the `https://` address. |
| *Manyfold did not accept the client ID or secret.* | The client ID or secret is wrong, or the application was removed. |
| *The Manyfold application needs the "read" scope. Create an application with the scopes "public" and "read".* | The application exists but lacks the `read` scope. |
| *Manyfold is limiting requests. Try again in a few minutes.* | Manyfold limits how often an application may sign in. Wait a few minutes. |
| *BamDude cannot reach Manyfold. Check the URL and that Manyfold is running.* | No answer: a wrong address, a closed port, a firewall, an untrusted certificate, or Manyfold is down. |
| *Manyfold answered in a form BamDude does not understand. Its version may be too old or too new, or a proxy in front of it changes its answers.* | See [compressed answers](#network-rules) below — a proxy that compresses is the usual cause. |

---

## :material-shield-lock-outline: Network and security rules { #network-rules }

!!! info "Where BamDude connects, and where it does not"
    - **The address may be on your own network.** `http://` and `https://` both work, also for a private or loopback address — a home server is the normal case.
    - **`https://` needs a certificate the BamDude server already trusts.** There is no field for a certificate of your own, and a self-signed one is not accepted. The server trusts the same certificate bundle as its other connections; `SSL_CERT_FILE` or `SSL_CERT_DIR` point it at another one.
    - **BamDude talks only to the configured address.** Addresses that appear inside Manyfold's answers are never fetched, and the sign-in is sent to that address alone. A proxy named by the server's environment variables is not used.
    - **Your browser never contacts Manyfold.** Previews and files are fetched by the BamDude server and handed to the browser by BamDude. The only link that leaves BamDude is **Open in Manyfold**, which you click yourself.
    - **Cloud-metadata, link-local and similar special addresses are never contacted**, whatever is entered or whatever a name resolves to.

!!! info "A library on S3 or other object storage"
    When Manyfold keeps its files in object storage, it answers a download with a redirect to that storage. BamDude follows it — to storage on the local network or on the internet, over `http://` or `https://` (a MinIO beside Manyfold, for instance) — and fetches the file there without the sign-in. Only addresses that are never file storage are refused: cloud-metadata and link-local addresses. The storage address has to be reachable from the BamDude server, not only from Manyfold.

!!! warning "A reverse proxy must not compress the answers"
    BamDude asks Manyfold for uncompressed answers and refuses a compressed one unread — the size limits are counted on the bytes that arrive. A reverse proxy in front of Manyfold that compresses regardless makes every request fail with:

    *Manyfold answered in a form BamDude does not understand. Its version may be too old or too new, or a proxy in front of it changes its answers.*

    Switch response compression off for BamDude's requests on that proxy.

---

## :material-view-grid-outline: Browsing { #browsing }

**Model sources** in the sidebar opens the page; it holds one tab per source — **MakerWorld** and **Manyfold**. A tab is shown to users who hold that source's permission, and the tab strip appears when you have more than one. The Manyfold tab is there once Manyfold is connected and switched on; before that it is shown only to users who can change the settings, with the note *Manyfold is not connected*.

| On the tab | What it does |
|------------|--------------|
| **Search models** + **Search** | Sends your words to Manyfold's own search and shows the first page of results. A query Manyfold cannot read is answered with *Manyfold did not understand this search. Try plain words.* |
| **Models: N** | How many models Manyfold reports for the current search. |
| Model tiles | The preview and the name of each model. Click a tile to open the model. A model without a preview shows a neutral tile. |
| **Previous** / **Next** | Paging. The page size is not BamDude's: it is the setting of the application's owner in Manyfold, and so is the order of the models. |

The address of the page holds the search words, the page number and the open model. A link to it can be shared, a reload returns to the same place, and the browser's **Back** returns from a model to the list it was opened from.

Previews come through BamDude: the server fetches the picture from Manyfold and serves it only when it really is an image.

---

## :material-cube-outline: A model { #a-model }

Opening a model shows its preview, name, caption, **Creator**, **Licence**, tags and an **Open in Manyfold** link to the model's page. **Back to models** returns to the list.

A model that Manyfold marks as sensitive carries a **Sensitive** badge, and its preview stays blurred — *Marked sensitive in Manyfold. Click to show.* — until you click it.

### Files { #files }

Under **Files** every file of the model is listed with its name and, once known, its size and the mark **with supports** when Manyfold says the file is pre-supported.

| File | What you can do |
|------|-----------------|
| `.3mf`, `.stl`, `.step`, `.stp`, `.obj` | Can be imported. |
| Any other type (pictures, documents, archives, other formats) | Listed, marked *not imported: BamDude takes 3MF, STL, STEP and OBJ files*. |
| A file Manyfold lists but does not let the application download | Listed, marked *Manyfold does not hand this file out*. |
| Already in the library | Marked **In library**, with **Show in library** opening its folder in the File Manager. |

**In library** means the library holds a file imported from exactly this file of this model, it is not in the trash, and you are allowed to see it.

Manyfold's file list carries no names or sizes, so BamDude asks for them file by file, twenty at a time. In a large model the rest are marked *details not loaded yet*, and **Load more file details** fetches the next batch. A file can be imported before its details are loaded.

---

## :material-import: Importing { #importing }

Imports need the Manyfold import permission and the right to upload to the library; without them the model is shown without import controls.

| You want | How |
|----------|-----|
| One file | **Import** on its row. |
| Some files of a model | Tick them (or **Select all**), then **Import selected (N)**. |
| Every file of a model | **Import all** — every importable file the library does not hold yet. |
| The files of several models | In the list, tick the models (or **Select page**), then **Import their files**. The ticks stay as you page through the catalogue; **Clear selection** drops them. Every printable file of those models that the library does not hold is imported. |

**Import to** chooses the destination. Left on **Manyfold folder**, files go to the top-level `Manyfold` folder of the library, which is created on the first import. A read-only external folder cannot be a destination.

### How a run goes { #import-run }

- Files are imported **one after another**. A file that fails does not stop the rest.
- **One run at a time.** While one is in flight the import controls are off: *An import is running. Wait for it to finish, or cancel it.*
- **The run keeps going when you leave the tab or the page.** It belongs to the browser tab BamDude is open in, not to the Manyfold tab: a toast — *Manyfold import: 2/5* — follows it across BamDude, and you can use everything else meanwhile.
- **Cancel import** stops before the next file. The file being imported at that moment is finished first: *Stopping after the file being imported…*
- **Reloading or closing the page ends the run.** Nothing is resumed. After a reload BamDude says how far it got: *The page was reloaded during an import: 3 of 5 files had been handled. The rest were not tried, and the file being imported at that moment may or may not have arrived — the "In library" marks show what your library holds.*
- A refusal no later file could get past — Manyfold switched off or disconnected, credentials refused, the connection changed, no space left on the server — ends the run at once and says how many files were not tried.

When the run ends, a summary stays on the Manyfold tab until you close it — *Imported: 4 · already in library: 1 · failed: 0* — with one line per file that failed and its reason. The toast says the same wherever you are. A single-file **Import** answers with a toast only: *Imported name.stl*, or *Already in your library*.

### Limits { #import-limits }

- **1 GB per file.** A larger file is refused: *The file is larger than 1 GB.*
- **30 minutes per download.** A download that takes longer is given up.
- **Two downloads at once per installation**, counted across all users. A file that finds both taken waits — the tab shows *Queued — 2 imports are running* or *Waiting for a free import slot…* — and is tried again; if nothing frees up it fails with *Imports are busy right now. Try again in a minute.*
- **Free space is checked first.** The server needs room for the file plus a reserve before the download starts: *The server has no space left for this file.*

A download is written to the `model-imports` folder of BamDude's data directory and moved into the library when it is complete. Unfinished downloads are removed, and anything a stopped server left there is cleaned up at the next start. The folder is not part of a backup.

### What lands in the library { #import-result }

| | |
|---|---|
| **Name** | The file's own name in Manyfold. When the library cannot take that name, the file is saved as `manyfold-<file id>` with its extension. |
| **No second download** | A file imported before is not downloaded again — the import answers with the library file that holds it and counts it as *already in library*. A file whose exact content the library already holds is not added twice either. |
| **Thumbnail** | An STL or OBJ gets the library's rendered picture, as an uploaded one does. A STEP file gets none automatically — use **Generate Thumbnail** in its menu, see [STEP thumbnails](file-manager.md#step-thumbnails). |
| **Origin** | The file keeps the address of its model's page as its **Source**, and the model's name, creator and licence as they were at the import. They are shown in the [File details](file-manager.md#file-details) dialog; **Open in Manyfold** in the file's menu opens the model's page. |

---

## :material-shield-key: Permissions { #permissions }

| Permission | Grants |
|------------|--------|
| `manyfold:view` | Open the Manyfold tab, search and page through the catalogue, open models, see previews. |
| `manyfold:import` | Import files into the library. The import also needs `library:upload`, because it adds a file to the library. |

The default groups grant both to **Administrators** and **Operators**, and `manyfold:view` to **Viewers**. On an existing installation every group that holds `makerworld:view` receives `manyfold:view`, and every group that holds `makerworld:import` receives `manyfold:import`; Administrators receive both.

Connecting Manyfold is not one of these permissions: saving and testing the connection needs an **administrator** who holds `settings:update`. Reading the connection — the address, the client ID and whether a secret is stored — needs `settings:read`.

For [API keys](api-keys.md), `manyfold:view` belongs to the **Can read status** scope and `manyfold:import` to **Manage Library**.

---

## :material-alert-circle-outline: Messages and what to do { #troubleshooting }

| Message | What it means | What to do |
|---------|---------------|------------|
| *Manyfold is not connected.* | No address, client ID or secret is saved. | An administrator sets it up under **Settings → Network**. |
| *Manyfold is switched off in Settings.* | The connection is saved, but **Use Manyfold** is off. | Switch it on and save. |
| *Manyfold did not accept the client ID or secret.* | Manyfold refuses the sign-in. | Check the application in Manyfold; enter its ID and secret again. |
| *The Manyfold application needs the "read" scope. Create an application with the scopes "public" and "read".* | The application lacks a scope. | Create an application with both scopes and connect with it. |
| *Manyfold is limiting requests. Try again in a few minutes.* | Manyfold's own request limit. | Wait a few minutes. |
| *BamDude cannot reach Manyfold. Check the URL and that Manyfold is running.* | No answer in time, or no connection at all. | Check from the BamDude server: the address, the port, the firewall, the certificate. |
| *Manyfold redirects to https. Enter its address starting with https://.* | The saved address uses `http://`, Manyfold insists on `https://`. | Save the `https://` address. |
| *This Manyfold is too old. BamDude needs Manyfold 0.150 or newer.* | The instance lacks the file download BamDude uses. | Update Manyfold. |
| *This address does not answer like Manyfold 0.150 or newer.* | The address is not a Manyfold, or a proxy answers in its place. | Check the address. |
| *Manyfold answered in a form BamDude does not understand. Its version may be too old or too new, or a proxy in front of it changes its answers.* | An answer BamDude cannot read — most often a proxy that compresses it. | Switch compression off for BamDude on the proxy; see [Network and security rules](#network-rules). |
| *Manyfold would not show this. The application's owner may not have access to it.* | Manyfold sent a redirect instead of an answer, as it does for something its user may not see. | Check the access of the application's owner in Manyfold. |
| *Manyfold cannot find this model or file, or the application's owner cannot see it.* | The model or file is gone, or hidden from that user. | Reload the list; check it in Manyfold. |
| *The file storage Manyfold redirected to has an address BamDude does not connect to.* | Manyfold redirected the download to a cloud-metadata or link-local address, to an address with a user name in it, or redirected too many times. | Check where Manyfold's storage is configured to live; see [Network and security rules](#network-rules). |
| *BamDude cannot reach the file storage Manyfold redirected to. The storage address must be reachable from the BamDude server, not only from Manyfold.* | Manyfold's storage endpoint is an address only Manyfold can open — a Docker service name such as `http://minio:9000`, for instance. | Set the storage endpoint in Manyfold to an address the BamDude server can open too. |
| *The file storage behind Manyfold refused the download.* | The storage answered the download with an error. | Try again; check the storage behind Manyfold. |
| *Only 3MF, STL, STEP and OBJ files can be imported.* | The file is of another type. | — |
| *The file is larger than 1 GB.* | Over the size limit. | Download it in Manyfold and upload it to the File Manager by hand. |
| *Imports are busy right now. Try again in a minute.* | Both download slots of the installation stayed taken. | Try again when the other imports are done. |
| *The server has no space left for this file.* | The disk of BamDude's data directory is full. | Free space on the server. |
| *The Manyfold connection changed during the import. Try again.* | The connection was edited or switched off while a file was downloading; that file was not saved. | Start the import again. |
| *The safety check of the Manyfold connection failed. Manyfold is switched off until BamDude restarts.* | BamDude could not prove that its connections go only to the addresses it checked, so it uses none. The tab shows *Manyfold is unavailable*. | Restart BamDude. If it comes back, reinstall BamDude's Python packages from `requirements.txt`. |
| *BamDude did not answer. Check your connection and try again.* | The browser got no answer from BamDude itself. | Check the connection between the browser and BamDude. |

Manyfold's API is version `v0`: Manyfold may change it in a later release, and a change BamDude does not know shows up as one of the messages above.

---

## :material-minus-circle-outline: What is not there { #not-there }

- **No import history tab.** Unlike MakerWorld, the Manyfold tab has no history; what was imported is marked **In library** on the model and lives in the File Manager.
- **Nothing is written to Manyfold.** No uploads, edits or deletions, and prints are not reported back.
- **One connection per installation.** One Manyfold instance, one application; every user sees what that application's owner sees.
- **No pasted links.** A Manyfold address pasted into the MakerWorld tab is not recognised — Manyfold is browsed, not pasted.

---

## :material-file-code: Developer reference { #developer-reference }

| Endpoint | Method | Auth | Purpose |
|---|---|---|---|
| `/api/v1/model-sources/manyfold/status` | GET | `manyfold:view` | `configured`, `usable`, the instance's `base_url`, and `imports` — downloads running, waiting, and the limit. |
| `/api/v1/model-sources/manyfold/models` | GET | `manyfold:view` | One page of models as Manyfold cut it. Query: `q` (up to 200 characters), `page`. |
| `/api/v1/model-sources/manyfold/models/{model_id}` | GET | `manyfold:view` | A model with all its files and `already_imported` — the library files that hold them. `files_page` selects which batch of file details is read. |
| `/api/v1/model-sources/manyfold/models/{model_id}/preview` | GET | media token, `manyfold:view` | The model's preview image. The token rides in `?token=` because `<img src>` cannot send headers. |
| `/api/v1/model-sources/manyfold/import` | POST | `manyfold:import` + `library:upload` | Import one file: `model_id`, `file_id`, optional `folder_id`. Answers `library_file_id`, `filename`, `folder_id`, `was_existing`. |
| `/api/v1/model-sources/manyfold/connection` | GET | `settings:read` | The connection: `enabled`, `url`, `client_id`, `client_secret_configured`. Never the secret. |
| `/api/v1/model-sources/manyfold/connection` | PUT | administrator + `settings:update` | Save the connection. |
| `/api/v1/model-sources/manyfold/connection/test` | POST | administrator + `settings:update` | Try a draft connection and save nothing. |

A refusal carries an `error` code and a `message`; the status is `400`, `404`, `409`, `413`, `502`, `503` or `507`. A refusal by Manyfold is never answered as `401` or `403` — those speak about the BamDude session.

---

## :material-link-variant: Related { #related }

- [File Manager](file-manager.md) — where imported files land.
- [MakerWorld import](makerworld.md) — the other tab of **Model sources**.
- [Slicer API](slicer-api.md) — slice an imported model on the server.
- [API keys](api-keys.md) — scopes for automated access.

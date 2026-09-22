# EcoExplorer — Earth Engine geometry uploader

A Streamlit companion to the EcoExplorer Google Earth Engine (GEE) web app. Upload a KML, zipped shapefile, or GeoJSON, then return to EcoExplorer and refresh to use the uploaded geometry.

**This repository contains the uploader, not the EcoExplorer GEE analysis application.** Its Python entry point is [`app.py`](app.py). Hosting or updating this repository does not publish or update the GEE web app.

## Upload a geometry

1. Open the deployed uploader provided by your EcoExplorer administrator.
2. Select one supported file:
   - **Shapefile:** a `.zip` containing the `.shp` and its companion files, including `.shx`, `.dbf`, and preferably `.prj`. Put files at the archive's top level, not inside a nested folder. The uploader uses the first `.shp` found.
   - **GeoJSON:** a `.geojson` file, preferably a FeatureCollection with longitude/latitude coordinates in WGS84. A top-level `.geojson` inside a ZIP is also accepted when no shapefile is found.
   - **KML:** a `.kml` file. The uploader removes geometry Z coordinates before conversion.
3. Wait for the upload/export to finish.
4. Return to EcoExplorer and refresh the browser. Select the uploaded asset if the GEE app is configured to list the same asset folder.

Use lowercase file extensions and a descriptive, unique filename suitable for an Earth Engine asset name. The filename without its extension becomes the asset name. Existing assets are not automatically overwritten or renamed.

### Asset visibility

**The current implementation makes uploaded Earth Engine assets publicly readable.** It explicitly sets `all_users_can_read` to `True` after export. The destination folder's name, `private`, does not make these assets private. Upload only geometries appropriate for that visibility.

Uploads use the deployment's service account, not an Earth Engine account supplied by each visitor. The app itself has no user sign-in or per-user asset separation.

## Current configuration

| Setting | Current value / location |
| --- | --- |
| Entry point | `app.py` |
| Earth Engine computation project | `careful-ensign-420823` in `ee.Initialize(...)` |
| Asset destination | `projects/ee-landflux/assets/private` in `import_asset_to_gee(...)` |
| Credentials | Streamlit secrets: `service_account` and `json_data` |
| Export behavior | Earth Engine table export; polls every five seconds |
| Asset access after export | Public read access |

The computation project and asset destination are distinct settings. The service account needs Earth Engine access through the configured project and permission to create assets and change their ACLs in the destination folder. The GEE analysis app must list this destination folder for uploads to appear in its selector.

## Run locally

Prerequisites: Python, Git, an Earth Engine-enabled Google Cloud project, and an authorized service account with a JSON key. Geospatial packages may require a compatible GDAL/Fiona installation; use an environment that supports those packages.

```sh
git clone https://github.com/kevinptu/ecoexplorer_kateri.git
cd ecoexplorer_kateri
python -m venv .venv
```

Activate the environment:

```powershell
# Windows PowerShell
.\.venv\Scripts\Activate.ps1
```

```sh
# macOS / Linux
source .venv/bin/activate
```

Install dependencies and launch:

```sh
python -m pip install -r requirements.txt
python -m streamlit run app.py
```

Before launching, create `.streamlit/secrets.toml` with these two keys:

```toml
service_account = "YOUR_SERVICE_ACCOUNT_EMAIL"
json_data = '''PASTE_THE_COMPLETE_SERVICE_ACCOUNT_JSON_OBJECT_HERE'''
```

Replace the placeholder with the complete valid JSON key object, including its escaped private-key newlines. `json_data` must be a JSON **string**, because the application calls `json.loads()` on it. Never commit the actual secrets file or key. This repository's `.gitignore` excludes the local secrets file.

The dependency file pins `geemap` and `earthengine-api`; other packages are not fully pinned. `pycrs` is installed directly from GitHub, so installation requires Git and network access. No specific Python version or reproducible environment lock is supplied.

## Deploy

For Streamlit Community Cloud, connect this repository, choose the branch to deploy, and set the entry point to `app.py`. Add the same two secrets through the deployment's secrets settings. Confirm the Earth Engine project, destination folder, and public-access behavior before enabling uploads.

For other hosts, install `requirements.txt`, provide Streamlit secrets securely, and run `python -m streamlit run app.py` with the host's required port/network settings.

The uploader needs outbound access to Earth Engine and writable temporary storage. The current `.streamlit/config.toml` and `.devcontainer/devcontainer.json` are empty and do not provide a configured runtime.

## Current limitations and troubleshooting

- **Missing secrets or initialization failure:** credential loading and Earth Engine initialization happen at startup. Check both secret keys, service-account authorization, and project access.
- **Asset does not appear in EcoExplorer:** confirm the export succeeded, then check that EcoExplorer lists the same asset folder and refresh it.
- **Duplicate filename:** choose a new filename/asset name or manage the existing asset separately.
- **ZIP not recognized:** place one shapefile and its sidecars at the top level of the archive. Nested directories are not searched.
- **Long-running export:** the request polls until the task stops. There is no timeout, and the code does not explicitly inspect the task's final success/failure status before attempting to change the asset ACL. Check Earth Engine task status for details.
- **Temporary files and concurrent uploads:** files are written into the working directory and extracted locally without automatic cleanup or full per-session isolation. Filename collisions are possible.
- **Upload validation:** the app does not implement comprehensive geometry, archive, or filename validation. Review upload handling before exposing it to untrusted users.

Although a `geemap.Map` object is constructed, the current app does not render a map preview. This uploader does not run ecological models or generate EcoExplorer analysis charts.

## Development

Source behavior is defined in `app.py`; dependencies are listed in [`requirements.txt`](requirements.txt). No automated test suite or license file is currently included. Documentation changes do not alter the upload code or asset permissions.

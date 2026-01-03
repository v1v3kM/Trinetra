# Trinetra - Copilot Instructions

> Real-Time Crowd Monitoring System by **XElite**

## Project Overview
- **Stack**: Node/Express API ([server.js](../Crowd_monitoring/server.js)), Python OpenCV detection ([detection.py](../Crowd_monitoring/detection.py)), Python Flask ML predictor ([app_model.py](../Crowd_monitoring/app_model.py))
- **Frontend**: Static HTML/JS dashboard in [public/](../Crowd_monitoring/public)
- **Database**: MongoDB Atlas (connection via `MONGO_URI` env var)

## Run/Debug
- Install Node deps with `npm install` inside [Crowd_monitoring](Crowd_monitoring); Python deps from [requirements_minimal.txt](Crowd_monitoring/requirements_minimal.txt) for detection and [requirements.txt](Crowd_monitoring/requirements.txt) for the ML app.
- Typical local start: run `node server.js` (port 3000) and `python detection.py` (posts counts to the Node API). Run `python app_model.py` separately (Flask on 5000) before exercising prediction/trend calls from the UI.
- Procfile defines two processes (`node server.js & python detection.py` and `python app_model.py`); Vercel routes are set in [vercel.json](Crowd_monitoring/vercel.json) but expect both Node and Python entrypoints.
- Frontend is plain HTML/JS served from `/crowd` via [public/index.html](Crowd_monitoring/public/index.html) plus [public/app.js](Crowd_monitoring/public/app.js); no bundler.

## Architecture & Data Flow
- Detection loop (OpenCV + MobileNetSSD in [detection.py](Crowd_monitoring/detection.py)) counts persons from video sources listed in `CAMERAS`, inserts into MongoDB collection `home.blogs`, and POSTs JSON to `POST /update_data` on the Node server.
- Node server ([server.js](Crowd_monitoring/server.js)) reads/writes the same Mongo collection via both native MongoDB driver (`collection` variable) and Mongoose models (`Crowd`, `History`, `Shop`, `User`). Data returned by `GET /data` feeds the dashboard map/charts.
- ML service ([app_model.py](Crowd_monitoring/app_model.py)) trains a RandomForest on historical Mongo data at startup; exposes `/predict` and `/past_trend` used by the browser to estimate future counts and trend comparisons.
- Frontend map/charts poll `/data` every 5s; trend comparison hits Flask at `http://127.0.0.1:5000/past_trend`; prediction form hits `http://127.0.0.1:5000/predict`; historic lookup uses `/history` on the Node side.

## Endpoints (Node)
- `GET /crowd` serves the dashboard HTML; `GET /` serves admin login at [admin/admin.html](Crowd_monitoring/admin/admin.html).
- Data: `GET /data` (latest 50 docs + summary), `POST /update_data` (ingest from detection), `GET /api/top-shops`, `GET /api/historical-data`, `GET /api/heatmap` (all backed by Mongoose `Crowd`).
- History routes are defined twice in [server.js](Crowd_monitoring/server.js): first takes `lat,lng,minutes` against the native driver, later takes `shop,time` against Mongoose `History`; the latter overrides the former—be explicit which behavior you expect before editing.
- Auth/shop mgmt in `POST /signup`, `POST /signin`, `POST /addShop`, `GET /shops` (no hashing/validation beyond basic checks). Separate legacy auth server in [login.js](Crowd_monitoring/login.js) (port 3001) also writes plain-text passwords.

## Conventions / Pitfalls
- Mongo URIs are hard-coded in [server.js](Crowd_monitoring/server.js), [detection.py](Crowd_monitoring/detection.py), [app_model.py](Crowd_monitoring/app_model.py), and [login.js](Crowd_monitoring/login.js); `mongoose.connect('process.env.mongo_URL'...)` literally uses the string, not the env var. Prefer env vars and avoid committing secrets.
- Data model inconsistency: native driver documents use `{coordinates:{latitude,longitude}, count, timestamp: Date}`; Mongoose `Crowd` uses `timestamp` as `String`. Align schemas before aggregations.
- Frontend assumes fixed shop-name-to-coordinate map in [public/app.js](Crowd_monitoring/public/app.js) and duplicates some logic inside [public/index.html](Crowd_monitoring/public/index.html); update both when changing shop lists.
- Long-running detection threads rely on Caffe model files [MobileNetSSD_deploy.prototxt](Crowd_monitoring/MobileNetSSD_deploy.prototxt) and [MobileNetSSD_deploy.caffemodel](Crowd_monitoring/MobileNetSSD_deploy.caffemodel); keep paths stable.
- No automated tests; validate changes manually by exercising `/data`, `/update_data`, `/predict`, `/past_trend`, and map refreshes in the browser.

## If you change things
- Coordinate API/front-end changes across Node, detection producer, and Flask predictor so schemas stay aligned.
- Be cautious with `/history` route conflicts and the dual Mongo drivers; confirm which collection/model handles the data before refactoring.
- Treat bundled creds as placeholders; migrate to env vars rather than rotating secrets in code.

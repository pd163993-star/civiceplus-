# CivicPulse

CivicPulse is a local-first civic issue reporting MVP. Residents can submit a structured complaint, include an optional photo, and follow the complaint by its ID. The dashboard summarizes reports by status.

## Problem

Local issues can be difficult to report and even harder to follow. Residents need a clear way to describe a problem, give a useful location, and know whether a report was received.

## Solution

CivicPulse brings issue reporting, complaint tracking, and a community status dashboard into one small web app. This hackathon MVP stores complaint records in a JSON file and uploaded photos on the local server by default. DynamoDB and private S3 adapters are opt-in and inactive unless configured.

## Features

- Dashboard counts for Submitted, Under Review, In Progress, and Resolved reports
- Complaint form with title, description, category, location, and optional image
- JPEG, PNG, and WebP image validation, with a 5 MB limit
- Mock AI-assisted drafting that structures a rough description and suggests a category
- Unique complaint IDs and a tracking view with a status progress display
- Local JSON storage behind a replaceable storage class
- Local image storage by default, with optional private S3 storage and API-served photo URLs
- Responsive layouts, loading and empty states, and field validation

## Technology

- Frontend: React 19, Vite, and Lucide icons
- Backend: Node.js, Express 5, and Multer
- Local data: JSON file and server-side image files
- Tests: Node.js built-in test runner

## Run locally

Requirements: Node.js 20.19+ or 22.12+ and npm.

```powershell
npm install
Copy-Item .env.example .env
npm run dev
```

Open `http://localhost:5173`. The API listens on `http://localhost:3001`; Vite proxies `/api` and `/uploads` to it. Leave `VITE_API_BASE_URL` empty for same-origin proxy requests, or set it to `http://localhost:3001` to call the local API directly (allowed by the example CORS origin). The initial complaint store is empty. Submitted reports are written to `server/data/complaints.json`, and validated photos are stored in the ignored `server/uploads/` directory.

Verify the project with:

```powershell
npm test
npm run lint
npm run build
```

## Environment variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `API_PORT` | `3001` | Local Express API port. The Vite proxy reads the same root `.env` value. |
| `PORT` | unset locally | Set to App Runner's configured container listener port (`3001` with the included Dockerfile); takes precedence over `API_PORT`. |
| `CORS_ORIGIN` | local Vite origin | Single browser origin allowed by the API. Set explicitly to the Amplify URL in production. Requests without an `Origin` work normally but receive no CORS header. |
| `STORAGE_DRIVER` | `json` | Storage selection: `json` by default, or explicitly `dynamodb`. |
| `DYNAMODB_TABLE` | none | Required only for DynamoDB storage. The table must use a string partition key named `id` containing the complaint ID. |
| `IMAGE_STORAGE_DRIVER` | `local` | Image backend: `local` by default, or explicitly `s3`. |
| `S3_BUCKET` | none | Required only when `IMAGE_STORAGE_DRIVER=s3`; the bucket must already exist and block public access. |
| `AWS_REGION` | AWS SDK default chain | Optional for AWS storage; set it to the DynamoDB table and S3 bucket region. The AWS SDK uses its standard credential provider chain; do not put credentials in this file. |
| `VITE_API_BASE_URL` | empty | Frontend API origin embedded at Vite build time. Leave empty for local proxy behavior; set to the App Runner API URL when building the production frontend. |

There are no AWS keys or external AI credentials in this project. Drafting currently uses a deterministic mock; it does not call Amazon Bedrock.

## Project architecture

```text
client/                 React + Vite single-page application
  src/App.jsx            Dashboard, report form, and tracking views
  src/civicpulse.css     Responsive CivicPulse styles
server/
  src/app.js             Express API, validation, and upload handling
  src/storage/            JSON complaint storage adapter
  src/images/             Local and optional S3 image storage adapters
  src/services/           Mock complaint drafting service
  data/complaints.json    Local complaint records
  uploads/                Local validated images (git-ignored)
  test/                   API integration tests
```

The API depends on a complaint-storage contract (`list`, `findById`, and `create`), selected by `STORAGE_DRIVER`. `JsonComplaintStorage` remains the local default. `DynamoDBComplaintStorage` uses complaint `id` as the table partition key and supports paginated list scans; create operations are conditional to avoid overwriting an existing ID. The API also depends on an image-storage contract (`save`, `get`, and `delete`), selected by `IMAGE_STORAGE_DRIVER`. `LocalImageStorage` remains the default and preserves `/uploads/<file>` URLs. `S3ImageStorage` stores objects without public ACLs and returns a stable `/api/complaints/<id>/photo` URL; the API retrieves the object by its server-side key. The API validates MIME type, file signature, and the 5 MB maximum before either adapter is called. AWS adapters require pre-created resources and use the AWS SDK credential provider chain (for example, an App Runner instance role); no credentials are embedded in the app. The drafting service remains a mock and can later call Bedrock from the server.

## AWS deployment plan

No AWS services are currently used, and CivicPulse has not been deployed. The intended production request path is:

```text
User
  ↓
AWS Amplify Hosting (React/Vite)
  ↓ HTTPS API request, VITE_API_BASE_URL
AWS App Runner (Express API, CORS_ORIGIN allowlist)
  ├── Amazon DynamoDB (complaint records)
  └── Private Amazon S3 bucket (complaint images, API-proxied reads)
```

### Production configuration

Set API variables on App Runner, not in frontend code: configure the service listener and `PORT` to `3001`, set `CORS_ORIGIN` to the exact Amplify origin, `STORAGE_DRIVER=dynamodb`, `DYNAMODB_TABLE`, `IMAGE_STORAGE_DRIVER=s3`, `S3_BUCKET`, and `AWS_REGION`. Attach an App Runner instance role for AWS access rather than setting access keys. Set `VITE_API_BASE_URL` in Amplify's build environment to the App Runner HTTPS URL; Vite embeds this value into the public JavaScript bundle, so it must only contain the non-secret API origin. Do not use the placeholder values in `.env.example` as actual service URLs.

### Docker

Build from the repository root so the Dockerfile can access both workspace manifests:

```powershell
docker build -f server/Dockerfile -t civicpulse-api .
docker run --rm -p 3001:3001 -e PORT=3001 -e CORS_ORIGIN=http://localhost:5173 civicpulse-api
```

The container listens on `0.0.0.0`, respects `PORT`, and contains no `.env`, upload, test, or `node_modules` folders from the build context. Local file storage inside a container is ephemeral; production configuration must use DynamoDB and S3.

### Future AWS deployment steps

1. Create an ECR repository, a DynamoDB table with string partition key `id`, a private S3 bucket, and least-privilege App Runner ECR-access and instance roles.
2. Build and push the API container to ECR, then create the App Runner service from that image. Set its listener and `PORT` to `3001`, configure its health check path as `/api/health`, attach the instance role, and set production storage variables. Leave cross-origin access closed until the frontend origin is known.
3. Create the Amplify Hosting app/branch, build with `VITE_API_BASE_URL` set to the App Runner HTTPS URL, and deploy the static frontend.
4. Set App Runner `CORS_ORIGIN` to the exact Amplify origin and update/redeploy the API service.
5. Verify health, complaint create/list/track, private image upload/retrieval, and browser CORS from the deployed frontend.

The App Runner instance role will need DynamoDB `GetItem`, `PutItem`, and `Scan` on the specific complaint table, and S3 `PutObject`, `GetObject`, and `DeleteObject` on the specific bucket's `complaints/*` object prefix. No administrator permissions are required. Keep the bucket private: the API returns photos through `/api/complaints/<id>/photo` and never exposes S3 credentials or a public object URL. No AWS resource has been created and nothing is deployed.

Before a public launch, add authentication or abuse controls, authorization for status updates, retention rules, request-rate limits, monitoring, and a privacy review. Local JSON and uploads are for development only; they are not durable or shared across deployed instances.

## Future improvements

- Add an authenticated civic-team workflow for reviewing reports and changing statuses.
- Add persistent cloud storage, photo retention controls, and protected access.
- Add rate limits, audit events, and operational monitoring.
- Improve location capture and accessible map-based reporting.
- Connect drafting to Bedrock with explicit consent and safe data handling.
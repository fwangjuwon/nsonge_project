# Local development (Windows / PowerShell)

Configured on 2026-11-17.

The root repository records the backend and frontend as Git submodules. To clone
the project with both repositories, use `git clone --recurse-submodules`.

## Addresses

- Frontend: http://localhost:3000
- Backend API: http://localhost:8081/api
- Swagger: http://localhost:8081/swagger-ui/index.html
- PostgreSQL: 127.0.0.1:5434 / creative_studio_db

Port 8080 is already used by the local Oracle listener, so this setup uses 8081.
Port 5432 is used by an existing native PostgreSQL server, so this project's
Docker database uses 5434 on the loopback interface.

## Start

Start Docker Desktop, then run:

```powershell
cd C:\juwon\workspace\nsonge_project\creative-studio-bo
docker compose up -d
.\gradlew.bat bootRun --args="--logging.level.root=INFO"
```

In another PowerShell terminal:

```powershell
cd C:\juwon\workspace\nsonge_project\creative-studio-fo
npm.cmd run local -- --host 127.0.0.1
```

Use Ctrl+C in each terminal to stop its server. To stop PostgreSQL:

```powershell
cd C:\juwon\workspace\nsonge_project\creative-studio-bo
docker compose stop postgres
```

## Local configuration

- Backend `.env`: Gemini API key, generated DB password and JWT secret,
  `SERVER_PORT=8081`, and `IMAGE_BASE_URL=http://localhost:8081`.
  This file is ignored by Git; keep its values private.
- Frontend `.env.local`: `VITE_API_BASE_URL=http://localhost:8081/api`.
- Backend `docker-compose.override.yml`: selects the dedicated
  `nsonge_local_postgres_volume` volume. This override is ignored by Git and
  automatically loaded by `docker compose` in the backend directory.

On a fresh checkout, copy the backend `.env.example` to `.env`, supply your own
Gemini key, DB credentials and JWT secret, and set the local port variables
listed above. The DB host/port in `.env` must be `127.0.0.1` and `5434` for this
Compose override, which should be created locally:

```yaml
services:
  postgres:
    ports: !override
      - "127.0.0.1:5434:5432"
volumes:
  prmpt_postgres_volume:
    name: nsonge_local_postgres_volume
```

The `!override` tag requires Docker Compose 2.24.4 or newer.

- The older `prmpt_postgres_volume` was preserved without resetting its data.
- The new database was initialized using `db/schema.sql` and `db/data.sql`.
  The SSO `nsist_org_mapping` table missing from the original schema was added
  from `creative-studio-bo/docs/erd.md`, both to the schema and this local DB.
  Its initial administrator login ID is `admin`; the seed stores a password hash,
  not a plaintext password. No administrator password was reset during setup.

Keep the local override when starting this database. Removing it selects the
older volume from the base Compose file. Do not use `docker compose down -v`
unless you intend to delete the selected database volume.

## Dependencies

- JDK 17 (installed: Temurin 17.0.17)
- Node.js (installed: 24.12.0), npm (installed: 11.6.2)
- Gradle 8.14.4 via the included wrapper; no separate Gradle install is needed.
- PostgreSQL 15 via Docker; no separate PostgreSQL install is needed.

Frontend dependencies are installed. To reinstall from the lockfile:

```powershell
cd C:\juwon\workspace\nsonge_project\creative-studio-fo
npm.cmd ci
```

## Verification

- Frontend production build passed.
- Frontend, Swagger and OpenAPI document returned HTTP 200.
- Frontend resolves its API URL to `http://localhost:8081/api`.
- Authentication API reaches PostgreSQL and returns HTTP 401 for an unknown user.
- Focused application-context and authentication tests: 11 passed, 2 failed.
  The two failures are SSO success cases expecting a pre-existing employee
  mapping for employee `12496` to `admin`; the seed SQL does not supply it.
  No administrator SSO mapping was added automatically.
- Focused tests excluded the JUnit suite engine through
  `creative-studio-bo/gradle/focused-tests.init.gradle` to avoid unrelated empty-suite failures
  when filtering individual test classes. The full test suite is not certified.
- Gemini credentials are configured; no billable image-generation/API request
  was made as part of environment setup.

## NSmall product image import

On the product-photo tab, choose **NS몰 상품에서 가져오기** in the reference-image
card. Enter a product code or NSmall goods URL. Gallery images can be attached
whole; detail images can be cropped by dragging and attached to the next empty
reference slot. Existing files stay attached, up to the existing three-image limit.

The backend reads the public NSmall product API and extracts image URLs from the
description HTML without rendering that HTML. Supplier-hosted images are fetched
through the authenticated backend. Only product-API-derived image IDs are
accepted by the image endpoint. Connections reject private/special IP addresses,
including redirect targets; downloads are limited to 32 MB and JPEG/PNG/WebP.
Large detail images are cropped and resized before attachment (15 MB limit).

Verified products: `36149436`, `35918024` and `36311160`. Backend tests: 57 passed,
including three live checks against public product/image data. Frontend utility
tests: 3 passed; production build passed. These checks do not invoke Gemini.

Run the focused backend tests from `creative-studio-bo`:

```powershell
$env:NSMALL_LIVE_CHECK='true' # Omit this to skip calls to the public NSmall API.
.\gradlew.bat -I gradle/focused-tests.init.gradle test --tests 'com.nsmall.ns_creative_studio_bo.domain.goods.*'
```

Run frontend checks from `creative-studio-fo`:

```powershell
node --test src/utils/goodsImage.test.js
npm.cmd run build
```

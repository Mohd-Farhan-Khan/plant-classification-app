# Plant Classification App — Project Overview

## What this project does
A lightweight web application that identifies plant species from uploaded images. It provides:
- A modern browser-based UI for image upload and viewing results.
- A FastAPI backend that handles file uploads, runs a TensorFlow/Keras model for classification, and enriches results with structured plant data from a relational database.
- Automatic API documentation and a simple data model for storing plant metadata.

## Who it’s for
- Students and hobbyists exploring computer vision for plants.
- Teams needing a replaceable inference service to classify plant images.
- Anyone wanting a minimal end‑to‑end reference (frontend ⇄ API ⇄ ML model ⇄ database).

## Key features
- Image upload via drag‑and‑drop or file picker.
- Real‑time inference using a pre‑trained Keras/TensorFlow model (.keras or .h5 format).
- Optional database enrichment: after classification, the app looks up the identified plant in the database and returns details such as scientific name, family, origin, uses, and an image URL.
- Clean, responsive UI with progress feedback and confidence meter.
- Auto‑generated interactive API docs (Swagger UI and ReDoc) for developers.
- CORS enabled for easy testing across clients.
- Structured logging for upload and inference flow.

## High‑level architecture
- Client (templates + static):
  - HTML template rendered by the server.
  - CSS for styling and animations.
  - Vanilla JavaScript for file handling, calling the upload endpoint, and updating the results view.
- API layer (FastAPI):
  - Routes for homepage, listing plants, and uploading images for classification.
  - Validates and stores uploaded files to a local uploads directory under static assets.
- ML inference layer (TensorFlow/Keras):
  - Loads a saved model at startup.
  - Preprocesses images to the model’s expected shape and normalization.
  - Returns the top class with a confidence score and a plant/not‑plant flag based on a configurable threshold.
- Data access layer (SQLAlchemy):
  - ORM models for a `plants` table and session management.
  - Optional lookup by classified plant name to attach metadata to the response.
- Storage:
  - Application database: SQLite by default; PostgreSQL supported via environment variable.
  - File storage: uploaded images saved to `static/uploads/`.

## Request → response lifecycle (upload & classify)
1. User selects or drops an image in the browser.
2. The frontend sends a multipart request to the upload endpoint.
3. The server stores the image to a local uploads directory.
4. The TensorFlow/Keras model runs inference on the saved image.
5. If confidence passes the threshold, the result is considered a recognized plant and an optional database lookup is performed to add details.
6. The API returns a JSON payload including filename, predicted class name, confidence, plant flag, and any matched plant metadata.
7. The browser updates the UI with the classification and details.

## Machine learning model
- Framework: TensorFlow/Keras.
- Input preprocessing: image resized to 224×224, converted to array, normalized to 0–1, batch dimension added.
- Output: class index probabilities; top index is selected as the prediction.
- Confidence threshold: 0.70 used to set a boolean “is_plant”.
- Replaceable model artifact: place a new `.keras` or `.h5` file in the `ml/` folder and adjust the configured path if needed.
- Class label mapping (30 plant categories): aloe vera, banana, bilimbi, cantaloupe, cassava, coconut, corn, cucumber, curcuma, eggplant, galangal, ginger, guava, kale, longbeans, mango, melon, orange, paddy, papaya, pepper chili, pineapple, pomelo, shallot, soybeans, spinach, sweet potatoes, tobacco, waterapple, watermelon.

## API surface (summary)
- Homepage (GET): serves the main UI.
- List plants (GET): returns existing plant records (pagination via query params).
- Upload image (POST): accepts an image file and returns classification results with optional plant metadata.
- Test endpoints: available to verify upload mechanics.
- Interactive API docs: Swagger UI and ReDoc.

## Data model (database)
- Single primary table: `plants`.
- Columns: id (PK), name (unique), scientific_name (unique), family, origin, description, uses, image_url.
- Usage: seed this table with your reference plant records to enable metadata enrichment after classification.
- Engines:
  - SQLite (default, file-based, zero-setup).
  - PostgreSQL (recommended for multi-user or production scenarios).

## Configuration and environment
- Environment variables loaded via `.env` (if present).
- Database URL key: `SQLALCHEMY_DATABASE_URL`.
- SQLite uses `check_same_thread=False` for compatibility with FastAPI’s threading model.
- CORS: configured to allow all origins by default (tighten for production).
- Logging: INFO level with clear markers for upload, result, and DB lookup steps.

## Frontend experience
- Responsive single page powered by a template and static assets.
- Drag‑and‑drop uploads, loading state, confidence bar, and live result rendering.
- Optional display of database‑sourced details (family, origin, uses, reference image).

## Technologies used
- Backend framework: FastAPI.
- ASGI server: Uvicorn.
- ORM & database: SQLAlchemy; SQLite by default; PostgreSQL via `psycopg2-binary`.
- Data validation: Pydantic (current codebase uses v1 style configuration).
- Machine learning: TensorFlow/Keras for model loading and inference; NumPy for array ops.
- Image processing: Pillow (PIL) for basic IO and resizing.
- File uploads and async IO: `python-multipart`, `aiofiles`.
- Templating: Jinja2.
- Environment management: `python-dotenv` for `.env` support.
- Frontend: HTML, CSS, vanilla JavaScript, Google Fonts, and Font Awesome icons.

## Operational notes
- Tables are created automatically on app startup if they don’t exist.
- Uploaded files persist in `static/uploads/`; clear periodically if disk space is a concern.
- If the model file is missing or cannot be loaded, the app still starts and the upload endpoint returns a graceful fallback.

## Security & privacy considerations (to address before production)
- File uploads: add MIME/type and size validation; consider antivirus scanning.
- CORS: restrict allowed origins to trusted domains.
- Authentication/authorization: none at present; add for administrative or private endpoints.
- Rate limiting and abuse protection: not implemented.
- Secrets management: ensure `.env` is not committed; use secret stores in production.

## Known limitations
- No CRUD endpoints for adding/updating plant records; enrichment requires manual DB seeding.
- No automated tests included.
- Pydantic compatibility: code uses `orm_mode` (a v1 pattern); if adopting Pydantic v2, switch to `from_attributes=True` in schema config.
- Model class labels are defined in code; externalizing to a config or database would improve maintainability.

## Suggested next steps / roadmap
- Add secure CRUD APIs (and admin UI) for managing plant metadata.
- Add authentication and role‑based authorization.
- Validate and sanitize uploads; store media in object storage.
- Externalize class label mapping; support label syncing to/from DB.
- Batch classification or background tasks for heavy processing.
- Observability: request metrics, structured logs, tracing.
- Containerize and add CI/CD pipeline for reproducible deployments.
- Add unit/integration tests for routes and model wrapper.

## How to run (conceptual)
- Prepare a Python 3.11 environment and install the listed dependencies.
- Ensure the ML model file is present in the `ml/` folder and the database URL is configured (or rely on the SQLite default).
- Start the FastAPI app with an ASGI server and open the base URL in your browser; API docs are available at the standard documentation endpoints.

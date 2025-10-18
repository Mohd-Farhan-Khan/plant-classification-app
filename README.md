# Plant Classification App

## Overview
FastAPI based web app for uploading plant images and getting a predicted class from a pre-trained Keras/TensorFlow model. Includes a simple HTML/CSS/JS frontend, image preprocessing, and a pluggable database layer (SQLite by default; PostgreSQL supported via environment variable).

## Project Structure (current)
```
plant_classification_app
├── main.py                       # FastAPI application entrypoint
├── database.py                   # SQLAlchemy engine/session setup (uses .env if present)
├── models.py                     # SQLAlchemy ORM models
├── schemas.py                    # Pydantic schemas
├── .env.sample                   # Copy to .env and adjust DB URL if needed
├── .gitignore                    # Git ignores (e.g., uploads, venv, caches)
├── ml/
│   ├── model.py                  # Load model + inference helpers
│   ├── preprocessing.py          # Image preprocessing utilities
│   ├── best_plant_classifier.keras / .h5  # Saved model variants
│   └── model.keras               # (Additional model file)
├── static/
│   ├── css/style.css             # Stylesheet (name may differ from earlier docs)
│   ├── js/script.js              # Frontend interactions
│   └── uploads/                  # Uploaded images (runtime generated)
│       └── .gitkeep              # Keeps folder in version control
├── templates/
│   └── main.html                 # Main HTML template
├── requirements.txt              # Python dependencies
└── README.md
```

## Prerequisites
- Python 3.11+
- (Optional) PostgreSQL if you want a production RDBMS

## Environment Configuration
The application reads `SQLALCHEMY_DATABASE_URL` from the environment (loaded via `.env` if present). If not set, it falls back to a local SQLite file `sqlite:///./plants.db`.

You can start from the provided `.env.sample`:
```
# Windows PowerShell (copy sample to .env)
Copy-Item .env.sample .env
```

Or create a `.env` file manually in the project root:
```
# Example for SQLite (default)
# SQLALCHEMY_DATABASE_URL=sqlite:///./plants.db

# Example for PostgreSQL
# SQLALCHEMY_DATABASE_URL=postgresql+psycopg2://username:password@localhost:5432/plants_db

# (Optional) Other environment variables could go here
```

## Setup Instructions
1. Clone the repository
   ```bash
   git clone <repository-url>
   cd plant_classification_app
   ```

2. Create & activate a virtual environment (example name `plantenv` like in repo)
   ```bash
   python -m venv plantenv
   # Windows PowerShell
   .\plantenv\Scripts\Activate.ps1
   # macOS/Linux (bash/zsh)
   # source plantenv/bin/activate
   ```

3. (Optional) Copy `.env.sample` to `.env` and adjust DB settings.

4. Install dependencies
   ```bash
   pip install -r requirements.txt
   ```

5. (Optional) Edit `.env` if you want a DB other than default SQLite.

6. Apply database migrations / create tables (SQLAlchemy `create_all` runs automatically on startup via `main.py`).

7. Run the application
   ```bash
   uvicorn main:app --reload
   ```

8. Open in browser
   http://127.0.0.1:8000

9. Interactive API docs (Swagger UI)
   http://127.0.0.1:8000/docs

## Usage
- Use the homepage form to upload a plant image.
- Backend preprocesses the image and runs inference with the loaded model file (`ml/best_plant_classifier.keras` preferred if present).
- JSON prediction response includes class name, confidence, and a flag indicating if a plant was detected.

## API Endpoints
| Method | Path            | Description | Auth |
|--------|-----------------|-------------|------|
| GET    | /               | Returns HTML UI (`templates/main.html`). | None |
| GET    | /plants         | List all plants in database (paginated via `skip`, `limit`). | None |
| POST   | /upload-image/  | Upload an image file and get classification + optional DB plant info. | None |
| POST   | /test-upload/   | Simple test endpoint to verify upload mechanics (debug). | None |
| GET    | /test-form      | Debug HTML form (template must exist; may be absent). | None |
| GET    | /docs           | Swagger UI (FastAPI auto docs). | None |
| GET    | /redoc          | ReDoc UI. | None |

### Classification Response Example
```json
{
   "filename": "leaf.jpg",
   "prediction": {
      "class_name": "Aloe Vera",
      "confidence": 0.9134,
      "is_plant": true
   },
   "plant_info": {
      "id": 1,
      "name": "Aloe Vera",
      "scientific_name": "Aloe barbadensis miller",
      "family": "Asphodelaceae",
      "origin": "Arabian Peninsula",
      "description": "Succulent plant species of the genus Aloe...",
      "uses": "Medicinal, cosmetic",
      "image_url": "https://example.com/aloe.jpg"
   }
}
```
`plant_info` is null if the classified class is below the confidence threshold, not recognized, or absent in the database.

### Error Response Example
```json
{"detail": "Error saving file: <message>"}
```

## Database Notes & Seeding
Currently only a READ path (`GET /plants`) is exposed; there is no public endpoint to insert/update plant records. To leverage `plant_info` enrichment you must seed the `plants` table manually.

### Quick SQLite Seeding (Python REPL)
```python
from database import SessionLocal
from models import Plant
db = SessionLocal()
plant = Plant(name="Aloe Vera", scientific_name="Aloe barbadensis miller", family="Asphodelaceae", origin="Arabian Peninsula", description="Succulent medicinal plant", uses="Soothing gels", image_url="")
db.add(plant); db.commit(); db.refresh(plant); print(plant.id)
db.close()
```

### Raw SQL (SQLite)
```sql
INSERT INTO plants (name, scientific_name, family, origin, description, uses, image_url)
VALUES ('Aloe Vera','Aloe barbadensis miller','Asphodelaceae','Arabian Peninsula','Succulent medicinal plant','Soothing gels','');
```

## Model Management
`main.py` attempts to load `ml/best_plant_classifier.keras`. If that fails it sets `model = None` and classification endpoint returns a fallback indicating the model is unavailable.

To update the model:
1. Export / train new model (Keras `.keras` or `.h5`).
2. Drop file into `ml/` directory.
3. Update `model_path` variable in `main.py` if filename differs.
4. Restart server.

### Adding Class Labels
Class label mapping lives inside `ml/model.py` (`class_names` dict). Update it if your new model changes index ordering.

### Built-in class labels and threshold
The current model wrapper includes 30 classes mapped by index:

- 0: aloe vera
- 1: banana
- 2: bilimbi
- 3: cantaloupe
- 4: cassava
- 5: coconut
- 6: corn
- 7: cucumber
- 8: curcuma
- 9: eggplant
- 10: galangal
- 11: ginger
- 12: guava
- 13: kale
- 14: longbeans
- 15: mango
- 16: melon
- 17: orange
- 18: paddy
- 19: papaya
- 20: pepper chili
- 21: pineapple
- 22: pomelo
- 23: shallot
- 24: soybeans
- 25: spinach
- 26: sweet potatoes
- 27: tobacco
- 28: waterapple
- 29: watermelon

Notes:
- Returned `class_name` is title-cased before being sent in the API response.
- `is_plant` is set to true when confidence >= 0.70. You can tweak this threshold in `ml/model.py` if desired.

## Environment Variable Summary
| Name | Required | Default | Purpose |
|------|----------|---------|---------|
| SQLALCHEMY_DATABASE_URL | No | sqlite:///./plants.db | Database connection string. SQLite uses `check_same_thread=False`. |

Add more variables as needed (e.g., `MODEL_PATH`, `LOG_LEVEL`) and read them in `main.py` / `database.py` similarly.

## CORS
All origins are currently allowed (`allow_origins=["*"]`). For production, restrict this to trusted domains.

## Logging
Configured at INFO level in `main.py`. Look for loggers with prefixes like `[UPLOAD]`, `[RESULT]`, and `[PLANT_INFO]` for tracing classification flow.

## Troubleshooting
| Issue | Possible Cause | Fix |
|-------|----------------|-----|
| Model not loaded | File path wrong / corrupted model | Verify `model_path`, file existence, and Keras version compatibility. |
| `plant_info` always null | DB empty or name mismatch (case / spaces) | Seed DB; ensure names match formatted class names (capitalized words). |
| SQLite locking errors | Concurrent writes | Consider switching to PostgreSQL in `.env`. |
| High memory usage | TensorFlow GPU / large model | Use a lighter model or enable memory growth for GPU. |
| 422 Unprocessable Entity | Upload field name mismatch | Ensure form field is `file`. |
| 500 on /plants response | Pydantic v2 config mismatch | See Pydantic v2 compatibility note below. |

### Pydantic v2 compatibility
This project currently declares `orm_mode = True` in `schemas.py` (a Pydantic v1 setting). If you install Pydantic v2 (common with modern FastAPI), you may need to update either the dependency or the schema config to serialize ORM objects from SQLAlchemy.

Two easy options:
- Pin Pydantic to v1 in `requirements.txt` (compatible with existing code):
   - Add a version constraint like `pydantic<2` and reinstall.
- OR update `schemas.py` for Pydantic v2:
   - Replace the inner `Config` with: `from pydantic import BaseModel, ConfigDict` and set `model_config = ConfigDict(from_attributes=True)` in `PlantBase`.

FastAPI 0.115+ works well with Pydantic v2 when `from_attributes=True` is set.

## Security Considerations (To Improve Before Production)
- Unrestricted file uploads (no extension / MIME validation).
- CORS wide open.
- No rate limiting or auth.
- Directory listing potential if misconfigured static serving.
- Suggest adding validation, antivirus scanning (e.g., `clamd`), and auth for administrative endpoints.

## Potential Next Steps
- Add CRUD endpoints for Plant records (create/update/delete).
- Externalize class label mapping to JSON or DB.
- Support batch image classification.
- Add async background tasks for heavy preprocessing.
- Write unit tests (currently absent) for model wrapper and API routes.
- Dockerize for reproducible deployment.

## Windows tips
- If PowerShell execution policies block venv activation, run an elevated PowerShell and execute: `Set-ExecutionPolicy -Scope CurrentUser RemoteSigned` once.
- Long TensorFlow installs on Windows are normal; consider a virtual environment per project.

## Key Modules
- `ml/model.py`: wraps Keras model loading & prediction.
- `ml/preprocessing.py`: image resizing / normalization.
- `database.py`: loads `.env`, builds SQLAlchemy engine, defaults gracefully to SQLite.
- `main.py`: FastAPI routes, file upload handling, CORS, logging.

## Dependencies (from `requirements.txt`)
- fastapi
- uvicorn
- sqlalchemy
- pydantic
- tensorflow
- pillow
- python-multipart
- aiofiles
- jinja2
- psycopg2-binary (only needed if using PostgreSQL)
- python-dotenv (for `.env` loading)

## Switching to PostgreSQL
1. Ensure PostgreSQL is running & a database exists (e.g., `plants_db`).
2. Set `SQLALCHEMY_DATABASE_URL` in `.env`:
   ```
   SQLALCHEMY_DATABASE_URL=postgresql+psycopg2://user:pass@localhost:5432/plants_db
   ```
3. Restart the app; tables are created automatically if absent.

## Logging
Configured in `main.py` using Python's `logging` module (INFO level). Adjust as needed.

## Development Tips
- Delete or clear files in `static/uploads/` periodically to save disk space.
- When updating the ML model, drop the new `.keras` or `.h5` file into `ml/` and adjust `model_path` in `main.py` if necessary.
- For reproducible environments, consider freezing exact versions with `pip freeze > requirements.lock`.

## License
MIT License. See LICENSE file if added later.

---
If you see outdated paths or filenames, update this README accordingly; the structure section reflects the current repository state at the time of the last edit.
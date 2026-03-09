# py_builder

`py_builder` is a FastAPI service that orchestrates AWS infrastructure build and unbuild workflows from YAML task files. It uses Jinja2 templates, shell scripts, and optional database-backed status tracking.

## Features

- YAML-driven orchestration (`tasks/<component>.yml`)
- Build and unbuild APIs
- Environment aggregation from global and component YAML files
- Jinja2 rendering for scripts/templates
- Step/application status tracking in SQLite via SQLAlchemy
- Unit tests with `unittest`

## Project Structure

```text
py_builder/
├── main.py
├── database.py
├── models.py
├── schemas.py
├── auth.py
├── routes/
│   ├── build.py
│   ├── unbuild.py
│   ├── status.py
│   ├── environment.py
│   └── resources.py
├── services/
│   ├── base_service.py
│   ├── build_service.py
│   ├── unbuild_service.py
│   ├── status_service.py
│   └── environment_service.py
├── tasks/
│   ├── test_cfn.yml
│   ├── test_cfn_template.yml
│   └── test_custom.yml
├── environments/
├── resources/
├── templates/
├── unit_tests/
├── requirements.txt
├── Dockerfile
└── README.md
```

## Prerequisites

- Python 3.10+
- `pip`
- AWS CLI credentials/profile configured if running real AWS actions

## Installation

```bash
git clone https://github.com/fjin/py_builder.git
cd py_builder

python3 -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install --upgrade pip
pip install -r requirements.txt
```

## Run the API

```bash
uvicorn main:app --host 0.0.0.0 --port 8000 --reload
```

The OpenAPI docs are available at `http://127.0.0.1:8000/docs`.

## API Endpoints

### `POST /build/`

Starts a build for a component. `component` maps to `tasks/<component>.yml`.

Request body:

```json
{
  "component": "test_cfn_template",
  "env_path": "/Users/you/work/py_builder",
  "resource_path": "/Users/you/work/py_builder",
  "task_path": "/Users/you/work/py_builder",
  "db_flag": false
}
```

Notes:

- `env_path`, `resource_path`, and `task_path` are base paths; service appends `environments/`, `resources/`, and `tasks/`.
- `db_flag` is present in the build schema but currently not used by `BuildService`.

### `POST /unbuild/`

Starts an unbuild for a component.

Request body:

```json
{
  "component": "test_cfn_template",
  "task_path": "/Users/you/work/py_builder",
  "db_flag": false
}
```

Notes:

- `db_flag` default is `false`.
- If `db_flag` is `true`, unbuild requires an existing application/build record.
- On successful unbuild, the application record is deleted, so a later status lookup may return not found.

### `GET /status/`

Returns current/latest status for a component:

```text
GET /status/?application_name=test_cfn_template
```

### `GET /environment/`

Loads and merges environment data for the component task file:

```text
GET /environment/?component=test_cfn_template&env_path=/Users/you/work/py_builder&resource_path=/Users/you/work/py_builder&task_path=/Users/you/work/py_builder
```

### `GET /resources/` and `POST /resources/`

Basic resource CRUD over the `Resource` table.

## Curl Examples

Build:

```bash
curl -X POST "http://127.0.0.1:8000/build/" \
  -H "Content-Type: application/json" \
  -d '{
    "component": "test_cfn_template",
    "env_path": "/Users/you/work/py_builder",
    "resource_path": "/Users/you/work/py_builder",
    "task_path": "/Users/you/work/py_builder",
    "db_flag": false
  }'
```

Unbuild:

```bash
curl -X POST "http://127.0.0.1:8000/unbuild/" \
  -H "Content-Type: application/json" \
  -d '{
    "component": "test_cfn_template",
    "task_path": "/Users/you/work/py_builder",
    "db_flag": false
  }'
```

Status:

```bash
curl "http://127.0.0.1:8000/status/?application_name=test_cfn_template"
```

## Tasks, Environments, and Resources

- Tasks are loaded from `tasks/<component>.yml`.
- `load_config()` expects both files below for each task entry:
  - Global: `environments/<environment>.yml`
  - Component-specific: `environments/<resource>/<environment>.yml`
- Resource scripts/configs are loaded from `resources/<resource>/`.
- Shared templates are under `templates/`.

## Database

The default database is SQLite at `test.db`:

```python
DATABASE_URL = "sqlite:///./test.db"
```

Tables are created at app startup (`Base.metadata.create_all(bind=engine)`).

## Testing

Run all unit tests:

```bash
python3 -m unittest discover -s unit_tests
```

Coverage (optional):

```bash
pip install coverage
coverage run -m unittest discover -s unit_tests
coverage report
coverage html
```

## Docker

Build and run:

```bash
docker build -t py_builder_image .
docker run -d -p 8000:8000 -v ~/.aws:/root/.aws py_builder_image
```

## Contributing

Open an issue or submit a pull request.

## License

MIT

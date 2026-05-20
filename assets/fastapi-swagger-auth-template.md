# FastAPI Swagger Auth Template

Copy this code to `app/common/swagger_auth.py`. Env vars: `TM_EXT_SWAGGER_USER`, `TM_EXT_SWAGGER_PASSWORD`

Create the FastAPI app with `docs_url=None, redoc_url=None, openapi_url=None` to disable default public docs, then call `setup_protected_docs(app)`.

```python
import os
import secrets

from fastapi import Depends, FastAPI, HTTPException, status
from fastapi.openapi.docs import get_redoc_html, get_swagger_ui_html
from fastapi.openapi.utils import get_openapi
from fastapi.security import HTTPBasic, HTTPBasicCredentials

security = HTTPBasic()

SWAGGER_USER = os.getenv("TM_EXT_SWAGGER_USER", "")
SWAGGER_PASSWORD = os.getenv("TM_EXT_SWAGGER_PASSWORD", "")


def verify_swagger_credentials(
    credentials: HTTPBasicCredentials = Depends(security),
) -> str:
    if not SWAGGER_USER or not SWAGGER_PASSWORD:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail="Swagger credentials not configured",
        )
    correct_user = secrets.compare_digest(credentials.username, SWAGGER_USER)
    correct_pass = secrets.compare_digest(credentials.password, SWAGGER_PASSWORD)
    if not (correct_user and correct_pass):
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Invalid credentials",
            headers={"WWW-Authenticate": "Basic"},
        )
    return credentials.username


def setup_protected_docs(app: FastAPI) -> None:
    """
    Call AFTER creating the FastAPI app with docs disabled:
        app = FastAPI(docs_url=None, redoc_url=None, openapi_url=None)
        setup_protected_docs(app)
    """

    @app.get("/openapi.json", include_in_schema=False)
    async def get_openapi_json(username: str = Depends(verify_swagger_credentials)):
        return get_openapi(
            title=app.title,
            version=app.version,
            routes=app.routes,
        )

    @app.get("/docs", include_in_schema=False)
    async def custom_swagger_ui(username: str = Depends(verify_swagger_credentials)):
        return get_swagger_ui_html(
            openapi_url="/openapi.json",
            title=f"{app.title} - Swagger UI",
        )

    @app.get("/redoc", include_in_schema=False)
    async def custom_redoc(username: str = Depends(verify_swagger_credentials)):
        return get_redoc_html(
            openapi_url="/openapi.json",
            title=f"{app.title} - ReDoc",
        )
```

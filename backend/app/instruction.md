To change the database handling in your FastAPI application from SQLAlchemy to TortoiseORM, you'll need to update various parts of your application. Here's a step-by-step guide on how to do it:

1. **Install TortoiseORM**:
   First, you need to install TortoiseORM and its dependencies. You can do this using pip:
   ```bash
   pip install tortoise-orm[asyncpg]
   ```

2. **Update Database Configuration**:
   Update your database configuration to use TortoiseORM. Create or update the `config.py` file to include TortoiseORM settings.
   ```python
   # app/core/config.py
   from pydantic import BaseSettings

   class Settings(BaseSettings):
       DATABASE_URL: str = "postgres://user:password@localhost:5432/dbname"
       TORTOISE_ORM = {
           "connections": {"default": DATABASE_URL},
           "apps": {
               "models": {
                   "models": ["app.models", "aerich.models"],
                   "default_connection": "default",
               },
           },
       }
   
   settings = Settings()
   ```

3. **Create TortoiseORM Models**:
   Define your models using TortoiseORM. Create or update the `models.py` file.
   ```python
   # app/models.py
   from tortoise import fields
   from tortoise.models import Model

   class User(Model):
       id = fields.IntField(pk=True)
       username = fields.CharField(max_length=20, unique=True)
       email = fields.CharField(max_length=50, unique=True)
       hashed_password = fields.CharField(max_length=128)
       is_active = fields.BooleanField(default=True)
       is_superuser = fields.BooleanField(default=False)
       created_at = fields.DatetimeField(auto_now_add=True)
       updated_at = fields.DatetimeField(auto_now=True)

       def __str__(self):
           return self.username
   ```

4. **Initialize TortoiseORM**:
   Initialize TortoiseORM in your main application file (`main.py`).
   ```python
   # app/main.py
   import sentry_sdk
   from fastapi import FastAPI
   from fastapi.routing import APIRoute
   from starlette.middleware.cors import CORSMiddleware
   from tortoise.contrib.fastapi import register_tortoise

   from app.api.main import api_router
   from app.core.config import settings

   def custom_generate_unique_id(route: APIRoute) -> str:
       return f"{route.tags[0]}-{route.name}"

   if settings.SENTRY_DSN and settings.ENVIRONMENT != "local":
       sentry_sdk.init(dsn=str(settings.SENTRY_DSN), enable_tracing=True)

   app = FastAPI(
       title=settings.PROJECT_NAME,
       openapi_url=f"{settings.API_V1_STR}/openapi.json",
       generate_unique_id_function=custom_generate_unique_id,
   )

   # Set all CORS enabled origins
   if settings.all_cors_origins:
       app.add_middleware(
           CORSMiddleware,
           allow_origins=settings.all_cors_origins,
           allow_credentials=True,
           allow_methods=["*"],
           allow_headers=["*"],
       )

   app.include_router(api_router, prefix=settings.API_V1_STR)

   # Register TortoiseORM
   register_tortoise(
       app,
       config=settings.TORTOISE_ORM,
       generate_schemas=True,
       add_exception_handlers=True,
   )
   ```

5. **Update CRUD Operations**:
   Update your CRUD operations to use TortoiseORM. Here is an example of creating and retrieving a user.
   ```python
   # app/crud.py
   from app.models import User

   async def create_user(username: str, email: str, hashed_password: str) -> User:
       user = await User.create(
           username=username,
           email=email,
           hashed_password=hashed_password
       )
       return user

   async def get_user_by_username(username: str) -> User:
       return await User.get(username=username)
   ```

6. **Run Database Migrations**:
   Use Aerich to manage database migrations with TortoiseORM. Install Aerich:
   ```bash
   pip install aerich
   ```

   Initialize Aerich:
   ```bash
   aerich init -t app.core.config.TORTOISE_ORM
   ```

   Create the first migration:
   ```bash
   aerich init-db
   ```

   Make changes to your models and run migrations:
   ```bash
   aerich migrate
   aerich upgrade
   ```

By following these steps, you can change your FastAPI application to use TortoiseORM for database handling.
# **Task-Specific Guide: Setting Up `alx_travel_app` with Swagger and MySQL**

---

## **1. Create the Django Project and App**

### **1.1 Install Python and Django**

Ensure Python 3.9+ is installed. Install Django and other required packages:

```bash
pip install django djangorestframework django-cors-headers celery drf-yasg mysqlclient python-environ
```

**Explanation:**

* `django` → web framework.
* `djangorestframework` → build RESTful APIs.
* `django-cors-headers` → handle cross-origin requests (frontend-backend communication).
* `celery` → handle background tasks (optional for now, but useful later).
* `drf-yasg` → automatically generate Swagger/OpenAPI documentation.
* `mysqlclient` → MySQL database connector for Django.
* `django-environ` → read `.env` files for secure environment variables.

---

### **1.2 Create the Project and App**

```bash
django-admin startproject alx_travel_app
cd alx_travel_app
python manage.py startapp listings
```

**Explanation:**

* `alx_travel_app` → main project.
* `listings` → app for managing your models (e.g., travel listings).

Add `listings` and `rest_framework` to `INSTALLED_APPS` in `alx_travel_app/settings.py`:

```python
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    ...,
    'rest_framework',
    'corsheaders',
    'listings',
    'drf_yasg',  # for Swagger
]
```

---

## **2. Environment Variables (.env)**

### **2.1 Create `.env` in Project Root**

```env
DEBUG=True
SECRET_KEY=your-secret-key
MYSQL_DB=alx_travel_db
MYSQL_USER=root
MYSQL_PASSWORD=your-password
MYSQL_HOST=localhost
MYSQL_PORT=3306
```

### **2.2 Load `.env` in `settings.py`**

```python
import environ, os

env = environ.Env(DEBUG=(bool, False))
BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
environ.Env.read_env(os.path.join(BASE_DIR, '.env'))

DEBUG = env('DEBUG')
SECRET_KEY = env('SECRET_KEY')
```

**Explanation:**

* Environment variables keep sensitive info (passwords, secret keys) out of your code.
* `django-environ` reads the `.env` file and allows type casting (e.g., `bool`).

---

## **3. Configure MySQL Database**

Install MySQL server if not already installed. Then configure `settings.py`:

```python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': env('MYSQL_DB'),
        'USER': env('MYSQL_USER'),
        'PASSWORD': env('MYSQL_PASSWORD'),
        'HOST': env('MYSQL_HOST'),
        'PORT': env('MYSQL_PORT'),
        'OPTIONS': {
            'init_command': "SET sql_mode='STRICT_TRANS_TABLES'",
        },
    }
}
```

**Explanation:**

* `ENGINE` → MySQL backend.
* `OPTIONS` → ensures strict SQL mode for data integrity.
* Using environment variables ensures credentials are not hardcoded.

---

## **4. Configure REST Framework**

In `settings.py`, add basic DRF configuration:

```python
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.AllowAny',
    ],
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.BasicAuthentication',
    ],
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10,
}
```

**Explanation:**

* `DEFAULT_PERMISSION_CLASSES` → who can access API endpoints.
* `DEFAULT_AUTHENTICATION_CLASSES` → supports session-based and basic auth.
* Pagination prevents returning too much data at once.

---

## **5. Configure CORS**

Add `corsheaders` middleware **at the top** of `MIDDLEWARE`:

```python
MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    ...
]
```

Add allowed origins:

```python
CORS_ALLOWED_ORIGINS = [
    "http://localhost:3000",  # your frontend dev server
    "https://example.com",
]
CORS_ALLOW_CREDENTIALS = True
```

**Explanation:**
CORS headers allow browsers to communicate with your backend API from a different origin (domain/port).

---

## **6. Add Swagger (drf-yasg)**

Install drf-yasg (already installed). Configure URLs for Swagger:

```python
# alx_travel_app/urls.py
from django.contrib import admin
from django.urls import path, include
from rest_framework import permissions
from drf_yasg.views import get_schema_view
from drf_yasg import openapi

schema_view = get_schema_view(
   openapi.Info(
      title="ALX Travel API",
      default_version='v1',
      description="API documentation for ALX Travel",
   ),
   public=True,
   permission_classes=(permissions.AllowAny,),
)

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('listings.urls')),
    path('swagger/', schema_view.with_ui('swagger', cache_timeout=0), name='schema-swagger-ui'),
]
```

**Explanation:**

* `drf-yasg` generates interactive API docs at `/swagger/`.
* `schema_view.with_ui('swagger')` → Swagger UI.

---

## **7. Initialize Git Repository**

```bash
git init
git add .
git commit -m "Initial commit: Django project setup with REST, CORS, MySQL, and Swagger"
```

**Explanation:**

* Git version control allows tracking changes and collaborating.
* Make initial commit with project setup files.

---

## **8. Optional: Celery for Background Tasks**

```python
# alx_travel_app/celery.py
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'alx_travel_app.settings')

app = Celery('alx_travel_app')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()
```

**Explanation:**

* Celery allows async tasks (sending emails, heavy computations).
* RabbitMQ or Redis is required as a broker.

---

## **9. Create Listings App Model (Optional Example)**

```python
# listings/models.py
from django.db import models

class Listing(models.Model):
    name = models.CharField(max_length=100)
    location = models.CharField(max_length=100)
    description = models.TextField()
    price = models.DecimalField(max_digits=10, decimal_places=2)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.name
```

Migrate:

```bash
python manage.py makemigrations
python manage.py migrate
```

---

## **10. Create CRUD API for Listings**

**Serializer:**

```python
# listings/serializers.py
from rest_framework import serializers
from .models import Listing

class ListingSerializer(serializers.ModelSerializer):
    class Meta:
        model = Listing
        fields = '__all__'
```

**Views:**

```python
# listings/views.py
from rest_framework import generics
from .models import Listing
from .serializers import ListingSerializer

class ListingListCreateView(generics.ListCreateAPIView):
    queryset = Listing.objects.all()
    serializer_class = ListingSerializer

class ListingRetrieveUpdateDestroyView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Listing.objects.all()
    serializer_class = ListingSerializer
```

**URLs:**

```python
# listings/urls.py
from django.urls import path
from .views import ListingListCreateView, ListingRetrieveUpdateDestroyView

urlpatterns = [
    path('listings/', ListingListCreateView.as_view(), name='listing-list-create'),
    path('listings/<int:pk>/', ListingRetrieveUpdateDestroyView.as_view(), name='listing-detail'),
]
```

---

## **11. Test Your API**

* `GET /api/listings/` → List all listings
* `POST /api/listings/` → Create a listing
* `GET /api/listings/<id>/` → Retrieve a single listing
* `PUT /api/listings/<id>/` → Update
* `DELETE /api/listings/<id>/` → Delete

Swagger UI is at: `/swagger/`

---

## ✅ **Summary of Concepts for the Task**

1. **Django Project Setup**: `startproject` + `startapp`.
2. **Environment Variables**: `.env` + `django-environ` for security.
3. **Database**: MySQL configuration via environment variables.
4. **REST Framework**: API endpoints, serializers, generic CBVs.
5. **CORS**: Allow frontend to communicate with backend.
6. **Swagger**: `drf-yasg` auto-docs at `/swagger/`.
7. **Celery (Optional)**: Async tasks.
8. **Git**: Initialize repo and commit project.

---


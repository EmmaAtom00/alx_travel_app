# **Comprehensive Guide: Building a Modern Django CRUD API in 2025**

This guide combines **environment setup, best practices, modern tools, CORS configuration, and CRUD API development**. It’s structured so you can follow step by step.

---

## **1. Project Setup and Environment Management**

### **1.1 Install Python and Django**

Ensure Python 3.9–3.14 is installed. Then install Django:

```bash
python --version
pip install django
```

### **1.2 Create a Django Project**

```bash
django-admin startproject myapi_project
cd myapi_project
```

### **1.3 Install Required Packages**

```bash
pip install djangorestframework django-environ django-cors-headers celery redis black isort
```

Optional (for GraphQL):

```bash
pip install graphene-django
```

Optional (for Django Ninja APIs):

```bash
pip install django-ninja
```

---

## **2. Environment Variables with `.env`**

### **2.1 Create a `.env` file**

Place it in the project root:

```env
DEBUG=True
SECRET_KEY=your-super-secret-key
DATABASE_URL=postgres://user:password@localhost:5432/mydb
SQLITE_URL=sqlite:///my-local-sqlite.db
CACHE_URL=memcache://127.0.0.1:11211
REDIS_URL=rediscache://127.0.0.1:6379/1?password=your-redis-pass
ALLOW_ROBOTS=False
ADMINS="John Doe <john@example.com>"
MANAGERS="Alice <alice@example.com>"
SERVER_EMAIL=webmaster@example.com
```

> Tip: Create `.env.dist` with dummy values for version control reference.

### **2.2 Load `.env` in `settings.py`**

```python
import environ, os

env = environ.Env(DEBUG=(bool, False))
BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
environ.Env.read_env(os.path.join(BASE_DIR, '.env'))

DEBUG = env('DEBUG')
SECRET_KEY = env('SECRET_KEY')

DATABASES = {
    'default': env.db(),
    'extra': env.db_url('SQLITE_URL', default='sqlite:////tmp/my-tmp-sqlite.db')
}

CACHES = {
    'default': env.cache(),
    'redis': env.cache_url('REDIS_URL')
}
```

---

## **3. CORS Configuration**

### **3.1 Install and Configure `django-cors-headers`**

Add to `INSTALLED_APPS` in `settings.py`:

```python
INSTALLED_APPS = [
    ...,
    "corsheaders",
    "rest_framework",
    "books",  # our app
]
```

Add middleware **as high as possible**:

```python
MIDDLEWARE = [
    "corsheaders.middleware.CorsMiddleware",
    "django.middleware.common.CommonMiddleware",
    ...,
]
```

### **3.2 CORS Settings**

```python
CORS_ALLOWED_ORIGINS = [
    "https://example.com",
    "http://localhost:3000",
]

CORS_ALLOW_ALL_ORIGINS = False  # only allow specific domains
CORS_ALLOW_CREDENTIALS = True
CORS_ALLOW_METHODS = (
    *default_methods,
    "PATCH",
)
CORS_ALLOW_HEADERS = (
    *default_headers,
    "my-custom-header",
)
CORS_URLS_REGEX = r"^/api/.*$"
```

---

## **4. Django Best Practices for 2025**

* **Asynchronous Views**: Use `async def` for I/O-bound tasks.
* **Type Hinting**: Use Python type hints (`HttpRequest`, `HttpResponse`, `List`, `Dict`).
* **Efficient Database Queries**: Use `select_related()` and `prefetch_related()` to avoid N+1 queries.
* **Background Tasks**: Use Celery with Redis for async tasks (emails, reports, etc.).
* **Code Formatting**: Use `black` and `isort`.
* **API Structure**: Consider token-based authentication (`rest_framework.authtoken`) and pagination for large datasets.
* **GraphQL**: Use `graphene-django` or `Ariadne` if you want flexible queries.

---

## **5. Create a Django App for CRUD API**

```bash
python manage.py startapp books
```

Add `'books'` to `INSTALLED_APPS`.

---

## **6. Create a Model**

```python
# books/models.py
from django.db import models

class Book(models.Model):
    title = models.CharField(max_length=100)
    author = models.CharField(max_length=100)
    description = models.TextField()
    published_date = models.DateField()

    def __str__(self):
        return self.title
```

Migrate the database:

```bash
python manage.py makemigrations
python manage.py migrate
```

---

## **7. Serializers**

```python
# books/serializers.py
from rest_framework import serializers
from .models import Book

class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = '__all__'
```

---

## **8. Views**

### **8.1 Function-Based Views (FBVs)**

```python
# books/views.py
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework import status
from .models import Book
from .serializers import BookSerializer

@api_view(['GET', 'POST'])
def book_list(request):
    if request.method == 'GET':
        books = Book.objects.all()
        serializer = BookSerializer(books, many=True)
        return Response(serializer.data)
    elif request.method == 'POST':
        serializer = BookSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

@api_view(['GET', 'PUT', 'DELETE'])
def book_detail(request, pk):
    try:
        book = Book.objects.get(pk=pk)
    except Book.DoesNotExist:
        return Response(status=status.HTTP_404_NOT_FOUND)

    if request.method == 'GET':
        serializer = BookSerializer(book)
        return Response(serializer.data)
    elif request.method == 'PUT':
        serializer = BookSerializer(book, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    elif request.method == 'DELETE':
        book.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

### **8.2 Class-Based Views (CBVs)**

```python
# books/views.py
from rest_framework.views import APIView

class BookAPIView(APIView):
    def get(self, request, pk=None):
        if pk:
            try:
                book = Book.objects.get(pk=pk)
                serializer = BookSerializer(book)
                return Response(serializer.data)
            except Book.DoesNotExist:
                return Response(status=status.HTTP_404_NOT_FOUND)
        books = Book.objects.all()
        serializer = BookSerializer(books, many=True)
        return Response(serializer.data)

    def post(self, request):
        serializer = BookSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def put(self, request, pk):
        try:
            book = Book.objects.get(pk=pk)
        except Book.DoesNotExist:
            return Response(status=status.HTTP_404_NOT_FOUND)
        serializer = BookSerializer(book, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

    def delete(self, request, pk):
        try:
            book = Book.objects.get(pk=pk)
            book.delete()
            return Response(status=status.HTTP_204_NO_CONTENT)
        except Book.DoesNotExist:
            return Response(status=status.HTTP_404_NOT_FOUND)
```

### **8.3 Generic CBVs (Recommended)**

```python
# books/views.py
from rest_framework import generics

class BookListCreateView(generics.ListCreateAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer

class BookRetrieveUpdateDestroyView(generics.RetrieveUpdateDestroyAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
```

---

## **9. URL Configuration**

```python
# books/urls.py
from django.urls import path
from .views import BookListCreateView, BookRetrieveUpdateDestroyView

urlpatterns = [
    path('books/', BookListCreateView.as_view(), name='book-list-create'),
    path('books/<int:pk>/', BookRetrieveUpdateDestroyView.as_view(), name='book-detail'),
]
```

Include in `myapi_project/urls.py`:

```python
from django.urls import path, include

urlpatterns = [
    path('api/', include('books.urls')),
]
```

---

## **10. Testing Your API**

### **10.1 Using Postman**

* `POST /api/books/` → Create a book
* `GET /api/books/` → List all books
* `GET /api/books/<id>/` → Retrieve a single book
* `PUT /api/books/<id>/` → Update a book
* `DELETE /api/books/<id>/` → Delete a book

### **10.2 Using Django Test Client**

```python
from django.test import TestCase, Client

client = Client()
response = client.get('/api/books/')
print(response.json())
```

---

## **11. Celery for Background Tasks**

```python
# myapp/tasks.py
from celery import shared_task
from time import sleep

@shared_task
def send_email_task(user_id):
    print(f"Sending email to {user_id}")
    sleep(10)
    print(f"Email sent to {user_id}")
```

Call in view:

```python
send_email_task.delay(user.id)
```

---

## **12. Additional Best Practices**

1. **Authentication**: DRF token-based or JWT.
2. **Validation**: Custom serializer `validate_<field>()` methods.
3. **Pagination**: Use `PageNumberPagination` or `LimitOffsetPagination`.
4. **Async Support**: Use `async def` for views that call external APIs or perform I/O.
5. **GraphQL APIs**: Optional, use `graphene-django` for flexible queries.
6. **Signals**: `django-cors-headers` supports `check_request_enabled` for dynamic CORS rules.
7. **Code Quality**: Always run `black` and `isort`.

---

## ✅ **13. Summary of Flow**

1. Create project → set up `.env` → configure Django settings.
2. Install CORS headers → configure middleware → allow specific origins.
3. Build the app → create model → migrate DB.
4. Create serializer → create views (FBV/CBV/Generic CBV).
5. Configure URLs → include app URLs in project.
6. Test API → optionally add authentication, pagination, and background tasks.
7. Follow best practices: async, type hints, code formatting, and efficient queries.

---

This note is **complete and actionable**, combining all the concepts from your sources **plus essential extras** (authentication, pagination, async, Celery, GraphQL). Following it, you can independently build a modern Django CRUD API with best practices for 2025.

---

If you want, I can also **draw a visual diagram of this project architecture**, showing apps, models, serializers, views, URLs, CORS, and Celery. It makes understanding and implementing much faster.

Do you want me to do that?

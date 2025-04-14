# Django Postgres Extensions

Django Postgres Extensions adds a lot of functionality to `django.contrib.postgres`, specifically in relation to `ArrayField`, `HStoreField` and `JSONField`, including much better form fields for dealing with these field types. The app also includes an **Array Many To Many Field**, so you can store the relationship in an array column instead of requiring an extra database table.

Check out [http://django-postgres-extensions.readthedocs.io/en/latest/](http://django-postgres-extensions.readthedocs.io/en/latest/) to get started.

**Latest release (0.9.3) tested with Django 2.0.2**

---

## Feature Overview

### Custom Postgres Backend

The customized Postgres backend adds the following features:

- HStore Extension is automatically activated when a test database is created so you don't need to create a separate migration.
- Uses a different update compiler which adds some functionality outlined in the ArrayField section below.
- If `db_index=True` for an `ArrayField`, a GIN index will be created which is more useful than the default database index for arrays.
- Adds some extra operators to enable `ANY` and `ALL` lookups.

---

### ArrayField

The included `ArrayField` is subclassed from `django.contrib.postgres.fields.ArrayField` and is a drop-in replacement.

Features:

- Get array values by index.
- Update array values by index.
- Database functions for interacting with arrays (auto-converts arguments to expressions).
- Add an array of values to an existing field (requires `output_field`).
- Additional lookups: `ANY`, `ALL`.
- Use either a split array field or a multiple choice field in a model form.

---

### HStoreField

The included `HStoreField` is subclassed from `django.contrib.postgres.fields.HStoreField` and is a drop-in replacement.

Features:

- Get HStore values by key.
- Update HStore by specific keys without overwriting others.
- Database functions for interacting with HStores (auto-converts arguments).
- Nested form field for better representation (via key list or form field list).

---

### JSONField

The included `JSONField` is subclassed from `django.contrib.postgres.fields.JSONField` and is a drop-in replacement.

Features:

- Get JSON values by key or key path.
- Update JSON by specific keys, leaving others untouched.
- Delete JSON by key or key path.
- Database functions for JSONField interaction (auto-converts arguments).
- Nested form field (via key list or form field list, same as HStore).

---

## ModelAdmin

Example `models.py`:

```python
from django.db import models
from django_postgres_extensions.models.fields import HStoreField, JSONField, ArrayField
from django_postgres_extensions.models.fields.related import ArrayManyToManyField
from django import forms
from django.contrib.postgres.forms import SplitArrayField
from django_postgres_extensions.forms.fields import NestedFormField

details_fields = (
    ('Brand', NestedFormField(keys=('Name', 'Country'))),
    ('Type', forms.CharField(max_length=25, required=False)),
    ('Colours', SplitArrayField(base_field=forms.CharField(max_length=10, required=False), size=10)),
)

class Buyer(models.Model):
    time = models.DateTimeField(auto_now_add=True)
    name = models.CharField(max_length=20)

    def __str__(self):
        return self.name

class Product(models.Model):
    name = models.CharField(max_length=15)
    keywords = ArrayField(models.CharField(max_length=20), default=[], form_size=10, blank=True)
    sports = ArrayField(models.CharField(max_length=20), default=[], blank=True, choices=(
        ('football', 'Football'), 
        ('tennis', 'Tennis'), 
        ('golf', 'Golf'), 
        ('basketball', 'Basketball'), 
        ('hurling', 'Hurling'), 
        ('baseball', 'Baseball')
    ))
    shipping = HStoreField(keys=('Address', 'City', 'Region', 'Country'), blank=True, default={})
    details = JSONField(fields=details_fields, blank=True, default={})
    buyers = ArrayManyToManyField(Buyer)

    def __str__(self):
        return self.name

    @property
    def country(self):
        return self.shipping.get('Country', '')
```

Example `admin.py`:

```python
from django.contrib import admin
from django_postgres_extensions.admin.options import PostgresAdmin
from models import Product, Buyer

class ProductAdmin(PostgresAdmin):
    filter_horizontal = ('buyers',)
    fields = ('name', 'keywords', 'sports', 'shipping', 'details', 'buyers')
    list_display = ('name', 'keywords', 'shipping', 'details', 'country')

admin.site.register(Buyer)
admin.site.register(Product, ProductAdmin)
```

*Form field screenshot (admin_form.jpg)*  
*List display screenshot (admin_list.jpg)*

---

### Additional Queryset Methods

Adds `format` method to all querysets. This defers a field and adds an annotation with a different format.

Example (return an `HStoreField` as JSON):

```python
qs = Model.objects.all().format('description', HstoreToJSONBLoose)
```

---

### Array Many To Many Field

The `ArrayManyToManyField` is a drop-in replacement for Django's standard `ManyToManyField`.

Supports:

- Descriptor queryset with `add`, `remove`, `clear`, `set` for both forward and reverse relationships.
- `prefetch_related` for both forward and reverse relationships.
- Lookups across relationships using `filter` (forward and reverse).
- Lookups across relationships using `exclude` (forward only).

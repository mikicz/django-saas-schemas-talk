---
lineNumbers: true
download: true
theme: dracula
highlighter: shiki
favicon: '/favicon.png'
fonts:
  sans: "Barlow"
---

# **Using Postgres schemas to separate** 
# **data of your SaaS application in Django**

### a talk by Mikuláš Poul
### July 18th, 2028 - [<img src="/europython.png" style="display: inline; height: 25px; margin-top: -5px" />](https://ep2025.europython.eu/) EuroPython

---

# About me

- he/him, go by Miki
- Born in the Czech Republic
- Live in London, UK

<v-clicks>

- Coding in Python for more years than not
- Have been using Django for 11 years
- Staff Engineer at [<img src="/xelix.svg" style="display: inline; height: 25px; margin-top: -10px" />](https://xelix.com/)
- [www.mikulaspoul.cz](https://www.mikulaspoul.cz/)

</v-clicks>

<!--
Mention all links & relevant details are on mikulaspoul, plus a blog and link to this talk
-->

---
layout: two-cols
---

# Links

## Slides

<img src="/slides.svg" style="width: 80%;background-color: white; margin: 5px auto 0;"  alt="QR Code"/>

::right::

# &nbsp;

## Contact details

- https://www.mikulaspoul.cz/talks/django-sass-schemas-talk/
- https://www.mikulaspoul.cz/
- mikulaspoul@gmail.com
- mikulas.poul@xelix.com

---
layout: section
---

# SaaS

---

# What even is SaaS 

- *Software as a Service* is a method of providing software
- Provided in the cloud managed by the vendor
- This is often a web application
- In contrast to e.g. on-promise model

---

# Tenants in Saas

- A common concept in SaaS is a tenant
- For example, each customer of Xelix is a tenant
- Infrastructure is shared between tenants

--- 

# Let's build a SaaS app

- We want to sell to multiple companies
- Our core business is to find duplicates between invoices
- We want to show invoices per supplier to the users
- Based on Xelix itself

---

# Basic tenant setup in Django

```python
from django.db import models
from django.contrib.auth.models import AbstractUser

class Tenant(models.Model):
    name = models.CharField(max_length=255)

class User(AbstractUser):
    tenant = models.ForeignKey(Tenant, on_delete=models.CASCADE)
    
class Supplier(models.Model):
    tenant = models.ForeignKey(Tenant, on_delete=models.CASCADE)
    name = models.CharField(max_length=255)
    
class Invoice(models.Model):
    supplier = models.ForeignKey(Supplier, on_delete=models.CASCADE)
    number = models.CharField(max_length=255)

class DuplicateInvoice(models.Model):
    invoice1 = models.ForeignKey(Invoice, on_delete=models.CASCADE)
    invoice2 = models.ForeignKey(Invoice, on_delete=models.CASCADE)
```

---

# Querying data for the current user

```python
Supplier.objects.filter(tenant=request.user.tenant)
```

```python
Invoice.objects.filter(supplier__tenant=request.user.tenant)
```

```python
DuplicateInvoice.objects.filter(
    invoice1__supplier__tenant=request.user.tenant,
    invoice2__supplier__tenant=request.user.tenant,
)
```

- Every single database query needs filtering
- It's worse the deeper your schema is

<!--
This in theory could be just one of the filters, but there are no possible constraints to ensure this is the case in DB.
-->

---
layout: section
---

# Issues with setup 

---

# The constant filtering

- You need to remember every single time
  - The `django-scopes` package can make it easier
- Every case needs tests
- Forgetting has severe consequences

---

# Query speeds

- Select queries are slower due to filtering and joins
- Insert queries are slower due to updating larger tables and indexes
- Denormalisation can help

```python
class DuplicateInvoice(models.Model):
    tenant = models.ForeignKey(Tenant, on_delete=models.CASCADE)
    invoice1 = models.ForeignKey(Invoice, on_delete=models.CASCADE)
    invoice2 = models.ForeignKey(Invoice, on_delete=models.CASCADE)

DuplicateInvoice.objects.filter(tenant=request.user.tenant)
```

---

# Database size impact

- Tables and indexes grow in size more quickly
- Especially if denormalised
- Large tables affect all tenants

<!--
~381MiB for 50M FKs in index
~1GiB for 50M FKs in index
-->

---
layout: section
---

# PostgreSQL schemas

---

# PostgreSQL schemas

- Schemas in PostgreSQL refer to its namespace solution
- By default data belongs to the `public` schema
- Each database can have multiple schemas
- Each schema has its own tables, indexes and data
- Tables can refer to tables in other schemas

---

# Using PostgreSQL schemas in SQL

```sql
CREATE SCHEMA tenant1;
CREATE SCHEMA tenant2;
CREATE TABLE tenant1.supplier_supplier (...);
CREATE TABLE tenant2.supplier_supplier (...);
```

<v-clicks>

```sql
SELECT * FROM tenant1.supplier_supplier;  /* returns data from tenant 1 */
```

```sql
SET search_path to 'tenant1';  /* sets namespace to tenant 1 */

SELECT * FROM supplier_supplier;  /* returns data from the active namespace */
SELECT * FROM tenant2.supplier_supplier;  /* returns data from tenant 2 */
```

```sql
SET search_path to 'tenant1', 'public';  /* primary / secondary namespace */

SELECT *
FROM supplier_supplier
JOIN user_user on supplier_supplier.owner_id = user_user.id;
```

</v-clicks>

---
layout: section
---

# django-tenants

---

# django-tenants
- A library which allows the utilisation of schemas in Django
- Full disclaimer, not written or maintained by me, maintained by Tom Turner
- Fork of earlier `django-tenant-schemas` by Bernardo Pires
- Have used both, for about a decade now
- Splits Django apps into shared and tenanted apps
- Routes requests to correct tenant by subdomain

<!--
Maintained by Tom Turner and Alexander Todorov
-->

---

# Addresses our issues

- Tenant is set and forget
- Do not need to remember to filter everywhere
- Queries are faster without the filters
- Tables and indexes don't grow as quickly

## With extra features

- Performance issues localised to tenants
- Full data segragation
- Easy to export / delete a specific tenant

---

# What django-tenants manages

- The lifecycle of the schemas
- Migrations in individual schemas
- Routing requests to the correct tenant
- Tenant-aware file handling (static, media)
- And more - check out the docs!

---

# Installation

- Update database settings
- Create the tenant model
- Configure which apps are shared and which are tenanted
- Add middleware

--- 

# Tenant models

```python
# src/tenant/models.py
from django.db import models
from django_tenants.models import TenantMixin, DomainMixin

class Tenant(TenantMixin):
    name = models.CharField(max_length=255)

class Domain(DomainMixin):
    # contains FK to Tenant and a domain
    pass
```

```python
# settings.py
TENANT_MODEL = "tenant.Tenant"  # todo check
DOMAIN_MODEL = "tenant.Domain"
```

---

# Defining shared and tenanted apps

```python
# settings.py
SHARED_APPS = [
  "django_tenants",
  "src.public.tenant",
  "src.public.user",
  ...
]
TENANT_APPS = [
  "src.tenant.supplier",
  "src.tenant.invoice",
  ...
]
INSTALLED_APPS = SHARED_APPS + TENANT_APPS 
```

---

# Middleware

```python
MIDDLEWARE = (
    'django_tenants.middleware.main.TenantMainMiddleware',
    #...
)
```

- Retrieves domain from request 
- Looks up `Domain` instance
- Activates the linked tenant

```python
tenant = ...  # look up the tenant from the domain
request.tenant = tenant
connection.set_tenant(request.tenant)
```

--- 

# Alternative methods of routing

- Default routing is by domain or subdomain
- django-tenants also supports routing by URL path (e.g. `/r/tenant1/`)
- Easy to hack for other methods

---

# Custom routing

- At Xelix we have custom middleware to route by user
- Staff users can pick which tenant they impersonate

```python
if request.user.is_staff:
  tenant = ... # set tenant from session storage
else:
  tenant = request.user.tenant
request.tenant = tenant
connection.set_tenant(request.tenant)
```

---

# Futher examples from Xelix

- Main deployment has 125 tenants
- ~8 tables in the public schema
- ~280 tables in each tenant schema
- Number of objects
  - 6M suppliers
  - 370M invoices
  - 1.5 duplicates

---
layout: section
---

# New and exciting problems

---

# Migrations

- Need to run migrations for public schema and each tenant
  - Slower
  - Migration progress can be inconsistent between tenants
- django-tenants can run these in parallel
- At xelix we use a "smart" executor, which we open-sourced
  - [django-tenants-smart-executor](https://pypi.org/project/django-tenants-smart-executor/)

---

# Cross-tenant analytics

- By definition the data is segragated
- Sometimes you want to run queries / scripts across all tenants
- You can create a SQL view which unifies data across all tenants
- Run query in each tenant and join the results

```python
for tenant in Tenant.objects.all():
    with tenant:
        ...  # run queries
```

---

# Harder testing

- In your tests you always need to create a tenant
- The tenant needs running migrations to work
- This slows down tests
- At Xelix we 
  - Use session-scoped fixtures for tenants in tests
  - Run migrations on one tenant, replicate schema to others

---

# Migrating to django-tenants

- Migrating an existing project is not trivial
- It is doable with down-time
- Probably a job for a custom SQL / Django management command

<!--
Took me 3 weeks to implemenent and release (very early on)
-->

---
layout: section
---

# Conclusion

---

# What have we learned

TODO

---
layout: section
---

# Questions?

---
layout: two-cols
---

# Links

## Slides

<img src="/slides.svg" style="width: 80%;background-color: white; margin: 5px auto 0;"  alt="QR Code"/>

::right::

# &nbsp;

## Contact details

- https://www.mikulaspoul.cz/talks/django-sass-schemas-talk/
- https://www.mikulaspoul.cz/
- mikulaspoul@gmail.com
- mikulas.poul@xelix.com

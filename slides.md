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

</v-clicks>

<!--
Mention Xelix just went through series B
Mention hiring (with right to work in the UK)
-->

---
layout: two-cols
---

# Links

## Slides

<img src="/slides.svg" style="width: 75%;background-color: white; margin: 5px auto 0;"  alt="QR Code"/>

::right::

# &nbsp;

## Contact details

- [mikulaspoul.cz](https://www.mikulaspoul.cz/)
- mikulaspoul@gmail.com
- mikulas.poul@xelix.com

---

# Contents

- Brief overview of SaaS setup and its issues
- Introduction to PostgreSQL schemas
- Introduction to `django-tenants`
- How we use it at [<img src="/xelix.svg" style="display: inline; height: 17px; margin-top: -5px" />](https://xelix.com/)
- Trade-offs with this solution

<v-click>

- Soft pre-requisites: Django & SQL

</v-click>

---
layout: section
---

# SaaS

---

# What even is SaaS 

- *Software as a Service* is a method of providing software
- Provided in the cloud managed by the seller

<v-clicks>

- This is often a web application
- In contrast to e.g. on-promise model

</v-clicks>

---

# Tenants in SaaS

<v-clicks>

- A common concept in SaaS is a tenant
- For example, each customer of SaaS is a tenant
- Infrastructure is shared between tenants

</v-clicks>

--- 

# Let's build a SaaS app

<v-clicks>

- We want to sell to multiple companies
- Core business is finding duplicates between invoices
- We want to also show invoices and suppliers to the users
- Needs to be fast, secure and reliable
- Based on [<img src="/xelix.svg" style="display: inline; height: 17px; margin-top: -5px" />](https://xelix.com/) itself

</v-clicks>

---

# Basic application setup in Django

```python
# src/tenant/models.py
class Tenant(models.Model):
    name = models.CharField(max_length=255)
```

<v-clicks>

```python
# src/user/models.py
class User(AbstractUser):
    tenant = models.ForeignKey(Tenant, on_delete=models.CASCADE)
```


```python
# src/supplier/models.py
class Supplier(models.Model):
    tenant = models.ForeignKey(Tenant, on_delete=models.CASCADE)
    name = models.CharField(max_length=255)
```

</v-clicks>

---

# Basic application setup in Django

```python
# src/invoice/models.py
class Invoice(models.Model):
    supplier = models.ForeignKey(Supplier, on_delete=models.CASCADE)
    number = models.CharField(max_length=255)

class DuplicateInvoice(models.Model):
    invoice1 = models.ForeignKey(Invoice, on_delete=models.CASCADE)
    invoice2 = models.ForeignKey(Invoice, on_delete=models.CASCADE)
```


---

# Querying data for the current user

<v-click>

```python
Supplier.objects.filter(tenant=request.user.tenant)
```

</v-click>

<v-click>

```python
Invoice.objects.filter(supplier__tenant=request.user.tenant)
```

</v-click>

<v-click>

```python
DuplicateInvoice.objects.filter(
    invoice1__supplier__tenant=request.user.tenant,
    invoice2__supplier__tenant=request.user.tenant,
)
```

</v-click>

<v-click>


<div style="text-align: center; padding-top: 50px;">

## Every single database query needs filtering

</div>
</v-click>

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

<v-clicks>

- Every case needs tests
- Forgetting has severe consequences

</v-clicks>

---

# Query speeds

- Select queries are slower due to filtering and joins
- Insert queries are slower due to updating larger tables and indexes

<v-click>

- Denormalisation can help

```python
# src/invoice/models.py
class DuplicateInvoice(models.Model):
    tenant = models.ForeignKey(Tenant, on_delete=models.CASCADE)
    invoice1 = models.ForeignKey(Invoice, on_delete=models.CASCADE)
    invoice2 = models.ForeignKey(Invoice, on_delete=models.CASCADE)

# querying
DuplicateInvoice.objects.filter(tenant=request.user.tenant)
```

</v-click>

---

# Database size impact

<v-clicks>

- Tables and indexes grow in size more quickly
- Especially if denormalised
- Large tables affect all tenants equally
- Example of denormalisation -- invoice table with ~460M rows
  - Tenant column adds 3.4GiB of data, 2.8GiB of indexes

</v-clicks>

<!--
~381MiB for 50M FKs in data
~1GiB for 50M FKs in index
-->

---
layout: section
---

# PostgreSQL schemas

<v-click>

## Namespaces are one honking great idea -- let's do more of those!

</v-click>

---

# PostgreSQL schemas

- Schemas in PostgreSQL refer to its namespace solution

<v-clicks>

- By default data belongs to the `public` schema
- Each database can have multiple schemas
- Each schema has its own tables, indexes and data
- Tables can refer to tables in other schemas

</v-clicks>

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

</v-clicks>

---

# Using PostgreSQL schemas in SQL


```sql
SET search_path to 'tenant1';  /* sets namespace to tenant 1 */

SELECT * FROM supplier_supplier;  /* returns data from the active namespace */
SELECT * FROM tenant2.supplier_supplier;  /* returns data from tenant 2 */
```


<v-clicks>

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

<v-clicks>

- Fork of earlier `django-tenant-schemas` by Bernardo Pires
- Have used both, for about a decade now
- Splits Django apps into shared and tenanted apps
- Routes requests to correct tenant by subdomain

</v-clicks>

<!--
Maintained by Tom Turner and Alexander Todorov
-->

---

# Addresses our issues

<v-clicks>

- Tenant is set and forget
- Do not need to remember to filter everywhere
- Queries are faster without the filters
  - Example of query speed -- invoice table with ~460M rows
  - Querying for all invoices for tenant with 50M invoices: 26s
  - Querying a table for identical invoices in separate table: 4s
- Tables and indexes don't grow as quickly

</v-clicks>

---

# Additionally leads to

<v-clicks>

- Performance issues localised to tenants
- Full data segregation
- Easy to export / delete a specific tenant
- Smaller locks on tables in migrations

</v-clicks>

---

# What django-tenants manages

<v-clicks>

- The lifecycle of the schemas
- Migrations in individual schemas
- Routing requests to the correct tenant
- Tenant-aware file handling (static, media)
- And more - check out the docs!

</v-clicks>

---

# Installation

<v-clicks>

- Update database settings
- Create the tenant model
- Configure which apps are shared and which are tenanted
- Add middleware

</v-clicks>

--- 

# Database settings

```python {all|5,10}
# settings.py

DATABASES = {
    "default": {
        "ENGINE": "django_tenants.postgresql_backend",
        # ..
    }
}

DATABASE_ROUTERS = ("django_tenants.routers.TenantSyncRouter",)

```

---

# Tenant models

```python {all|5,8|all}
# src/public/tenant/models.py
from django.db import models
from django_tenants.models import TenantMixin, DomainMixin

class Tenant(TenantMixin):
    name = models.CharField(max_length=255)

class Domain(DomainMixin):
    # contains FK to Tenant and a domain
    pass
```

<v-click>

```python
# settings.py
TENANT_MODEL = "tenant.Tenant"
TENANT_DOMAIN_MODEL = "tenant.Domain"
```

</v-click>


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

<v-clicks>

- Retrieves domain from the request 
- Looks up `Domain` instance
- Activates the linked tenant

```python
tenant: Tenant = ...  # look up the tenant from the domain

request.tenant = tenant
connection.set_tenant(request.tenant)
```

</v-clicks>

---

# Querying data for the current user with django-tenants

<v-clicks>

```python
Supplier.objects.all()
```


```python
Invoice.objects.all()
```


```python
DuplicateInvoice.objects.all()
```


<div style="text-align: center; padding-top: 50px;">

## Models from tenant apps do not need filtering for tenant

</div>

</v-clicks>

---
layout: section
---

# django-tenants at [<img src="/xelix.svg" style="display: inline; height: 50px; margin-top: -19px" />](https://xelix.com/)

---

# Alternative methods of routing

- Default routing is by domain or subdomain
- django-tenants also supports routing by URL path (e.g. `/r/tenant1/`)
- Easy to hack other methods

---

# Alternative methods of routing

- Default routing is by domain or subdomain
- django-tenants also supports routing by URL path (e.g. `/r/tenant1/`)
- Easy to ~~hack~~ implement other methods

---

# Custom routing

- At [<img src="/xelix.svg" style="display: inline; height: 17px; margin-top: -5px" />](https://xelix.com/) we have a custom middleware to route by user
- Staff users can pick which tenant they impersonate

<v-click>

```python
tenant: Tenant
if request.user.is_staff:
  tenant = ... # set tenant from session storage
else:
  tenant = request.user.tenant

request.tenant = tenant
connection.set_tenant(request.tenant)
```

</v-click>


---

# Some numbers from [<img src="/xelix.svg" style="display: inline; height: 35px; margin-top: -15px" />](https://xelix.com/)

- Main deployment has ~ 130 tenants
- ~ 80 tables in the public schema
- ~ 300 tables in each tenant schema

<v-click>

- Number of objects
  - Millions of suppliers
  - Hundreds of millions of invoices
  - Millions of duplicates
  - ~1.5 billion in another table relating to invoices

</v-click>

---
layout: section
---

# New and exciting problems

---

# Migrations

- Need to run migrations for public schema and each tenant
  - This is slower
  - Migration progress can be inconsistent between tenants

<v-clicks>

- django-tenants can run these in parallel
- At [<img src="/xelix.svg" style="display: inline; height: 17px; margin-top: -5px" />](https://xelix.com/) we use a "smart" executor, which we open-sourced
  - [django-tenants-smart-executor](https://pypi.org/project/django-tenants-smart-executor/)

</v-clicks>

---

# Cross-tenant analytics

<v-clicks>

- By definition the data is segragated
- Sometimes you want to run queries / scripts across all tenants
- You can create a SQL view which unifies data across all tenants
- Or run query in each tenant and join the results

</v-clicks>

<v-click>

```python
for tenant in Tenant.objects.all():
    with tenant:
        ...  # run queries
```

</v-click>

---

# Harder testing

<v-clicks>

- In your tests you always need to create a tenant
- The tenant needs running migrations to work
- This slows down tests
- At [<img src="/xelix.svg" style="display: inline; height: 17px; margin-top: -5px" />](https://xelix.com/) we 
  - Use session-scoped fixtures for tenants in tests
  - Run migrations on one tenant, replicate schema to others

</v-clicks>


---

# Migrating to django-tenants

<v-clicks>

- Migrating an existing project is not trivial
- It is doable with down-time
- Probably a job for a custom SQL / Django management command

</v-clicks>

<!--
Took me 3 weeks to implemenent and release (very early on)
-->

---
layout: section
---

# Conclusion

---

# What have we learned

- Building an SaaS is hard
- `django-tenants` can help with several aspects
- Trade-off like everything else 

---
layout: two-cols
---

# Questions?

## Slides

<img src="/slides.svg" style="width: 75%;background-color: white; margin: 5px auto 0;"  alt="QR Code"/>

::right::

# &nbsp;

## Contact details

- [mikulaspoul.cz](https://www.mikulaspoul.cz/)
- mikulaspoul@gmail.com
- mikulas.poul@xelix.com

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

# What even is SaaS 

- *Software as a Service* is a method of providing software
- Provided in the cloud on a managed by the vendor
- This is often a web application
- In contrast to e.g. on-promise model

---

# Tenants in Saas

- A common concept in a SaaS is a tenant
- For example, each customer of Xelix is a tenant
- Infrastructure is shared between tenants

--- 

# Let's build a tenant app

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
- Especially if dernomalised (trade-off)
- Large tables affect all tenants

<!---
~381Mb for 50M FKs in index
~1GiB for 50M FKs in index
--->


---

# Postgres schemas [3m]

---

# Django-tenants [3m]

---


# Using django-tenants [8m]


---


# New and exciting problems [5m]

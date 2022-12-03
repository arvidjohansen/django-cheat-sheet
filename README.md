# django-cheat-sheet
Welcome to this Django cheat sheet

Here I will gather all my go-to commands / templates / copy pastes / code syntaxes that I use when working with Django.

# Links
1. [Eric the coder - My beloved Django cheat sheet](https://dev.to/ericchapman/my-beloved-django-cheat-sheet-2056)
2. [Lucrae @ github django-cheat-sheet](https://github.com/lucrae/django-cheat-sheet)

# Full installation on Debian 11.5 w Nginx Gunicorn Postgres
Following guide [Guide on Digitcalocean](https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-22-04)

```sh
sudo apt update
sudo apt install python3-venv python3-dev libpq-dev postgresql postgresql-contrib nginx curl git
```
`sudo -u postgres psql`

Script to create db, user, give acces and set some tuning parameters:
```sql
CREATE DATABASE mdb;
CREATE USER mdb WITH PASSWORD 'MyPAsswordInTheFace1';

ALTER ROLE mdb SET client_encoding TO 'utf8';
ALTER ROLE mdb SET default_transaction_isolation TO 'read committed';
ALTER ROLE mdb SET timezone TO 'UTC';

GRANT ALL PRIVILEGES ON DATABASE mdb TO mdb;
```

```sh
ssh-keygen
# Fix authorized keys
mkdir mdb
cd mdb
python3 -m venv venv
source venv/bin/activate
# Fix github key access
git clone git@github.com:arvidjohansen/django-snippets.git
cd django-snippets
pip install -r requirements.txt
# ./manage.py runserver if you want, but dont forget ufw allow 8k
# Rdy for Nginx
```

## Or with apache instead:
https://gist.github.com/Kyle-Koivukangas/9f6627b03c2d80ecb4b4f722ea26da35
https://docs.djangoproject.com/en/4.1/howto/deployment/wsgi/modwsgi/





# Admin
[Django Admin Cookbook - How to set ordering of apps in admin-dashboard](https://books.agiliq.com/projects/django-admin-cookbook/en/latest/set_ordering.html)


# Models

## Fixtures
Best practice is to always create fixture without contenttypes and authpermissions.
```python
python manage.py dumpdata --natural-foreign --natural-primary -e contenttypes -e auth.Permission --indent 4 > dump.json
```

## Django ORM optimization
gist from [levidyrek @ github](https://gist.github.com/levidyrek/6db1cf88b953f3f006bf678a0f09da8e)
```python
"""
Django ORM Optimization Tips

Caveats:
* Only use optimizations that obfuscate the code if you need to.
* Not all of these tips are hard and fast rules.
* Use your judgement to determine what improvements are appropriate for your code.
"""

# ---------------------------------------------------------------------------
# 1. Profile
# ---------------------------------------------------------------------------

## Use these tools:
## * django-debug-toolbar
## * QuerySet.explain()


# ---------------------------------------------------------------------------
# 2. Be aware of QuerySet's lazy evaluation.
# ---------------------------------------------------------------------------

## 2a. When QuerySets are evaluated

# Iteration
for person in Person.objects.all():
    # Some logic

# Slicing/Indexing
Person.objects.all()[0]

# Pickling (i.e. serialization)
pickle.dumps(Person.objects.all())

# Evaluation functions
repr(Person.objects.all())
len(Person.objects.all())
list(Person.objects.all())
bool(Person.objects.all())

# Other
[person for person in Person.objects.all()]  # List comprehensions
person in Person.objects.all()  # `in` checks

## 2b. When QuerySets are cached/not cached

### Not Cached

# Not reusing evaluated QuerySets
print([p.name for p in Person.objects.all()])  # QuerySet evaluated and cached
print([p.name for p in Person.objects.all()])  # New QuerySet is evaluated and cached

# Slicing/indexing unevaluated QuerySets
queryset = Person.objects.all()
print(queryset[0])  # Queries the database
print(queryset[0])  # Queries the database again

# Printing
print(Person.objects.all())

### Cached

# Reusing an evaluated QuerySet
queryset = Person.objects.all()
print([p.name for p in queryset])  # QuerySet evaluated and cached
print([p.name for p in queryset])  # Cached results are used

# Slicing/indexing evaluated QuerySets
queryset = Person.objects.all()
list(queryset)  # Queryset evaluated and cached
print(queryset[0])  # Cache used
print(queryset[0])  # Cache used


# ---------------------------------------------------------------------------
# 3. Be aware of which attributes are not cached.
# ---------------------------------------------------------------------------

## Not initially retrieved/cached

# Foreign-key related objects
person = Person.objects.get(id=1)
person.father  # foreign object is retrieved and cached
person.father  # cached version is used

## Never cached

# Callable attributes
person = Person.objects.get(id=1)
person.children.all()  # Database hit
person.children.all()  # Another database hit


# ---------------------------------------------------------------------------
# 4. Use select_related() and prefetch_related() when you will need everything.
# ---------------------------------------------------------------------------

# DON'T
queryset = Person.objects.all()
for person in queryset:
    person.father  # Foreign key relationship results in a database hit each iteration

# DO
queryset = Person.objects.all().select_related('father')  # Foreign key object is included in query and cached
for person in queryset:
    person.father  # Hits the cache instead of the database


# ---------------------------------------------------------------------------
# 5. Try to avoid database queries in a loop.
# ---------------------------------------------------------------------------

# DON'T (contrived example)
filtered = Person.objects.filter(first_name='Shallan', last_name='Davar')
for age in range(18):
    person = filtered.get(age=age)  # Database query on each iteration

# DO (contrived example)
filtered = Person.objects.filter(  # Narrow down the QuerySet to only what you need
    first_name='Shallan',
    last_name='Davar',
    age_gte=0,
    age_lte=18,
)
lookup = {person.age: person for person in filtered}  # Evaluate the QuerySet and construct lookup
for age in range(18):
    person = lookup[age]  # No database query


# ---------------------------------------------------------------------------
# 6. Use iterator() to iterate through a very large QuerySet only once.
# ---------------------------------------------------------------------------

# Save memory by not caching anything
for person in Person.objects.iterator():
    # Some logic


# ---------------------------------------------------------------------------
# 7. Do work in the database rather than in Python.
# ---------------------------------------------------------------------------

## 7a. Use filter() and exclude()

# DON'T
for person in Person.objects.all():
    if person.age >= 18:
        # Do something

# DO
for person in Person.objects.filter(age__gte=18):
    # Do something

## 7b. Use F expressions

# DON'T
for person in Person.objects.all():
    person.age += 1
    person.save()

# DO
Person.objects.update(age=F('age') + 1)

## 7c. Do aggregation in the database, if possible

# DON'T
max_age = 0
for person in Person.objects.all():
    if person.age > max_age:
        max_age = person.age

# DO
max_age = Person.objects.all().aggregate(Max('age'))['age__max']


# ---------------------------------------------------------------------------
# 8. Use values() and values_list() to get only the things you need.
# ---------------------------------------------------------------------------

## 8a. Use values()

# DON'T
age_lookup = {
    person.name: person.age
    for person in Person.objects.all()
}

# DO
age_lookup = {
    person['name']: person['age']
    for person in Person.objects.values('name', 'age')
}

## 8b. Use values_list()

# DON'T
person_ids = [person.id for person in Person.objects.all()]

# DO
person_ids = Person.objects.values_list('id', flat=True)


# ---------------------------------------------------------------------------
# 9. Use defer() and only() when you know you won't need certain fields.
#
# * Use when you need a QuerySet instead of a list of dicts from values().
# * Really only useful to defer fields that require significant processing to convert to a python object.
# ---------------------------------------------------------------------------

## 9a. Use defer()

queryset = Person.objects.defer('age')  # Imagine age is computationally expensive
for person in queryset:
    print(person.id)
    print(person.name)

## 9b. Use only()

queryset = Person.objects.only('name')
for person in queryset:
    print(person.name)


# ---------------------------------------------------------------------------
# 10. Use count() and exists() when you don't need the contents of the QuerySet.
#
# * Caveat: Only use these when you don't need to evaluate the QuerySet.
# ---------------------------------------------------------------------------

## 10a. Use count()

# DON'T
count = len(Person.objects.all())  # Evaluates the entire queryset

# DO
count = Person.objects.count()  # Executes more efficient SQL to determine count

## 10b. Use exists()

# DON'T
exists = len(Person.objects.all()) > 0

# DO
exists = Person.objects.exists()


# ---------------------------------------------------------------------------
# 11. Use delete() and update() when possible.
# ---------------------------------------------------------------------------

## 11a. Use delete()

# DON'T
for person in Person.objects.all():
    person.delete()

# DO
Person.objects.all().delete()

## 11b. Use update()

# DON'T
for person in Person.objects.all():
    person.age = 0
    person.save()

# DO
Person.objects.update(age=0)


# ---------------------------------------------------------------------------
# 12. Use bulk_create() when possible.
#
# * Caveats: https://docs.djangoproject.com/en/2.1/ref/models/querysets/#django.db.models.query.QuerySet.bulk_create
# ---------------------------------------------------------------------------

# Bulk Create
names = ['Jeff', 'Beth', 'Tim']
creates = []
for name in names:
    creates.append(
        Person(name=name, age=0)
    )
Person.objects.bulk_create(creates)

# Bulk add to many-to-many fields
person = Person.objects.get(id=1)
person.jobs.add(job1, job2, job3)


# ---------------------------------------------------------------------------
# 13. Use foreign key values directly.
# ---------------------------------------------------------------------------

# DON'T
father_id = Person.objects.get(id=1).father.id  # Causes a needless database query

# DO
father_id = Person.objects.get(id=1).father_id  # The foreign key is already cached. No query
```

# Views

# Forms

# Templates

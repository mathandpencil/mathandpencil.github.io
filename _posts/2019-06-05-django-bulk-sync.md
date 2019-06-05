---
layout: post
title: Introducing django-bulk-sync
author: Scott Stafford
comments: true
offer_help: false
---

django-bulk-sync is a package to support the Django ORM that synthesizes the concepts of [bulk_create][bulk-create], [bulk_update][bulk-update], and delete into a single call: `bulk_sync()`.

## A Usage Scenario

Companies have zero or more Employees. You want to efficiently sync the names of all employees for a single `Company` from an import from that company, but some are added, updated, or removed.  The simple approach is inefficient -- read the import line by line, and:

For each of N records:

- SELECT to check for the employee's existence
- UPDATE if it exists, INSERT if it doesn't

Then figure out some way to identify what was missing and delete it.  As is so often the case, the speed of this process is controlled mostly by the number of queries run, and here it is about two queries for every record, and so O(N).

Instead, with `bulk_sync`, we can avoid the O(N) number of queries, and simplify the logic we have to write as well. 
	
## Example Usage

```python
from django.db.models import Q
from bulk_sync import bulk_sync

new_models = []
for line in company_import_file:
	# The `.id` (or `.pk`) field should not be set. Instead, `key_fields` 
	# tells it how to match.
	e = Employee(name=line['name'], phone_number=line['phone_number'], ...)
	new_models.append(e)

# `filters` controls the subset of objects considered when deciding to 
# update or delete.
filters = Q(company_id=501)  
# `key_fields` matches an existing object if all `key_fields` are equal.
key_fields = ('name', )  
ret = bulk_sync(
        new_models=new_models,
        filters=filters,
        key_fields=key_fields)

print("Results of bulk_sync: "
      "{created} created, {updated} updated, {deleted} deleted."
      		.format(**ret['stats']))
```

Under the hood, it will fetch the primary keys of all filtered objects, then atomically call `bulk_create`, `bulk_update`, and a single queryset `delete()` call, to correctly and efficiently update all fields of all employees for the filtered Company, using `name` to match properly. 

## Argument Reference

`def bulk_sync(new_models, key_fields, filters, batch_size=None):`
Combine bulk create, update, and delete.  Make the DB match a set of in-memory objects.
- `new_models`: An iterable of Django ORM `Model` objects that you want stored in the database. They may or may not have `id` set, but you should not have already called `save()` on them.
- `key_fields`: Identifying attribute name(s) to match up `new_models` items with database rows.  If a foreign key is being used as a key field, be sure to pass the `fieldname_id` rather than the `fieldname`.
- `filters`: Q() filters specifying the subset of the database to work in.
- `batch_size`: passes through to Django `bulk_create.batch_size` and `bulk_update.batch_size`, and controls how many objects are created/updated per SQL query.

`def bulk_compare(old_models, new_models, key_fields, ignore_fields=None):`
Compare two sets of models by `key_fields`.
- `old_models`: Iterable of Django ORM objects to compare.
- `new_models`: Iterable of Django ORM objects to compare.
- `key_fields`: Identifying attribute name(s) to match up `new_models` items with database rows.  If a foreign key
        is being used as a key field, be sure to pass the `fieldname_id` rather than the `fieldname`.
- `ignore_fields`: (optional) If set, provide field names that should not be considered when comparing objects.
- Returns dict of: ```
    {
        'added': list of all added objects.
        'unchanged': list of all unchanged objects.
        'updated': list of all updated objects.
        'updated_details': dict of {obj: {field_name: (old_value, new_value)}} for all changed fields in each updated object.
        'removed': list of all removed objects.
    } ```

## Other techniques

A simpler overall approach might be to simply do this:

```
with transaction.atomic():
    MyObject.objects.filter(filters).delete()
    MyObject.objects.bulk_create(new_models)
```

In some cases, this may be faster (as it skips the initial fetch). However, the `bulk_sync` approach has a number of advantages:
- Primary key of existing objects will not change unnecessarily: This may be critical if you use it as a foreign key.
- You cannot find statistics on how many were created/updated/deleted.
- `auto_now_add` fields will be changed, incorrectly.

## Installation and Quick Start

The package is available on pip as [django-bulk-sync][django-bulk-sync].  Run:

`pip install django-bulk-sync`

then import via:

`from bulk_sync import bulk_sync`

And use as in the example above.


[bulk-create]: https://docs.djangoproject.com/en/dev/ref/models/querysets/#bulk-create
[bulk-update]: https://docs.djangoproject.com/en/dev/ref/models/querysets/#bulk-update
[django-bulk-sync]: https://pypi.org/project/django-bulk-sync/



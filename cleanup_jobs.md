## Old Way

slow : 7 hours to cleanup 1 Million jobs

```
for j in jobs:
    j.delete()
```

## Bulk delete doesn't work well either

`Jobs.objects.filter(..).delete()`

django's `Collector` class, which does the actual deletion, will attempt to load all objects of the queryset. This uses way too much memory and is slow.

## Optimizing Collector

Don't load the objects, rather just keep the querysets

example instead of

`parent_objs = [getattr(obj, ptr.name) for obj in new_objs]`

we do

`parent_objs = ptr.objects.filter(pk__in = new_objs.values_list('pk', flat=True))`


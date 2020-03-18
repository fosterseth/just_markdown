### OLD WAY

slow : 7 hours to cleanup 1 Million jobs

```
for j in jobs:
    j.delete()
```

### Bulk delete doesn't work well either

`Jobs.objects.filter(..).delete()`

django's `Collector` class, which does the actual deletion, will attempt to load all objects of the queryset. This uses way too much memory and is slow.

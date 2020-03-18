### OLD WAY

slow : 7 hours to cleanup 1 Million jobs

```
for j in jobs:
    j.delete()
'''

### Bulk delete doesn't work well either

`Jobs.objects.filter(..).delete()`

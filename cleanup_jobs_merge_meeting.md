### SUMMARY

awx/main/management/commands/cleanup_jobs.py

This system job is used to clean up older jobs in the database. Clearing up the database is essential to keep certain parts of our api responsive.

The old cleanup jobs was pretty slow. It would delete one object at a time.

```
for j in jobs:
    j.delete()
```

Removing the loop and doing it batch style also has problems.

`Job.objects.filter(..)[0:10000].delete()`

This method will attempt to load in all objects into memory first before deletion. Also, it will send pre- and post- delete signals on each object.



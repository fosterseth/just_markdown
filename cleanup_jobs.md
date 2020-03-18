## Old Way

slow : 7 hours to cleanup 1 Million jobs

```
for j in jobs:
    j.delete()
```

## a Bulk delete work well

`Jobs.objects.filter(..).delete()`

django's `Collector` class, which does the actual deletion, will attempt to load all objects of the queryset. This uses way too much memory and is slow.

## deleting at SQL level doesn't work either

Because of foreign key contraints on `main_job` and nearly every other table - we cannot simply call delete at the SQL level.

## Optimizing Collector
### 1 Don't load the objects, rather just work with querysets

Anywhere that querysets were being converted to objects, I kept them as querysets

![](add_old.png)

becomes

![](add_new.png)

Anywhere that assumed a list of objects, I changed to work with querysets

`parent_objs = [getattr(obj, ptr.name) for obj in new_objs]`

becomes

`parent_objs = ptr.objects.filter(pk__in = new_objs.values_list('pk', flat=True))`

### 2 Don't do pre- and post- signaling

Profiling results showed that a lot of the time was spent during `send`. We can greatly speed up the deletion by skipping these signals.

![](send_collect.png)

## Speed improvements

6 minutes to remove 1 million jobs

Note, this fast delete method can be used for any model, not just Job

## Comparing Old Collector to New Collector

`Collector.collect()` returns a dictionary of objects to be deleted
`Collector.sort()` sorts this dictionary in order of deletion. This avoids foreign key contraint errors. For example, `Job` must be deleted before `UnifiedJob`.

### OLD
```
OrderedDict([(awx.main.models.unified_jobs.UnifiedJob_credentials,
              {<UnifiedJob_credentials: UnifiedJob_credentials object (416)>,
               <UnifiedJob_credentials: UnifiedJob_credentials object (417)>,
               <UnifiedJob_credentials: UnifiedJob_credentials object (415)>}),
             (awx.main.models.jobs.JobLaunchConfig,
              {<JobLaunchConfig: job launch config-416>,
               <JobLaunchConfig: job launch config-417>,
               <JobLaunchConfig: job launch config-418>}),
             (awx.main.models.jobs.JobHostSummary,
              {<JobHostSummary: localhost changed=0 dark=0 failures=0 ignored=0 ok=2 processed=1 rescued=0 skipped=0>,
               <JobHostSummary: localhost changed=0 dark=0 failures=0 ignored=0 ok=2 processed=1 rescued=0 skipped=0>,
               <JobHostSummary: localhost changed=0 dark=0 failures=0 ignored=0 ok=2 processed=1 rescued=0 skipped=0>}),
             (awx.main.models.jobs.Job,
              {<Job: 2020-03-18 20:20:34.699446+00:00-416-successful>,
               <Job: 2020-03-18 20:20:43.163874+00:00-417-successful>,
               <Job: 2020-03-18 20:20:51.594965+00:00-418-successful>}),
             (awx.main.models.unified_jobs.UnifiedJob,
              {<UnifiedJob: 2020-03-18 20:20:34.699446+00:00-416-successful>,
               <UnifiedJob: 2020-03-18 20:20:43.163874+00:00-417-successful>,
               <UnifiedJob: 2020-03-18 20:20:51.594965+00:00-418-successful>})])
```

### NEW

```
OrderedDict([(awx.main.models.unified_jobs.UnifiedJob_credentials,
              [<QuerySet [<UnifiedJob_credentials: UnifiedJob_credentials object (415)>, <UnifiedJob_credentials: UnifiedJob_credentials object (416)>, <UnifiedJob_credentials: UnifiedJob_credentials object (417)>]>]),
             (awx.main.models.jobs.JobLaunchConfig,
              [<QuerySet [<JobLaunchConfig: job launch config-416>, <JobLaunchConfig: job launch config-417>, <JobLaunchConfig: job launch config-418>]>]),
             (awx.main.models.jobs.JobHostSummary,
              [<QuerySet [<JobHostSummary: localhost changed=0 dark=0 failures=0 ignored=0 ok=2 processed=1 rescued=0 skipped=0>, <JobHostSummary: localhost changed=0 dark=0 failures=0 ignored=0 ok=2 processed=1 rescued=0 skipped=0>, <JobHostSummary: localhost changed=0 dark=0 failures=0 ignored=0 ok=2 processed=1 rescued=0 skipped=0>]>]),
             (awx.main.models.jobs.Job,
              [<PolymorphicQuerySet [<Job: 2020-03-18 20:20:34.699446+00:00-416-successful>, <Job: 2020-03-18 20:20:43.163874+00:00-417-successful>, <Job: 2020-03-18 20:20:51.594965+00:00-418-successful>]>]),
             (awx.main.models.unified_jobs.UnifiedJob,
              [<PolymorphicQuerySet [<UnifiedJob: 2020-03-18 20:20:34.699446+00:00-416-successful>, <UnifiedJob: 2020-03-18 20:20:43.163874+00:00-417-successful>, <UnifiedJob: 2020-03-18 20:20:51.594965+00:00-418-successful>]>])])
```

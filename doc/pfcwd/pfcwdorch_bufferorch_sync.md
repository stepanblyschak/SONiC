# PFC Watch Dog & BufferOrch synchronization

## Problem statement

PFC WD action handler has two implementations:
    - ACL based approach
    - Zero buffer approach

Below is the flow described for zero buffer approach:

PFC WD action handler flow (storm):
    - Copy original PG/queue profiles
    - Set zero buffer profile on PG/queue

PFC WD action handler flow (restore):
    - Set copied original PG/queue profiles


Since, there are two Orchs - PfcWdOrch and BufferOrch which can change PG/queue configuration it may lead to a problem in specific scenarios:

#### Warm reboot flow

If **orchagent** starts before **buffermgrd** and BufferOrch gets initialized before **buffermgrd**, 
**buffermgrd** will send initial update.
In case storm started before warm reboot and is ongoing during warm start and after orchagent reconciled, **buffermgrd** initial update will cause BufferOrch to override zero buffer profile on PG.

#### Buffer configuration changes when storm is ongoing

Similar will happen when user is changing configuration during storm; moreover since PFC WD action handler saved the original profile, after storm is finished, original profile will be restored, so DB and HW will be out of sync.

## Quick fix

In **buffermgrd** add a condition - if we are trying to set PG profile that is already set according to DB, skip setting the same profile:

```c++
+    string current_profile_ref;
+    if (m_cfgBufferPgTable.hget(buffer_pg_key, "profile", current_profile_ref) &&
+           current_profile_ref == profile_ref)
+    {
+        SWSS_LOG_NOTICE("Already set correct PG profile");
+        // already set correct profile
+        return task_process_status::task_success;
+    }
```

In this case, buffermgrd will not set PG profile if it is already set to desired.
However, this will not fix scenario when buffer configuration is changed during storm.


## Solution proposal

Since there are two configuration producers for same objects, there should be a syncronization mechanism.
The idea is that PFC WD action handler should protect PG/queue profile from being changed until storm is restored.

Before PFC WD action handler sets zero buffer profile it notifies BufferOrch that PG/queue should be protected from being changed.

If BufferOrch receives update on PG/queue that is protected BufferOrch will add this configuration to pending list.

Once BufferOrch is notified from PFC WD action handler that PG/queue is unlocked, BufferOrch process pending list.








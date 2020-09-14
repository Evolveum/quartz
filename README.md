
## About
Contains original Quartz Scheduler code with some functionality provided by Evolveum.

The main functionality that was added allows execution nodes to specify limits on the number of threads 
for so called _execution groups_. It is a partial implementation of [issue #175](https://github.com/quartz-scheduler/quartz/issues/175).

## Execution groups
An execution group is a tag optionally attached to a trigger that characterizes the execution requirements 
or characteristics of a job related to the given trigger. Examples of execution groups might be "batch jobs", "large RAM", 
"lot of CPU" and so on. So, each node can declare things like:

- out of my available threads, at most 2 can run triggers of "lot of CPU" execution group,
- out of my available threads, only triggers with no execution group can run,
- out of my available threads, only triggers with execution groups "A" and "B" can run,
- out of my available threads, at most 10 triggers with no execution group can run, no triggers from "batch-jobs" group, and at most 5 triggers from any other execution group. 

(It is expected that all triggers for a given job would share the same execution group.)

### Setup using properties
Execution limits can be set up using the following properties (showing the last scenario described above):

```
org.quartz.executionLimit.null=10
org.quartz.executionLimit.batch-jobs=0
org.quartz.executionLimit.*=5
```

Other available features:

```
org.quartz.executionLimit._=10              # underscore is an alternative way of specifying "no execution group"
org.quartz.executionLimit.grp1=unlimited    # a way of saying "there is no limit for grp1"
org.quartz.executionLimit.grp1=none         # another way of saying the same
org.quartz.executionLimit.grp1=null         # yet another way
```

### Setup using Scheduler API
Alternative way of setting these limits is by calling: 

```
scheduler.setExecutionLimits(Map<String, Integer> limits)
``` 

where limits is a map where keys are referring to execution groups (including null) and values provide information 
on limitations (null meaning "no limit", 0 meaning "cannot be run here", N > 0 meaning "can run use most N threads").

So, the following will set the last scenario described above:

```java
Map<String, Integer> limits = new HashMap<>();
limits.put(null, 10);
limits.put("batch-jobs", 0); 
limits.put(Scheduler.LIMIT_FOR_OTHER_GROUPS, 5);
scheduler.setExecutionLimits(limits);
``` 

### Setting execution groups on triggers
Execution group for a trigger can be set either by calling _trigger.setExecutionGroup_ method or using TriggerBuilder:
```java
Trigger t1 = TriggerBuilder.newTrigger()
        .withIdentity("t1")
        .forJob(job)
        .executionGroup("grp1")
        .startNow()
        .build();
``` 

### Required database schema changes
In order to hold information about triggers' execution groups there are two new columns:
- EXECUTION_GROUP in QRTZ_TRIGGERS
- EXECUTION_GROUP in QRTZ_FIRED_TRIGGERS

To make migration from Quartz 2.3.0 easy (at the cost of a bit more complex code) they are nullable. Their data type is the
same as the data type of TRIGGER_GROUP and JOB_GROUP columns except for nullability. Updated DB scripts are present in 
[the source code](https://github.com/Evolveum/quartz/tree/evolveum-master/quartz-core/src/main/resources/org/quartz/impl/jdbcjobstore). 

Upgrade scripts are specific for individual databases but in general they will look like this (e.g. for PostgreSQL)
```
ALTER TABLE QRTZ_TRIGGERS ADD COLUMN EXECUTION_GROUP VARCHAR(200) NULL;
ALTER TABLE QRTZ_FIRED_TRIGGERS ADD COLUMN EXECUTION_GROUP VARCHAR(200) NULL;
```

## Additional information
_Execution groups_ feature is supported on JDBC and RAM job stores.

Current version of the code is 2.3.2.e1, which is based on Quartz 2.3.2.

This version requires Java 1.8 to successfully build and run the tests.  

Evolveum-specific development is carried out on "evolveum-master" branch. 

## Build instructions:

### To compile:
```
  %> ./mvnw install -DskipTests
```

Note:  the final Quartz jar is found under quartz/target 

### To build Quartz distribution kit:
```
  %> cd distribution
  %> ./mvnw package
```


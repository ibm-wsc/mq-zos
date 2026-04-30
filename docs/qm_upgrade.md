# Upgrading a Queue Manager on IBM MQ for z/OS

### Audience level
Some knowledge of MQ or z/OS

### Skillset
z/OS Systems Programming, MQ Administration

### Background
This lab walks through the high-level steps required to upgrade an existing IBM MQ for z/OS queue manager to a newer product level after the SMP/E installation work has already been completed.

In this example, the target release is IBM MQ 9.4.2 CD, so the product high-level qualifier used in the examples is `MQ942CD`.

This lab focuses on the queue manager upgrade activities themselves. It does not cover SMP/E installation, system-wide product deployment, or broader release planning activities.

### Overview of the exercise

In this lab, you will:

I. Stop the queue manager

II. Optionally rebind the queue sharing group environment

III. Update the queue manager started task procedures

IV. Update early code and product libraries where needed

V. Restart and validate the queue manager

### Prerequisites

Before you begin, make sure the following are available:

- The new IBM MQ for z/OS code has already been installed with SMP/E
- You know the high-level qualifier for the new release, for example `MQ942CD`
- You know the queue manager name you are upgrading, for example `ZQS1`
- You have authority to stop and start the queue manager and channel initiator
- You have authority to update `SYS1.PROCLIB` or the procedure library used by the queue manager
- You know which PARMLIB members, APF definitions, and LPA definitions are active in your environment
- You have a rollback plan in case the queue manager fails to start after the update
- If the queue manager is part of a queue sharing group, you understand whether any Db2 bind or rebind work is required in your environment

> Note: This lab uses `ZQS1` as an example queue manager name and `MQ942CD` as an example MQ high-level qualifier. Replace those values with the ones used in your environment.

### Lab begin

#### I. Stop the queue manager

1. From the ISPF main menu, enter `SDSF` and press Enter.

2. In SDSF, enter `/` and press Enter to access the MVS command line through the System Command Extension.

3. Stop the channel initiator:

   ```text
   ZQS1 STOP CHINIT
   ```

4. Stop the queue manager:

   ```text
   ZQS1 STOP QMGR
   ```

5. Confirm in SDSF or the queue manager logs that both address spaces have stopped cleanly before proceeding.

#### II. Optional: rebind queue sharing group objects

6. If the queue manager is part of a queue sharing group, review whether any Db2 packages or plans need to be rebound for the new MQ release.

7. Follow your site's documented queue sharing group upgrade procedure for any required bind or rebind activities.

8. If your environment does not require this step, continue to the next section.

> Note: This step is environment-dependent. Some labs may not require any queue sharing group rebind activity at all.

#### III. Update MSTR and CHIN started tasks

9. Locate the started task procedures used by the queue manager, typically in `SYS1.PROCLIB` or a site-specific procedure library.

10. Find the queue manager `MSTR` and `CHIN` procedures. For `ZQS1`, these are commonly named:

   - `ZQS1MSTR`
   - `ZQS1CHIN`

11. Edit the `MSTR` procedure and update the MQ library references so that they point to the new release libraries, for example changing older MQ libraries to `MQ942CD...` libraries.

12. Edit the `CHIN` procedure and make the same update to the MQ library references.

13. Review the procedures carefully for references such as:
   - load libraries
   - language environment libraries
   - SSL or security-related libraries
   - exit libraries
   - any site-specific overrides

14. Save the updated procedures.

> Note: If you use a site-specific started task library instead of `SYS1.PROCLIB`, update the procedures there instead.

#### IV. Update early code and product libraries

15. Review whether the new MQ release libraries are correctly APF-authorized.

16. Review whether any MQ modules in the LPA or LPALST need to be updated for the new release.

17. Review whether the subsystem definition for the queue manager still points to the correct initialization routines and load libraries.

18. If your environment uses dynamic definitions, verify that the required MQ modules are available from the new release libraries.

19. If your environment uses static definitions in `SYS1.PARMLIB`, review the relevant members, such as:
   - `LPALSTxx`
   - `PROGxx`
   - `IEFSSNxx`

20. Confirm that the active definitions used at IPL time are the ones you expect for the upgraded release.

> Note: The exact updates required here depend heavily on how your z/OS environment is configured. In some labs, this work may already have been completed as part of the product installation setup.

#### V. Restart and validate the queue manager

21. Start the queue manager `MSTR` address space. For example:

   ```text
   /START ZQS1MSTR
   ```

22. After the queue manager initializes successfully, start the channel initiator:

   ```text
   /START ZQS1CHIN
   ```

23. Review the JES log, MSTR job log, and CHIN job log for startup messages.

24. Confirm that the queue manager starts successfully on the new release.

25. Use MQ commands to validate that the queue manager is responding normally. For example, verify that basic display commands work and that queue manager attributes can be queried.

26. If client connectivity is used in your environment, confirm that MQ Explorer or another client can still connect successfully.

27. If channels are used in your environment, confirm that the channel initiator is healthy and that expected channels can start.

28. If the queue manager is part of a queue sharing group, verify that it rejoins the group correctly.

### Validation checklist

Before considering the upgrade complete, verify the following:

- The old queue manager and channel initiator were stopped cleanly
- The `MSTR` and `CHIN` procedures were updated to reference the new MQ release libraries
- Any required APF, LPA, or subsystem updates were reviewed and applied
- The queue manager started successfully on the new release
- The channel initiator started successfully
- Basic MQ administration commands work as expected
- Client or channel connectivity still works if applicable
- Queue sharing group behavior is normal if applicable

### Troubleshooting

If the queue manager does not start correctly after the update, check the following:

- The `MSTR` and `CHIN` procedures reference the correct new release libraries
- Required MQ libraries are APF-authorized
- Any LPA or LPALST changes for early code were made correctly
- The subsystem definition still points to the correct MQ initialization routines
- Exit libraries and LE libraries are still available and correctly referenced
- The queue manager was stopped cleanly before the upgrade
- Any queue sharing group bind or rebind work required by your environment was completed
- Site-specific security, SSL, or exit configuration was not accidentally changed

Common symptoms and causes:

- `MSTR` fails immediately at startup: wrong load library references, missing APF authorization, or missing early code updates
- `CHIN` fails to start: incorrect procedure changes, missing channel-related libraries, or unresolved security or exit library references
- Queue manager starts but clients cannot connect: listener, channel, security, or TLS configuration needs review
- Queue sharing group problems after startup: required Db2 or QSG-specific post-upgrade tasks may be incomplete

### Rollback considerations

If the queue manager does not start successfully on the new release and you need to back out the change:

1. Stop any partially started address spaces.
2. Restore the previous `MSTR` and `CHIN` procedures.
3. Restore any earlier APF, LPA, or subsystem definitions if they were changed.
4. Restart the queue manager using the previous release libraries.
5. Review logs and correct the upgrade issue before attempting the upgrade again.

### Summary

In this lab, you stopped an existing queue manager, updated its runtime procedures to point to a newer IBM MQ for z/OS release, reviewed any early code dependencies, restarted the queue manager, and validated the upgrade.

This is the core operational workflow for a queue manager upgrade after the product installation work has already been completed.

# Configuring RACF Resources for MQ Security

#### **Audience level**
Some knowledge of MQ or z/OS

#### **Skillset**
MQ Administration, z/OS systems programming

#### **Background**

### Exercise objectives

The objective of this exercise is to gain experience protecting IBM MQ queues, MQ commands, and queue manager connections by using RACF. In this lab, you will enable RACF protection for MQ resources and then grant appropriate access to different sets of users, both local and remote.

For this exercise, you will use the data set `ZQS1.SECURITY.JCL`. The following members are referenced in the lab:

- `ADDGROUP` - Defines `MQSTC`, `CICSSTC`, and `MQUSERS`, and connects users to those groups
- `MQCMDS`
- `MQCONN`
- `MQQUEUE`

### Important notes

Before you begin, review the RACF terms used in this lab:

1. `RDEFINE` - Adds a profile for a resource to the RACF database so that access can be controlled
2. `PERMIT` - Maintains the list of users and groups authorized to access a resource

## Lab begin

#### I. Enable security checking on queue manager ZQS1

External security checking is enabled or disabled for a specific queue manager based on the presence or absence of certain `MQADMIN` RACF resources during queue manager initialization. During startup, the queue manager uses its name, for example `ZQS1`, to look for queue-manager-specific RACF resources. If a disabling resource is defined, the corresponding security check is turned off.

For example, if the `MQADMIN` resource `ZQS1.NO.TOPIC.CHECKS` is defined, external security checking for topics is disabled. If `ZQS1.NO.QUEUE.CHECKS` is defined, external security checking for queue access is disabled.

In this lab, you will move from the current lab state toward a RACF-protected configuration by identifying what is disabled, removing the global subsystem bypass, and defining the RACF profiles needed for basic queue manager operation.

1. Log on to TSO/ISPF using your assigned z/OS credentials.

2. From option `6` on the ISPF main menu, use the RACF command `LISTGROUP` (or `LG`) to list users currently connected to the groups used in this lab. For example:

   ```text
   LG SYS1
   ```

   You may see output similar to the following:

   ```text
   NO MODEL DATA SET
   TERMUACC
   SUBGROUP(S)= DFSGRP   ZFSGRP   ZOSV210  @PL      ABJ      AIO
               AOK      AOP      APK      ASM      ATX      AUP
               BDT1     B8R      CAZ      CBC      CDS      CEE
               CFZ      CKL      CKR      CPAC     CSD      CSF
               C4R      DGA      DIT      EEL      ELA      EMS
               EOX      EOY      EPH      EQAW     EQQ      EUVF
               FFST     FMN      GDDM     GIM      GLD      GSK
               HAP      HVT      IBMZ     ICQ      IDI      IGY
               IMW      ING      IOA      IOE      IPV      ISF
   ....
   USER(S)=      ACCESS=      ACCESS COUNT=      UNIVERSAL ACCESS=
       IBMUSER       JOIN          000004               READ
           CONNECT ATTRIBUTES=NONE
           REVOKE DATE=NONE                 RESUME DATE=NONE
       SYSPROG       JOIN          014895               NONE
           CONNECT ATTRIBUTES=NONE
           REVOKE DATE=NONE                 RESUME DATE=NONE
   ```

   > Tech tip: `LISTUSER userid` is also useful when you want to inspect the permissions of a specific user. For example: `LISTUSER USER1`

3. If needed, review or submit the `ADDGROUP` member in `ZQS1.SECURITY.JCL` so that the required groups and user connections exist before you define MQ resource profiles.

4. To determine which `MQADMIN` resources are currently defined for `ZQS1`, use:

   ```text
   SEARCH CLASS(MQADMIN) FILTER(ZQS1.**)
   ```

   The results should look similar to this:

   ```text
   ZQS1.NO.ALTERNATE.USER.CHECKS
   ZQS1.NO.CMD.RESC.CHECKS
   ZQS1.NO.CONTEXT.CHECKS
   ZQS1.NO.NLIST.CHECKS
   ZQS1.NO.PROCESS.CHECKS
   ZQS1.NO.SUBSYS.SECURITY
   ZQS1.NO.TOPIC.CHECKS
   ZQS1.RESLEVEL
   ```

   > Tech tip: RACF resources associated with a specific queue manager use the queue manager name as a prefix. This allows `ZQS1.SYSTEM.DEFAULT.LOCAL.QUEUE` and `ZQS2.SYSTEM.DEFAULT.LOCAL.QUEUE` to have different protection.

5. Review the list carefully. Some `MQADMIN` resources disable specific external security checks. One resource in particular disables all external security checking, regardless of the others:

   ```text
   ZQS1.NO.SUBSYS.SECURITY
   ```

   If this profile exists, subsystem security is currently disabled. You would typically see a startup message like:

   ```text
   CSQH021I ZQS1 CSQHINIT SUBSYSTEM security switch set
   OFF, profile 'ZQS1.NO.SUBSYS.SECURITY' found
   ```

6. To enable external security checking for the next restart of the queue manager, delete this resource:

   ```text
   RDELETE MQADMIN ZQS1.NO.SUBSYS.SECURITY
   ```

7. Refresh the RACF in-storage profiles:

   ```text
   SETROPTS RACLIST(MQADMIN) REFRESH
   ```

8. Shut down the `ZQS1` queue manager from SDSF:

   ```text
   /ZQS1 STOP QMGR
   ```

   > Tech tip: To disable subsystem security again, redefine the resource:
   >
   > ```text
   > RDEFINE MQADMIN ZQS1.NO.SUBSYS.SECURITY OWNER(SYS1)
   > ```

   > Tech tip: You may receive message `ICH14070I SETROPTS RACLIST REFRESH had no effect on class MQADMIN`. Not every RACF configuration requires a refresh of in-storage profiles, but refreshing after changes is still a good habit.

9. Before restarting the queue manager, define some basic MQ RACF resources. Select member `MQCMDS` in data set `ZQS1.SECURITY.JCL`. This member defines `MQCMDS` RACF resources for common MQ commands and grants suitable access to administrative groups.

10. Submit `MQCMDS` and verify that it completes with a condition code of zero.

    > Tech tip: The `SEARCH` and `EXEC` commands at the beginning of the job remove any existing `MQCMDS` profiles for queue manager `ZQS1` before redefining them.

    Example content includes:

    ```text
    RDEFINE MQCMDS ZQS1.DEFINE.** OWNER(SYS1)

    PERMIT ZQS1.DEFINE.** CLASS(MQCMDS) RESET

    PERMIT ZQS1.DEFINE.** CLASS(MQCMDS) ID(MQSTC,MQSYSP) ACC(ALTER)

    RDEFINE MQCMDS ZQS1.DELETE.** OWNER(SYS1)

    PERMIT ZQS1.DELETE.** CLASS(MQCMDS) RESET

    PERMIT ZQS1.DELETE.** CLASS(MQCMDS) ID(MQSTC,MQSYSP) ACC(ALTER)

    RDEFINE MQCMDS ZQS1.DISPLAY.** OWNER(SYS1)

    PERMIT ZQS1.DISPLAY.** CLASS(MQCMDS) RESET

    PERMIT ZQS1.DISPLAY.** CLASS(MQCMDS) ID(MQSTC,MQUSERS) ACC(READ)

    RDEFINE MQCMDS ZQS1.REFRESH.** OWNER(SYS1)

    PERMIT ZQS1.REFRESH.** CLASS(MQCMDS) RESET

    PERMIT ZQS1.REFRESH.** CLASS(MQCMDS) ID(MQSTC,MQSYSP) ACC(ALTER)

    RDEFINE MQCMDS ZQS1.START.** OWNER(SYS1)

    PERMIT ZQS1.START.** CLASS(MQCMDS) RESET

    PERMIT ZQS1.START.** CLASS(MQCMDS) ID(MQSTC,MQSYSP) ACC(CONTROL)

    RDEFINE MQCMDS ZQS1.STOP.** OWNER(SYS1)

    PERMIT ZQS1.STOP.** CLASS(MQCMDS) RESET

    PERMIT ZQS1.STOP.** CLASS(MQCMDS) ID(MQSTC,MQSYSP) ACC(CONTROL)

    RDEFINE MQCMDS ZQS1.SET.** OWNER(SYS1)

    PERMIT ZQS1.SET.** CLASS(MQCMDS) RESET

    PERMIT ZQS1.SET.** CLASS(MQCMDS) ID(MQSTC,MQSYSP) ACC(CONTROL)

    RDEFINE MQCMDS ZQS1.CLEAR.** OWNER(SYS1)

    PERMIT ZQS1.CLEAR.** CLASS(MQCMDS) RESET

    PERMIT ZQS1.CLEAR.** CLASS(MQCMDS) ID(MQSTC,MQSYSP) ACC(ALTER)

    RDEFINE MQCMDS ZQS1.** OWNER(SYS1)

    PERMIT ZQS1.** CLASS(MQCMDS) RESET

    PERMIT ZQS1.** CLASS(MQCMDS) ID(MQSTC,MQSYSP) ACC(READ)

    SETROPTS RACLIST(MQCMDS) REFRESH
    ```

11. Select member `MQCONN` in `ZQS1.SECURITY.JCL`. This member defines `MQCONN` RACF resources required for access to the queue manager from different environments such as batch, CICS, and channel initiator processing.

12. Submit `MQCONN` and verify that it completes with a condition code of zero.

13. Select member `MQQUEUE` in `ZQS1.SECURITY.JCL`. This member defines `MQQUEUE` RACF resources required for access to system-related queues.

    Example content includes:

    ```text
    RDEFINE MQCONN ZQS1.BATCH OWNER(SYS1)
    PERMIT ZQS1.BATCH CLASS(MQCONN) RESET
    PERMIT ZQS1.BATCH CLASS(MQCONN) ID(MQSTC,MQUSERS) ACC(READ)

    RDEFINE MQCONN ZQS1.CHIN OWNER(SYS1)
    PERMIT ZQS1.CHIN CLASS(MQCONN) RESET
    PERMIT ZQS1.CHIN CLASS(MQCONN) ID(MQSTC) ACC(READ)

    RDEFINE MQCONN ZQS1.CICS OWNER(SYS1)
    PERMIT ZQS1.CICS CLASS(MQCONN) RESET
    PERMIT ZQS1.CICS CLASS(MQCONN) ID(CICSSTC) ACC(READ)

    SETROPTS RACLIST(MQCONN) REFRESH

    RDEFINE MQQUEUE ZQS1.** OWNER(SYS1)
    PERMIT ZQS1.** CLASS(MQQUEUE) RESET
    PERMIT ZQS1.** CLASS(MQQUEUE) ID(MQSTC) ACC(READ)

    RDEFINE MQQUEUE ZQS1.SYSTEM.** OWNER(SYS1)
    PERMIT ZQS1.SYSTEM.** CLASS(MQQUEUE) RESET
    PERMIT ZQS1.SYSTEM.** CLASS(MQQUEUE) ID(MQSTC) ACC(UPDATE)
    PERMIT ZQS1.SYSTEM.** CLASS(MQQUEUE) ID(MQUSERS) ACC(READ)

    RDEFINE MQQUEUE ZQS1.SYSTEM.CLUSTER.COMMAND.QUEUE OWNER(SYS1)
    PERMIT ZQS1.SYSTEM.CLUSTER.COMMAND.QUEUE CLASS(MQQUEUE) RESET
    PERMIT ZQS1.SYSTEM.CLUSTER.COMMAND.QUEUE CLASS(MQQUEUE) +
     ID(MQSTC) ACC(ALTER)
    PERMIT ZQS1.SYSTEM.CLUSTER.COMMAND.QUEUE CLASS(MQQUEUE) +
     ID(MQUSERS) ACC(UPDATE)

    RDEFINE MQQUEUE ZQS1.SYSTEM.BROKER.** OWNER(SYS1)
    PERMIT ZQS1.SYSTEM.BROKER.** CLASS(MQQUEUE) RESET
    PERMIT ZQS1.SYSTEM.BROKER.** CLASS(MQQUEUE) +
     ID(MQSTC) ACC(ALTER)

    RDEFINE MQQUEUE ZQS1.AMQ.MQEXPLORER.** OWNER(SYS1)
    PERMIT ZQS1.AMQ.MQEXPLORER.** CLASS(MQQUEUE) RESET
    PERMIT ZQS1.AMQ.MQEXPLORER.** CLASS(MQQUEUE) +
     ID(MQSTC,MQUSERS) ACC(UPDATE)

    RDEFINE MQQUEUE ZQS1.SYSTEM.COMMAND.INPUT OWNER(SYS1)
    PERMIT ZQS1.SYSTEM.COMMAND.INPUT CLASS(MQQUEUE) RESET
    PERMIT ZQS1.SYSTEM.COMMAND.INPUT CLASS(MQQUEUE) +
     ID(MQSTC,MQUSERS) ACC(UPDATE)

    RDEFINE MQQUEUE ZQS1.SYSTEM.CSQUTIL.** OWNER(SYS1)
    PERMIT ZQS1.SYSTEM.CSQUTIL.** CLASS(MQQUEUE) RESET
    PERMIT ZQS1.SYSTEM.CSQUTIL.** +
     CLASS(MQQUEUE) ID(MQSTC,MQUSERS) ACC(UPDATE)
    ```

14. Submit `MQQUEUE` and verify that it completes with a condition code of zero. This completes the base RACF configuration required to start a basic queue manager with protected resources.

15. Restart the queue manager from SDSF. For example:

    ```text
    /START ZQS1MSTR
    ```

    If required in your environment, also start the channel initiator:

    ```text
    /START ZQS1CHIN
    ```

16. Review the queue manager startup messages. Confirm that the queue manager initializes successfully and that you no longer see the message indicating subsystem security is turned off because `ZQS1.NO.SUBSYS.SECURITY` was found.

17. Validate the configuration with a combination of positive and negative tests. At a minimum, confirm the following:

    - The queue manager starts successfully
    - Authorized users can connect and issue allowed commands
    - Unauthorized users are denied access to protected MQ resources
    - MQ Explorer, batch, or CICS access behaves according to the profiles you defined

18. Perform a positive command authorization test using a user in an authorized administrative group. For example, verify that an authorized user can issue display commands against `ZQS1`.

19. Perform a negative command authorization test using a user that is not permitted to issue administrative commands. Confirm that the command is rejected by RACF or by MQ external security checking.

20. Perform a positive connection test for an allowed access path. For example:
    - A batch user in `MQUSERS` can connect through the `MQCONN` profile `ZQS1.BATCH`
    - A CICS started task in `CICSSTC` can connect through `ZQS1.CICS`

21. Perform a negative connection test using a user or address space that is not covered by the required `MQCONN` profile. Confirm that the connection is denied.

22. Perform a queue authorization test:
    - Verify that an authorized user can access a queue covered by an allowed `MQQUEUE` profile
    - Verify that an unauthorized user cannot update a protected system queue

23. Expected failures will vary by access path and command, but you should expect unauthorized actions to fail with RACF or MQ security errors rather than succeeding silently. Record the message IDs you see in your environment for future reference.

24. If access does not behave as expected, review:
    - group membership for the test user
    - the active `MQADMIN`, `MQCMDS`, `MQCONN`, and `MQQUEUE` profiles
    - whether `SETROPTS RACLIST(... ) REFRESH` was issued after changes
    - whether the queue manager was restarted after changing subsystem security behavior

### Cleanup or rollback

If you want to roll back the lab to the earlier unsecured state, use the following approach:

1. Stop the queue manager:

   ```text
   /ZQS1 STOP QMGR
   ```

2. Redefine the subsystem bypass profile:

   ```text
   RDEFINE MQADMIN ZQS1.NO.SUBSYS.SECURITY OWNER(SYS1)
   ```

3. Refresh the in-storage profiles:

   ```text
   SETROPTS RACLIST(MQADMIN) REFRESH
   ```

4. Optionally remove or back out the `MQCMDS`, `MQCONN`, and `MQQUEUE` definitions you added for the lab, according to your site's RACF standards.

5. Restart the queue manager and confirm that subsystem security is again disabled.

### Additional MQQUEUE examples

The following examples are useful as extensions to the base lab. They provide additional protection for specific system and application queues.

```text
RDEFINE MQQUEUE ZQS1.SYSTEM.MQEXPLORER.REPLY.MODEL OWNER(SYS1)
PERMIT ZQS1.SYSTEM.MQEXPLORER.REPLY.MODEL CLASS(MQQUEUE) RESET
PERMIT ZQS1.SYSTEM.MQEXPLORER.REPLY.MODEL +
 CLASS(MQQUEUE) ID(MQSTC,MQUSERS) ACC(UPDATE)

RDEFINE MQQUEUE ZQS1.SYSTEM.PROTECTION.POLICY.QUEUE OWNER(SYS1)
PERMIT ZQS1.SYSTEM.PROTECTION.POLICY.QUEUE CLASS(MQQUEUE) RESET
PERMIT ZQS1.SYSTEM.PROTECTION.POLICY.QUEUE CLASS(MQQUEUE) +
 ID(MQUSERS,MQSTC) ACC(UPDATE)

RDEFINE MQQUEUE ZQS1.DEAD.LETTER.QUEUE OWNER(SYS1)
PERMIT ZQS1.DEAD.LETTER.QUEUE CLASS(MQQUEUE) RESET
PERMIT ZQS1.DEAD.LETTER.QUEUE CLASS(MQQUEUE) +
 ID(MQSTC,CICSUSER) ACC(UPDATE)

RDEFINE MQQUEUE ZQS1.SYSTEM.COMMAND.REPLY.MODEL OWNER(SYS1)
PERMIT ZQS1.SYSTEM.COMMAND.REPLY.MODEL CLASS(MQQUEUE) RESET
PERMIT ZQS1.SYSTEM.COMMAND.REPLY.MODEL CLASS(MQQUEUE) +
 ID(MQSTC,MQUSERS) ACC(UPDATE)

RDEFINE MQQUEUE ZQS1.SYSTEM.CSQOREXX.** OWNER(SYS1)
PERMIT ZQS1.SYSTEM.CSQOREXX.** CLASS(MQQUEUE) RESET
PERMIT ZQS1.SYSTEM.CSQOREXX.** CLASS(MQQUEUE) +
 ID(MQSTC) ACC(ALTER)
PERMIT ZQS1.SYSTEM.CSQOREXX.** CLASS(MQQUEUE) +
 ID(MQUSERS) ACC(UPDATE)

RDEFINE MQQUEUE ZQS1.SYSTEM.PROTECTION.ERROR.QUEUE OWNER(SYS1)
PERMIT ZQS1.SYSTEM.PROTECTION.ERROR.QUEUE CLASS(MQQUEUE) RESET
PERMIT ZQS1.SYSTEM.PROTECTION.ERROR.QUEUE CLASS(MQQUEUE) +
 ID(MQUSERS) ACC(UPDATE)

RDEFINE MQQUEUE ZQS1.SYSTEM.DEFAULT.LOCAL.QUEUE OWNER(SYS1)
PERMIT ZQS1.SYSTEM.DEFAULT.LOCAL.QUEUE CLASS(MQQUEUE) RESET
PERMIT ZQS1.SYSTEM.DEFAULT.LOCAL.QUEUE CLASS(MQQUEUE) +
 ID(MQUSERS,MQSTC,MQSYSP) ACC(UPDATE)

RDEFINE MQQUEUE ZQS1.DEAD.LETTER.QUEUE OWNER(SYS1)
PERMIT ZQS1.DEAD.LETTER.QUEUE CLASS(MQQUEUE) RESET
PERMIT ZQS1.DEAD.LETTER.QUEUE CLASS(MQQUEUE) +
 ID(MQUSERS,MQSTC,MQSYSP) ACC(UPDATE)

RDEFINE MQQUEUE ZQS1.AMSDEMO.** OWNER(SYS1)
PERMIT ZQS1.AMSDEMO.** CLASS(MQQUEUE) RESET
PERMIT ZQS1.AMSDEMO.** CLASS(MQQUEUE) ID(MQUSERS) ACC(UPDATE)
PERMIT ZQS1.AMSDEMO.** CLASS(MQQUEUE) ID(MQSTC) ACC(UPDATE)

RDEFINE MQQUEUE ZQS1.USER1.** OWNER(SYS1)
PERMIT ZQS1.USER1.** CLASS(MQQUEUE) RESET
PERMIT ZQS1.USER1.** CLASS(MQQUEUE) ID(USER1) ACC(UPDATE)
PERMIT ZQS1.USER1.** CLASS(MQQUEUE) ID(MQSTC) ACC(UPDATE)

SETROPTS RACLIST(MQQUEUE) REFRESH
```



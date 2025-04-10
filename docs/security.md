Implementing Queue Manager Security

Exercise Objectives
General Exercise Information and Guidelines
Part 1 – Configuring Base RACF resources
Part 2 – Testing Local Security to the Queue Manager

## Exercise Objectives

The objective of this exercise is to gain experience with protecting access to IBM MQ queues, MQ commands and connection to a queue manager using RACF. In this exercise you will enable the RACF protection of these resources and then provide appropriate access to these resources to different sets of users both local and remote.

## General Exercise Information and Guidelines

 This exercise requires using TSO user USER1 on the MQ sysplex z/OS system.

 The TSO password for this exercise will be provided by the lab instructor. 

 This exercise uses data set USER1.SECURITY.JCL.

 MVS commands are identified by a leading slash (/). This is reminder that these commands need to be entered using SDSF. 

 Case (upper or lower) is very important when invoking the RACDCERT and keytool commands in this exercise. When you see the use of upper characters in these commands enter the commands exactly as they appear in the text. More than likely if there are any unexpected results in the SSL sections of this exercise they will be caused by case inconsistencies.

 Text in bold and highlighted in yellow in this document should be available for copying and pasting in a file named MQ Exercises Cut Paste file on the desktop. 

## Part 1 – Configuring Base RACF resources

In this part of the exercise we begin by reviewing the users and the access their roles require. There are multiple functions that should be protected. 

The users have been connected to RACF groups and where each groups had access to different functions. 

**MQUSERS** - The first group will be the general MQ users that need access to basic MQ functions. They will be able to use MQExplorer and ISPF panels and are limited to what actions they can perform. 

**MQSYSP** - The second set of users will be MQ administrators. These are users who need the same MQ access as general users but will also need the ability to enter protected MQ commands and full authority in MQExplorer and ISPF panels. 

Another group will be the identities used for the MQ started tasks (MQSTC) and CICS regions (CICSSTC). 

Another set of general uses are CICS users who also need access to CICS resources as well as a set of CICS related MQ resources. 

A single user could be a member of multiple groups.

## Lab Begin

1. Log on to TSO using the PComm-3270 icon on this desktop

2. Use the RACF command LISTGROUP (or LG) to list the users currently connected to the groups described above, e.g. LG SYS1. You will see something like the results below.

    ```
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

    > The LISTUSER command can also be useful for investigating the permissions of a particular userid. As an example, LISTUSER USER1

3. External security checking is enabled or disabled for a specific queue manager either by the presence or absence of specific MQADMIN RACF resources during the queue manager initialization. During startup, the queue manager uses its name (e.g. ZQS1) to look for a specific SYSPROG resource. If the resource is defined, then its presence disables the corresponding security checking. 

For example for queue manager ZQS1, if MQADMIN resource ZQS1.NO.TOPIC.CHECKS is defined then external security checking for topics will be disabled. If MQADMIN resource ZQS1.NO.QUEUE.CHECKS is defined then external security checking for queue access will be 
disabled. 

To determine what MQADMIN resources are currently defined use the RACF SEARCH command 
to display the currently defined MQADMIN resources for ZQS1. 

    SEARCH CLASS(MQADMIN) FILTER(ZQS1.**) 

The results should look like the list below: 

> Tech-Tip: All RACF resources associated with a specific queue manager are named with a prefix of the queue manager name. This allows the coexistences of multiples RACF resources for the same base resource name, i.e. ZQS1.SYSTEM.DEFAULT.LOCAL.QUEUE will have different security protection than ZQS2.SYSTEM.DEFAULT.LOCAL.QUEUE.


> Tech-Tip: The RACF SEARCH command is a useful for managing RACF resources. The FILTER option is flexible for specifying the search criteria and adding the EXEC option provides a means for generating additional commands. For example adding 

    CLIST(’RLIST MQADMIN ’, ’AUTHUSER’) 
 
will generate a CLIST named EXEC.RACF.CLIST) with a series of RLIST commands for 
displaying the authorized users for every resource found. Adding 
 CLIST(’RDELETE MQADMIN ’) will generate a CLIST named EXEC.RACF.CLIST with a series of RDELETE commands for deleting the resource found.

```
ZQS1.NO.ALTERNATE.USER.CHECKS 
ZQS1.NO.CMD.RESC.CHECKS 
ZQS1.NO.CONTEXT.CHECKS 
ZQS1.NO.NLIST.CHECKS 
ZQS1.NO.PROCESS.CHECKS 
ZQS1.NO.SUBSYS.SECURITY 
ZQS1.NO.TOPIC.CHECKS 
ZQS1.RESLEVEL 
```

4. The MQADMIN resources in this list disable various MQ external security checks. One of these one in particular disables all external checking regardless of any other MQADMIN resource and that is ZQS1.NO.SUBSYS.SECURITY. The presence of this resource during the queue manager’s initialization disables all external security checking as shown by the startup message shown below. 


    CSQH021I ZQS1 CSQHINIT SUBSYSTEM security switch set 
    OFF, profile 'ZQS1.NO.SUBSYS.SECURITY' found 


5. To enable external security checking for the next restart of the queue manager, use the RACF RDELETE command to delete this MQADMIN resource. 

    RDELETE MQADMIN ZQS1.NO.SUBSYS.SECURITY

6. Refresh the RACF instorage profiles with the RACF SETROPTS command 

    SETROPTS RACLIST(MQADMIN) REFRESH

7. Shutdown the ZQS1 queue manager with MVS command 

     /ZQS1 STOP QMGR

> Tech-Tip: To disable external security checking use the RDEFINE to command to define this resource.

    RDEFINE MQADMIN ZQS1.NO.SUBSYS.SECURITY OWNER(SYS1)

> Tech-Tip: You probably will receive message ICH14070I SETROPTS RACLIST REFRESH had no effect on class MQADMIN. Not every RACF configuration requires a refresh of the RACF instorage profiles but it is a good practice to get in the habit of doing a refresh after making changes. This will avoid making a configuration change and not having it made active.

CSQH021I ZQS1 CSQHINIT SUBSYSTEM security switch set 
OFF, profile 'ZQS1.NO.SUBSYS.SECURITY' found 

8. Before the queue manager is restarted, some basic MQ RACF resources should be defined. Select member MQCMDS in data set USER1.SECURITY.JCL. This JCL defines MQCMDS RACF resources for some of the basic MQ commands and then grants users in group STCMQ and MQADMS access.

9. Submit MQCMDS for execution and verify that it completes with a condition code of zero. 
  
> Tech-Tip: The SEARCH and EXEC commands at the beginning of this job delete all existing MQCMDS profiles for queue manager ZQS1. 


```
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

10. Select member MQCONN in data set USER1.SECURITY.JCL. This JCL defines MQCONN RACF resources required for accessing the queue manager from various sources, e.g., CICS, batch, remote, etc.

11. Submit MQCONN for execution and verify that it completes with a condition code of zero.

12. Select member MQQUEUE in data set USER1.SECURITY.JCL. This JCL defines MQQUEUE 
RACF resources required for accessing the system related queues.

```
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
 
13. Submit MQQUEUE for execution and verify that it completes with a condition code of zero.This completes the configuration of the RACF resources required to start a basic queue manger.

```
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

## Part 2 – Testing Local Security to the Queue Manager

Queue manager security has been enabled by the deletion of MQADMIN resource 
ZQS1.NO.SUBSYS.SECURITY and some of the initial RACF resources for MQCONN, MQQUEUE 
and MQCMDS have been defined and access granted. 

Now restart the queue manager and see if the resources that have been defined are sufficient or if adjustments should be made.

1. Start the ZQS1 queue manager with MVS command /ZQS1 START QMGR

2. Monitor the startup messages of MQ task ZQS1MSTR and note the additional messages that now appear because ZQS1.NO.SUBSYS.SECURITY was not found.

• MQADMIN resources ZQS1.NO.CONNECT.CHECKS was not found so connection security 
is now enabled. 

• MQADMIN resource ZQS1.NO.CMD.CHECKS was not found so command security is now 
enabled. 

• Finally MQADMIN resource ZQS1.NO.QUEUE.CHECKS was not found so queue security 
is enabled. 

• Security for the other MQ resources is stilled disabled because of the presence of the MQADMIN resources displayed in Part 1 Step 3.

```
CSQH024I ZQS1 CSQHINIT SUBSYSTEM security switch set 
ON, profile 'ZQS1.NO.SUBSYS.SECURITY' not found 
CSQH024I ZQS1 CSQHINIT CONNECTION security switch set 
ON, profile 'ZQS1.NO.CONNECT.CHECKS' not found 
CSQH024I ZQS1 CSQHINIT COMMAND security switch set ON, 
profile 'ZQS1.NO.CMD.CHECKS' not found 
CSQH021I ZQS1 CSQHINIT CONTEXT security switch set 
OFF, profile 'ZQS1.NO.CONTEXT.CHECKS' found 
CSQH021I ZQS1 CSQHINIT ALTERNATE USER security switch 
set OFF, profile 'ZQS1.NO.ALTERNATE.USER.CHECKS' found 
CSQH021I ZQS1 CSQHINIT COMMAND RESOURCES security 
switch set OFF, profile 'ZQS1.NO.CMD.RESC.CHECKS' found 
CSQH021I ZQS1 CSQHINIT PROCESS security switch set 
OFF, profile 'ZQS1.NO.PROCESS.CHECKS' found 
CSQH021I ZQS1 CSQHINIT NAMELIST security switch set 
OFF, profile 'ZQS1.NO.NLIST.CHECKS' found 
CSQH024I ZQS1 CSQHINIT QUEUE security switch set ON, 
profile 'ZQS1.NO.QUEUE.CHECKS' not found 
CSQH021I ZQS1 CSQHINIT TOPIC security switch set OFF,
profile 'ZQS1.NO.TOPIC.CHECKS' found 
```

3. Continue monitoring the startup messages of ZQS1 and eventually you will see these messages. 

```
ICH408I USER(START3 ) GROUP(SYS1 ) NAME(################# 
 QMZ1.ZCONN2.TRIGGER.INITQ CL(MQQUEUE ) 
 INSUFFICIENT ACCESS AUTHORITY 
 FROM QMZ1.** (G) 
 ACCESS INTENT(UPDATE ) ACCESS ALLOWED(NONE ) 
 ```

These messages occurred because name of queues ZQS1.CICS01.INITQ and ZQS1.ZCONN2.TRIGGER.INITQ only matched the generic MQQUEUE profile ZQS1.** which only provides READ access to a queue. In this case user START1 needs UPDATE access to these queues. START1 is a member of group MQSTC so this group should be granted this access. 

4. Use RACF command RDEFINE to define a profile for this queue giving START1 update access.

    RDEFINE MQQUEUE ZQS1.CICS01.INITQ OWNER(SYS1)
    RDEFINE MQQUEUE ZQS1.ZCONN2.TRIGGER.INITQ OWNER(SYS1)

5. Use the RACF PERMIT command to give UPDATE access to this queue to group MQSTC and 
CICSSTC. 

    PERMIT ZQS1.CICS01.INITQ CLASS(MQQUEUE) ID(MQSTC,CICSSTC,CICSUSER) ACC(UPDATE)
    PERMIT ZQS1.ZCONN2.TRIGGER.INITQ CLASS(MQQUEUE) ID(MQSTC,CICSSTC,CICSUSER) ACC(UPDATE)

6. Refresh the RACF instorage profiles will command RACF command SETROPTS.
SETROPTS RACLIST(MQQUEUE) REFRESH

7. Refresh the queue manager’s RACF information with MVS command

    /ZQS1 REFRESH SECURITY(MQQUEUE)

> Tech-Tip: In this workshop we do not use mixed case MQ resources so the case of the queues, etc. name is not important in this workshop. If you are using mixed case for your MQ resources, there are corresponding RACF resources to the one used in this workshop. For example, MXQUEUE is the mixed case equivalent of MQQUEUE and MXADMIN is the mixed case equivalent of MQADMIN.

```
ICH408I USER(START1 ) GROUP(SYS1 ) NAME(#################
 ZQS1.CICS01.INITQ CL(MQQUEUE ) 
 INSUFFICIENT ACCESS AUTHORITY 
 FROM ZQS1.** (G) 
 ACCESS INTENT(UPDATE ) ACCESS ALLOWED(READ ) 
ICH408I USER(START1 ) GROUP(SYS1 ) NAME(################# 
 ZQS1.ZCONN2.TRIGGER.INITQ CL(MQQUEUE ) 
 INSUFFICIENT ACCESS AUTHORITY 
 FROM ZQS1.** (G) 
 ACCESS INTENT(UPDATE ) ACCESS ALLOWED(READ ) ) 
```

You should see these messages in the console.

8. Display the new MQQUEUE profile just defined with a RACF RLIST command 

    RLIST MQQUEUE ZQS1.CICS01.INITQ AUTHUSER
    RLIST MQQUEUE ZQS1.ZCONN2.TRIGGER.INITQ AUTHUSER

Notice that user USER1 has explicit alter authority to this resource. This may not be what you want and that is why the PERMIT RESET command is useful in the jobs run earlier.

```
USER ACCESS ACCESS COUNT 
---- ------ ------ ----- 
USER1   ALTER     000000 
MQSTC   UPDATE    000000 
CICSSTC UPDATE    000000
```

Next enter command MVS command /ZQS1 ALTER QMGR MAXCHL(350). This request should fail 
because USER1 does not have sufficient access to the MQCMDS resource ZQS1.**.

```
QMZ1 ALTER QMGR MAXCHL(350) 
ICH408I USER(USER1 ) GROUP(SYS1 ) NAME( ) 
 QMZ1.ALTER.QMGR CL(MQCMDS ) 
 INSUFFICIENT ACCESS AUTHORITY 
 FROM QMZ1.** (G) 
 ACCESS INTENT(ALTER ) ACCESS ALLOWED(READ ) 
CSQ9016E QMZ1 ' ALTER' command request not authorized
```

9. MQCMDS resource ZQS1.** is a fail-safe or default profile for MQ commands. If it did not exist and a queue manager did a security check for a command that did not have an explicit MQCMDS resource, then this message would appear. 

```
ICH408I USER(USER1 ) GROUP(SYS1 ) NAME( 
 QMZ1.ALTER.QMGR CL(MQCMDS ) 
 PROFILE NOT FOUND - REQUIRED FOR AUTHORITY CHECKING 
 ACCESS INTENT(ALTER ) ACCESS ALLOWED(NONE ) 
 CSQ9016E QMZ1 ' ALTER' command request not authorized 
 CSQ9023E QMZ1 CSQ9SCND 'ALTER QMGR' ABNORMAL COMPLETION
```

READ access to some MQ commands is sufficient for authority purposes, e.g. DISPLAY but 
commands like ALTER should have a higher level of protection. This is why there is a default MQ command profile that only gives READ access. 

10. To give access to the MQ ALTER command an explicit MQCMDS profile needs to be created with command 

```
RDEFINE MQCMDS ZQS1.ALTER.** OWNER(SYS1)
ZQS1 ALTER QMGR MAXCHL(350) 
ICH408I USER(USER1 ) GROUP(SYS1 ) NAME( ) 
 ZQS1.ALTER.QMGR CL(MQCMDS ) 
 INSUFFICIENT ACCESS AUTHORITY 
 FROM ZQS1.** (G) 
 ACCESS INTENT(ALTER ) ACCESS ALLOWED(READ ) 
CSQ9016E ZQS1 ' ALTER' command request not authorized 
 
ZQS1 REFRESH SECURITY(MQQUEUE) 
CSQH001I ZQS1 CSQHCHK4 Security using uppercase classes 
CSQ9022I ZQS1 CSQHSREF ' REFRESH SECURITY' NORMAL COMPLETION
USER ACCESS ACCESS COUNT 
---- ------ ------ ----- 
USER1 ALTER 000000 
MQSTC UPDATE 000000 
CICSSTC UPDATE 000000 
ICH408I USER(USER1 ) GROUP(SYS1 ) NAME( 
 ZQS1.ALTER.QMGR CL(MQCMDS ) 
 PROFILE NOT FOUND - REQUIRED FOR AUTHORITY CHECKING 
 ACCESS INTENT(ALTER ) ACCESS ALLOWED(NONE ) 
 CSQ9016E ZQS1 ' ALTER' command request not authorized 
CSQ9023E ZQS1 CSQ9SCND 'ALTER QMGR' ABNORMAL COMPLETION
```

____11. And groups MQSTC and MQSYSP given ALTER access with command

    PERMIT ZQS1.ALTER.** CLASS(MQCMDS) ID(MQSTC,MQSYSP) ACC(ALTER)

____12. Refresh the RACF instorage profiles with RACF command SETROPTS.
SETROPTS RACLIST(MQCMDS) REFRESH MQCMDS resources cannot be refreshed with a MQ REFRESH command. The only way they are update is during a queue manager restart. 

____13. Shut the queue manager with MVS command 

    /ZQS1 STOP QMGR

____14. Restart the queue manager and try to repeat the ALTER command. The change to the queue manger should now be successful.

____15. Start MQ Explorer, if it is not already active. Select the ZQS1 queue manger and right mouse button click and select Connection Details -> Properties. On the ZQS1 - Properties windows select Useridon the left-hand side and change the user id from USER1 to USER2 and if necessary use the Clear password and Enter password buttons to change the password to USER2. 

____16. Click OK to continue and reconnect to queue manager ZQS1 as USER2. 

____17. Try to create a new queue using name USER2.TEST.QUEUE. The request should fail because USER2 does not have ALTER authority to MQCMDS resource ZQS1.DEFINE.**. 
The messages below should appear in the console.

____18. Try to delete queue USER1.TEST.QUEUE. Again the request should fail because USER2 does not have ALTER authority MQCMDS resource to ZQS1.DELETE.**.

____19. Use MQ Explorer to try putting a message to queue USER1.TEST.QUEUE. This should fail because of insufficient authority to MQQUEUE resource ZQS1.**.

____20. Next use MQ Explorer to try putting a message to queue SYSTEM.DEFAULT.LOCAL.QUEUE. This should work because there is a MQQUEUE RACF resource for queue 
ZQS1.SYSTEM.DEFAULT.LOCAL.QUEUE that allows groups MQUSERS, MQSTC and MQSYSP
UPDATE access. Confirm with a RLIST RACF command 

    RLIST MQQUEUE ZQS1.SYSTEM.DEFAULT.LOCAL.QUEUE AUTHUSER

____21. Change the MQ Explorer connection to queue manager ZQS1 back to USER1 and try creating a queue and then deleting to confirm that USER1 has the authority to perform these functions.

____22. Now select member PUTMSG in data set USER1.SECURITY.JCL. This JCL places messages on the SYSTEM.DEFAULT.LOCAL.QUEUE. This job runs using USER1’s authority. Submit the job and confirm that the messages are successfully written to the queue. Again this is successfully because USER1 is a member of group MQUSERS which has access to this queue.

____23. Change all occurrences of USER1 to USER3 and resubmit this job for execution. Putting the messages to the queue should fail this time with message.

____24. A review of the console will find this message:

```
CH408I USER(USER2 ) GROUP(SYS1 ) NAME(####################)
 ZQS1.DEFINE.QUEUE CL(MQCMDS ) 
 INSUFFICIENT ACCESS AUTHORITY 
 FROM ZQS1.DEFINE.** (G) 
ACCESS INTENT(ALTER ) ACCESS ALLOWED(NONE ) 
ICH408I USER(USER2 ) GROUP(SYS1 ) NAME(####################)
 ZQS1.DELETE.QUEUE CL(MQCMDS ) 
 INSUFFICIENT ACCESS AUTHORITY 
 FROM ZQS1.DELETE.** (G) 
 ACCESS INTENT(ALTER ) ACCESS ALLOWED(NONE ) 
ICH408I USER(USER2 ) GROUP(SYS1 ) NAME(####################)
 ZQS1.USER1.TEST.QUEUE CL(MQQUEUE ) 
 INSUFFICIENT ACCESS AUTHORITY 
 FROM ZQS1.USER1.** (G) 
 ACCESS INTENT(UPDATE ) ACCESS ALLOWED(NONE ) 
An MQSeries error occurred : Authentication Error. Ensure the correct credentials were used.
A WebSphere MQ error occurred : Completion code 2 Reason code 2035
ICH408I USER(USER3 ) GROUP(SYS1 ) NAME( )
 ZQS1.BATCH CL(MQCONN ) 
 INSUFFICIENT ACCESS AUTHORITY 
 ACCESS INTENT(READ ) ACCESS ALLOWED(NONE ) 
```

____25. USER3 is not a member of group MQUSERS and therefore does not have basic MQ authority. This security issue can be address (assuming USER3 should have access) by connecting USER3 to group MQUSERS with RACF command:

    CONNECT USER3 GROUP(MQUSERS)

____26. Resubmit PUTMST under USER3’s authority and the messages should now be written to the queue successfully.

These steps have tested the MQ command authority, queue access and connection resources using the external security manager.
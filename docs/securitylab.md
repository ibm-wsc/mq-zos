# Configuring RACF resources for MQ Security

#### **Audience level**

Some knowledge of MQ or z/OS 

#### **Skillset**

MQ Administration, z/OS systems programming

#### **Background**

### Exercise Objectives

The objective of this exercise is to gain experience with protecting access to IBM MQ queues, MQ commands and connection to a queue manager using RACF. In this exercise you will enable the RACF protection of these resources and then provide appropriate access to these resources to different sets of users both local and remote.

For this exercise, you will be using a data set called ZQS1.SECURITY.JCL. We will be using the following members:
-ADDGROUP - Define MQSTC, CICSSTC, MQUSERS and connect users to those groups
-MQCMDS  
-MQCONN  
-MQQUEUE 

### Important Notes

Before we get started, let's review some RACF terminology you will use in the lab:

1) RDEFINE - This command adds a profile for the resource to the RACF database in order to control access to the resource

2) PERMIT - This command maintains the lists of users and groups authorized to access a particular resource.

Please note: MVS commands are identified by a leading slash (/). This is reminder that these commands need to be entered using SDSF. 

Please note: Case (upper or lower) is very important when invoking the RACDCERT and keytool commands in this exercise.


## Lab Begin

#### I. Enable Security Checking on queue manager ZQS1

External security checking is enabled or disabled for a specific queue manager either by the presence or absence of specific MQADMIN RACF resources during the queue manager initialization. During startup, the queue manager uses its name (e.g. ZQS1) to look for a specific SYSPROG resource. If the resource is defined, then its presence disables the corresponding security checking. 

For example for queue manager ZQS1, if MQADMIN resource ZQS1.NO.TOPIC.CHECKS is defined then external security checking for topics will be disabled. If MQADMIN resource ZQS1.NO.QUEUE.CHECKS is defined then external security checking for queue access will be 
disabled. 

What we do in this lab is start from the default of security ON, disable security, and then add back customized security settings for our MQ resources.

1. Log on to TSO/ISPF using your assigned z/OS credentials.

2. From option 6 on the ISPF main menu, use the RACF command <code>LISTGROUP</code> (or LG) to list the users currently connected to the groups described above, e.g. LG SYS1. You will see something like the results below.

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


3\. To determine what MQADMIN resources are currently defined use the RACF SEARCH command 
to display the currently defined MQADMIN resources for ZQS1. 

    SEARCH CLASS(MQADMIN) FILTER(ZQS1.**) 

The results should look like the list below: 

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

> Tech-Tip: All RACF resources associated with a specific queue manager are named with a prefix of the queue manager name. This allows the coexistences of multiples RACF resources for the same base resource name, i.e. ZQS1.SYSTEM.DEFAULT.LOCAL.QUEUE will have different security protection than ZQS2.SYSTEM.DEFAULT.LOCAL.QUEUE.

4\. The MQADMIN resources in this list disable various MQ external security checks. One of these one in particular disables all external checking regardless of any other MQADMIN resource and that is ZQS1.NO.SUBSYS.SECURITY. If you see  ZQS1.NO.SUBSYS.SECURITY, this indicates that all external security checking is currently disabled. 

    CSQH021I ZQS1 CSQHINIT SUBSYSTEM security switch set 
    OFF, profile 'ZQS1.NO.SUBSYS.SECURITY' found 

5\. To enable external security checking for the next restart of the queue manager, use the RACF RDELETE command to delete this MQADMIN resource. 

    RDELETE MQADMIN ZQS1.NO.SUBSYS.SECURITY

6\. Refresh the RACF instorage profiles with the RACF SETROPTS command 

    SETROPTS RACLIST(MQADMIN) REFRESH

7\. Shutdown the ZQS1 queue manager with MVS command 

     /ZQS1 STOP QMGR

> Tech-Tip: To disable external security checking use the RDEFINE to command to define this resource.

    RDEFINE MQADMIN ZQS1.NO.SUBSYS.SECURITY OWNER(SYS1)

> Tech-Tip: You probably will receive message ICH14070I SETROPTS RACLIST REFRESH had no effect on class MQADMIN. Not every RACF configuration requires a refresh of the RACF instorage profiles but it is a good practice to get in the habit of doing a refresh after making changes. This will avoid making a configuration change and not having it made active.

CSQH021I ZQS1 CSQHINIT SUBSYSTEM security switch set 
OFF, profile 'ZQS1.NO.SUBSYS.SECURITY' found 

8\. Before the queue manager is restarted, some basic MQ RACF resources should be defined. Select member MQCMDS in data set ZQS1.SECURITY.JCL. This JCL defines MQCMDS RACF resources for some of the basic MQ commands and then grants users in group STCMQ and MQADMS access.

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

10. Select member MQCONN in data set ZQS1.SECURITY.JCL. This JCL defines MQCONN RACF resources required for accessing the queue manager from various sources, e.g., CICS, batch, remote, etc.

11. Submit MQCONN for execution and verify that it completes with a condition code of zero.

12. Select member MQQUEUE in data set ZQS1.SECURITY.JCL. This JCL defines MQQUEUE 
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

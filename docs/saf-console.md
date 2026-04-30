# Implementing Basic Authentication Using SAF for the MQ Web Console

#### **Audience level**
Some knowledge of MQ, z/OS, and Liberty

#### **Skillset**
MQ Administration, z/OS systems programming

#### **Background**

Basic authentication uses a user ID and password to access a resource. The MQ Web Console supports two basic authentication approaches: the basic registry and the SAF registry. This lab covers the SAF registry approach.

Modern production environments often prefer stronger authentication approaches such as tokens, passkeys, or enterprise single sign-on. For that reason, this lab should be treated as a learning exercise for sandbox or lab environments rather than a production deployment pattern.

### **Overview of the exercise**

In this lab, you will:

I. Copy `zos_saf_registry.xml`

II. Alter `server.xml`

III. Customize `zos_saf_registry.xml`

IV. Configure RACF for the MQ Web Console SAF registry

V. Start the Liberty angel process

VI. Configure RACF for the angel process

VII. Restart and validate the MQ Web Console

### Prerequisites

Before you begin, make sure the following are available:

- IBM MQ for z/OS with the MQ Web Console installed
- Access to the `mqweb` server configuration directory
- Authority to edit the Liberty configuration files
- Authority to define and permit RACF resources in the `APPL`, `EJBROLE`, and `SERVER` classes
- The user ID under which the MQ Web Console started task runs
- The SAF profile prefix you intend to use, for example `MQW942`
- The unauthenticated or pre-authentication user ID you intend to use, for example `MQGUEST`
- Authority to start the Liberty angel started task and the MQ Web Console started task

> Note: This lab uses `MQW942` as the profile prefix and `SYSPROG` as the started task user in examples. Replace those values with the ones used in your environment.

### **Exercise**

#### I. Copy `zos_saf_registry.xml`

1. From the ISPF main menu, enter:

   ```text
   tso omvs
   ```

2. In OMVS, list the sample MQ Web Console configuration files:

   ```text
   ls /usr/lpp/mqm/V9R4M2X/web/mq/samp/configuration
   ```

   You should see a list of XML files, including `zos_saf_registry.xml`.

3. Change to the `mqweb` server directory and copy the sample SAF registry file into it:

   ```text
   cd /var/mqm/servers/mqweb
   cp /usr/lpp/mqm/V9R4M2X/web/mq/samp/configuration/zos_saf_registry.xml .
   ```

   > Important: Do not miss the final `.`. It means the file is copied into the current directory.

4. Type `exit` to leave OMVS.

5. Return to the `mqweb` directory in ISPF and confirm that `zos_saf_registry.xml` is now present.

#### II. Alter `server.xml`

6. Navigate to the `/var/mqm/servers/mqweb` directory from ISPF option `3.4`.

7. List the contents of the directory and locate `server.xml` and `zos_saf_registry.xml`.

8. Edit `server.xml`.

9. Replace:

   ```xml
   <include location="basic_registry.xml"/>
   ```

   with:

   ```xml
   <include location="zos_saf_registry.xml"/>
   ```

10. Save `server.xml` and exit.

#### III. Customize `zos_saf_registry.xml`

11. Edit `zos_saf_registry.xml` in ASCII mode.

12. Modify the sample configuration so it looks similar to the following:

   ```xml
   <server>
       <featureManager>
           <feature>appSecurity-2.0</feature>
           <feature>zosSecurity-1.0</feature>
           <feature>basicAuthenticationMQ-1.0</feature>
           <feature>apiDiscovery-1.0</feature>
       </featureManager>

       <enterpriseApplication id="com.ibm.mq.console" />

       <enterpriseApplication id="com.ibm.mq.rest" />

       <safRegistry id="saf" />
       <safAuthorization id="saf" reportAuthorizationCheckDetails="true"/>
       <safCredentials unauthenticatedUser="MQGUEST" profilePrefix="MQW942" suppressAuthFailureMessages="false"/>
       <sslDefault sslRef="mqDefaultSSLConfig"/>
   </server>
   ```

13. Review the key values:
   - `profilePrefix="MQW942"` identifies the RACF profile prefix used by Liberty and MQ Web
   - `unauthenticatedUser="MQGUEST"` identifies the user used before full authentication occurs, allowing the login page to be presented
   - `safAuthorization` enables SAF-based authorization for the web console and REST applications

14. Save the file and exit.

15. Edit `mqwebuser.xml` in ASCII mode and make sure it contains a configuration similar to the following:

   ```xml
   <server>
       <featureManager>
           <feature>appSecurity-2.0</feature>
       </featureManager>
       <webAppSecurity allowFailOverToBasicAuth="true"/>
       <webAppSecurity overrideHttpAuthMethod="BASIC"/>
       <variable name="httpsPort" value="9443"/>
       <variable name="httpHost" value="-1"/>
       <httpEndpoint host="*" httpPort="-1" httpsPort="9443"
           id="defaultHttpEndpoint"/>
       <sslDefault sslRef="mqDefaultSSLConfig"/>
   </server>
   ```

16. Save `mqwebuser.xml` and exit.

> Note: Some sites choose to keep this configuration in a single file. In this lab, the configuration is kept separate to make it easier to switch between basic registry and SAF registry examples.

#### IV. Configure RACF for the MQ Web Console SAF registry

17. Return to the ISPF main menu and enter option `6` for the TSO command line.

18. Define the application profile for the MQ Web Console using your SAF profile prefix. For example:

   ```text
   RDEFINE APPL MQW942 UACC(NONE)
   ```

19. Permit access to users who need to reach the MQ Web login screen, including the unauthenticated user ID specified in `zos_saf_registry.xml`. For example:

   ```text
   PERMIT MQW942 CLASS(APPL) ACCESS(READ) ID(MQGUEST)
   PERMIT MQW942 CLASS(APPL) ACCESS(READ) ID(userID)
   ```

20. Refresh the `APPL` class:

   ```text
   SETROPTS RACLIST(APPL) REFRESH
   ```

21. Define the `EJBROLE` profiles used for MQ Console and REST security roles:

   ```text
   RDEFINE EJBROLE MQW942.com.ibm.mq.console.MQWebAdmin UACC(NONE)
   RDEFINE EJBROLE MQW942.com.ibm.mq.console.MQWebAdminRO UACC(NONE)
   RDEFINE EJBROLE MQW942.com.ibm.mq.console.MQWebUser UACC(NONE)
   RDEFINE EJBROLE MQW942.com.ibm.mq.rest.MQWebAdmin UACC(NONE)
   RDEFINE EJBROLE MQW942.com.ibm.mq.rest.MQWebAdminRO UACC(NONE)
   RDEFINE EJBROLE MQW942.com.ibm.mq.rest.MQWebUser UACC(NONE)
   RDEFINE EJBROLE MQW942.com.ibm.mq.rest.MFTWebAdmin UACC(NONE)
   RDEFINE EJBROLE MQW942.com.ibm.mq.rest.MFTWebAdminRO UACC(NONE)
   ```

22. Permit users to the appropriate roles. For example, to allow a user full administrative access to the REST API:

   ```text
   PERMIT MQW942.com.ibm.mq.rest.MQWebAdmin CLASS(EJBROLE) ACCESS(READ) ID(userID)
   ```

23. Repeat the necessary `PERMIT` commands for the MQ Console and REST roles required by your users.

24. A simple role mapping summary is shown below:

   | Role | Typical use |
   | --- | --- |
   | `MQWebAdmin` | Full administrative access to the MQ Web Console or REST API |
   | `MQWebAdminRO` | Read-only administrative access |
   | `MQWebUser` | Basic user access with fewer administrative capabilities |

24. Refresh the relevant RACF class for these role definitions according to your site standards.

> Note: If `EJBROLE` is RACLISTed in your environment, refresh that class after the updates. The original lab refreshed `APPL` a second time here, but in practice you should make sure the class containing the updated definitions is refreshed.

#### V. Start the angel process

25. The Liberty angel process is a started task that allows Liberty servers to use z/OS authorized services. It is long-lived and can be shared by multiple Liberty servers.

26. Navigate to SDSF from the ISPF main menu.

27. In the SDSF command input line, start the angel task:

   ```text
   /S MQANGEL
   ```

28. In SDSF `DA`, set the prefix to `MQ*` and confirm that the `MQANGEL` address space is running.

#### VI. Configure the MQ Web started task for angel usage

29. Return to ISPF option `3.4` and navigate to the procedure library that contains the MQ Web started task, for example `SYS1.PROCLIB`.

30. Locate the `MQWEBS` procedure and edit it.

31. Review the `STDENV` section. Make sure the following Java option is present in the active environment section used by the started task:

   ```text
   IBM_JAVA_OPTIONS=-Dcom.ibm.ws.zos.core.angelName=MQANGEL
   ```

32. Make sure it is placed in the active environment section, not in an unused or commented section.

33. Save the procedure and exit.

#### VII. Configure RACF for angel process access

34. If you now try to restart the MQ Web Console, it may fail because the started task user ID does not yet have access to the required Liberty angel resources. In the following examples, the MQ Web started task user is `SYSPROG`.

35. From ISPF option `6`, define a profile for the angel and permit the started task user:

   ```text
   RDEFINE SERVER BBG.ANGEL UACC(NONE) OWNER(SYSPROG)
   PERMIT BBG.ANGEL CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
   ```

36. Define a server profile for the named angel process using the same profile prefix concept:

   ```text
   RDEFINE SERVER BBG.ANGEL.MQW942 UACC(NONE) OWNER(SYSPROG)
   PERMIT BBG.ANGEL.MQW942 CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
   ```

37. Define a server profile for the security prefix:

   ```text
   RDEFINE SERVER BBG.SECPFX.MQW942 UACC(NONE)
   PERMIT BBG.SECPFX.MQW942 CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
   ```

38. Define a server profile for the authorized module `BBGZSAFM`:

   ```text
   RDEFINE SERVER BBG.AUTHMOD.BBGZSAFM UACC(NONE) OWNER(SYSPROG)
   PERMIT BBG.AUTHMOD.BBGZSAFM CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
   ```

39. Define a server profile for SAF credential services:

   ```text
   RDEFINE SERVER BBG.AUTHMOD.BBGZSAFM.SAFCRED UACC(NONE)
   PERMIT BBG.AUTHMOD.BBGZSAFM.SAFCRED CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
   ```

40. Define a server profile for WLM services:

   ```text
   RDEFINE SERVER BBG.AUTHMOD.BBGZSAFM.ZOSWLM UACC(NONE)
   PERMIT BBG.AUTHMOD.BBGZSAFM.ZOSWLM CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
   ```

41. Define a server profile for RRS transaction services:

   ```text
   RDEFINE SERVER BBG.AUTHMOD.BBGZSAFM.TXRRS UACC(NONE)
   PERMIT BBG.AUTHMOD.BBGZSAFM.TXRRS CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
   ```

42. Define a server profile for dump services:

   ```text
   RDEFINE SERVER BBG.AUTHMOD.BBGZSAFM.ZOSDUMP UACC(NONE)
   PERMIT BBG.AUTHMOD.BBGZSAFM.ZOSDUMP CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
   ```

43. Define server profiles for local adapter services:

   ```text
   RDEFINE SERVER BBG.AUTHMOD.BBGZSAFM.WOLA UACC(NONE) OWNER(SYSPROG)
   PERMIT BBG.AUTHMOD.BBGZSAFM.WOLA CLASS(SERVER) ACCESS(READ) ID(SYSPROG)

   RDEFINE SERVER BBG.AUTHMOD.BBGZSAFM.LOCALCOM UACC(NONE) OWNER(SYSPROG)
   PERMIT BBG.AUTHMOD.BBGZSAFM.LOCALCOM CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
   ```

44. Define a server profile for product management services:

   ```text
   RDEFINE SERVER BBG.AUTHMOD.BBGZSAFM.PRODMGR UACC(NONE) OWNER(SYSPROG)
   PERMIT BBG.AUTHMOD.BBGZSAFM.PRODMGR CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
   ```

45. Define a server profile for asynchronous I/O services:

   ```text
   RDEFINE SERVER BBG.AUTHMOD.BBGZSAFM.ZOSAIO UACC(NONE) OWNER(SYSPROG)
   PERMIT BBG.AUTHMOD.BBGZSAFM.ZOSAIO CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
   ```

46. Define a server profile for the authorized client module `BBGZSCFM`:

   ```text
   RDEFINE SERVER BBG.AUTHMOD.BBGZSCFM UACC(NONE) OWNER(SYSPROG)
   PERMIT BBG.AUTHMOD.BBGZSCFM CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
   ```

47. Define a server profile for the client optimized local adapter services:

   ```text
   RDEFINE SERVER BBG.AUTHMOD.BBGZSCFM.WOLA UACC(NONE) OWNER(SYSPROG)
   PERMIT BBG.AUTHMOD.BBGZSCFM.WOLA CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
   ```

48. Refresh the `SERVER` class:

   ```text
   SETROPTS RACLIST(SERVER) REFRESH
   ```

#### VIII. Restart and validate MQ Web

49. Restart the MQ Web Console started task from SDSF.

50. Open the MQ Web Console in a browser and confirm that the login screen appears.

51. Sign in using a permitted RACF user ID and password.

52. Confirm that the user can access the MQ Console or REST API functions allowed by the assigned `EJBROLE` profiles.

### Validation checklist

Before considering the lab complete, confirm the following:

- `zos_saf_registry.xml` was copied into the `mqweb` server directory
- `server.xml` includes `zos_saf_registry.xml`
- `zos_saf_registry.xml` contains the correct `profilePrefix` and `unauthenticatedUser`
- RACF `APPL` profiles were created and permitted correctly
- RACF `EJBROLE` profiles were created and permitted correctly
- `MQANGEL` is running
- The `MQWEBS` started task environment includes the angel Java option
- RACF `SERVER` profiles for angel access were defined and refreshed
- The MQ Web Console starts successfully
- A permitted RACF user can log in successfully

### Troubleshooting

If the MQ Web Console does not work as expected, check the following:

- The sample file was copied from the correct MQ version directory
- `server.xml` includes `zos_saf_registry.xml` and not `basic_registry.xml`
- `mqwebuser.xml` contains a valid `httpsPort` attribute and valid XML syntax
- The `profilePrefix` in the XML matches the RACF profile names you created
- The `unauthenticatedUser` exists and has the required `APPL` access
- The MQ Web started task user has access to the required `SERVER` class profiles
- `MQANGEL` is running
- The Java option for the angel process is in the active `STDENV` section
- The correct RACF classes were refreshed after updates
- The MQ Web started task was restarted after configuration changes

Common symptoms and causes:

- Login page does not appear: MQ Web did not start, XML is invalid, or HTTPS endpoint configuration is broken
- Login page appears but authentication fails for all users: `APPL` profile, SAF registry, or `profilePrefix` configuration is wrong
- Login works but authorization is wrong: `EJBROLE` definitions or `PERMIT` commands are incomplete
- MQ Web fails after angel changes: `MQANGEL` is not running, `SERVER` profiles are missing, or the Liberty JVM option is not active
- XML edit causes startup failure: the file was not edited in ASCII mode, or a syntax error was introduced

### Cleanup or rollback

If you want to back out the change and return to the basic registry example:

1. Edit `server.xml` and replace:

   ```xml
   <include location="zos_saf_registry.xml"/>
   ```

   with:

   ```xml
   <include location="basic_registry.xml"/>
   ```

2. Remove or ignore the SAF registry XML file if it is no longer needed.

3. Back out the RACF definitions according to your site's standards if they were created only for this lab.

4. Restart the MQ Web Console.

### Appendix

More information about angel RACF profiles: [IBM Documentation](https://www.ibm.com/docs/en/cics-ts/6.x?topic=SSJL4D_6.x/security/java/security_angel.htm)

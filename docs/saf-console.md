# Implementing basic authentication using SAF for the MQ Web Console

#### **Audience level**
Some knowledge of MQ, z/OS, Liberty

#### **Skillset**
MQ Administration, z/OS systems programming

#### **Background**

Basic authentication necessitates the use of a username and password to access a resource. MQ Web Console has two methods of basic authentication: using the basic registry and using the SAF registry. This lab covers using the SAF registry. Today's security requirements have moved away from basic authentication towards the use of tokens and passkeys, so this lab should be used for learning purposes in sandbox or lab environments, not production environments. 

### **Overview of exercise**

I. Copy over the zos_saf_registry.xml
II. Alter the server.xml
III. Customize the zos_saf_registry.xml
IV. Configure RACF
V. Run the angel process
VI. Configure RACF for angel process

### **Exercise** 

#### I.	Copy over the zos_saf_registry.xml

1\. Type **tso omvs** in your command line from the main ISPF main menu

2\. Once in OMVS, type ls /usr/lpp/mqm/V9R4M2X/web/mq/samp/configuration. When you press Enter, you should see a list of the XML files in the directory, including basic_registry.xml.

3\. Change directories to /var/mqm/servers/mqweb, and then execute this command:
```
cp /usr/lpp/mqm/V9R4MX/web/mq/samp/configuration/zos_saf_registry.xml .
```
> Important: Do not miss the last . mark. It specifies you want the copy to be made to the directory you are currently in.

4\. Enter exit in the command line to quit out of OMVS. Back out of the /MQS1/var/mqm/servers/mqweb directory, and then re-enter it. Now, you should see the zos_saf_registry file in the directory! Browse the zos_saf_registry.xml file using VA to look at what credentials users will be able to use for the MQ Console.

#### II. Alter the server.xml

1\. Navigate to the /var/mqm/servers/mqweb directory from option 3.4 in ISPF. In the mqweb directory, several XML files were created. You need to modify these files.

2\. Put an L to the left of the mqweb option to browse its contents. You should see several files. Notice the read-write permissions required here.

3\. Type EA next to zos_saf_registry.xml to open the XML file in edit mode. An EDIT Entry Panel, like the one shown below, will be displayed. Press Enter twice to continue until you see the code in edit mode.

4\. Edit the server.xml file. Replace the line <include location="basic_registry.xml"/> with this line: <include location="zos_saf_registry.xml"/>. Save the XML file, and back out using F3.

#### III. Customize the zos_saf_registry.xml

1\. Navigate to option 3.4 from the ISPF main menu

2\. Navigate to the configuration files for the web server from the Dslist search utility. For example:

```                                                                       
    Enter one or both of the parameters below:                             
    Dsname Level . . . /var/mqm/servers/mqweb                           
    Volume serial  . .       
```

3\. Using F8, scroll to the zos_saf_registry.xml file and using **ea**, enter edit asci mode to modify the XML file.

4\.  Modify the sample configuration to reflect these changes:

```
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

> Note: You will notice we set the profilePrefix variable to MQW942 to correspond with the version of MQ we are running. You will notice we set the unauthenticatedUser to MQGUEST. Please note, the unauthenticatedUser is more like a pre-authenticated user. The purpose of this parameter is to get users to the login screen before checking their credentials.

7\. Back out of the zos_saf_registry.xml and enter the mqwebuser.xml in **ea** mode.

8\. Your mqwebuser.xml should look something like:

```
<server>                                                  
    <featureManager>                                         
        <feature>appSecurity-2.0</feature>                   
    </featureManager>                                        
    <webAppSecurity allowFailOverToBasicAuth="true"/>        
    <webAppSecurity overrideHttpAuthMethod="BASIC"/>         
    <variable name="httpsPort" value="9443"/>                
    <variable name="httpHost" value="-1"/>                   
    <httpEndpoint host="*" httpPort="-1" httpsPost="9443"    
        id="defaultHttpEndpoint"/>                              
    <sslDefault sslRef="mqDefaultSSLConfig"/>                
 </server>                                                  
```

 > Note: You may opt to put the configuration for zos_saf_registry and mqwebuser in one file, but in our example we have both files separate to make it easier to switch between basic_registry and zos_saf_registry.


#### IV. Configure RACF

1\. Now that we have our profilePrefix specified, we need to complete the RACF set up. Navigate to the ISPF main menu, then to option 6 for accessing the TSO command line.

2\. In the command line enter **RDEFINE APPL profilePrefix UACC(NONE)** to define the mqweb server APPLID to RACF (In our case, this APPLID is MQW942).

3\. Enter the command **PERMIT profilePrefix CLASS(APPL) ACCESS(READ) ID(userID)** for each user that requires READ access to the MQWEB server. You must also grant access to the unauthenticatedUser you specified in the zos_saf_registry.xml. This access means they will be able to get to the login screen on a web browser.

4\. Enter the command **SETROPTS RACLIST(APPL) REFRESH** to refresh RACF with these updates.

5\. Next, we want to run the following commands to define the profiles in the EJBROLE class needed to give specific users access to the predefined security roles in the IBM MQ Console and REST API.


```
    RDEFINE EJBROLE profilePrefix.com.ibm.mq.console.MQWebAdmin UACC(NONE)

    RDEFINE EJBROLE profilePrefix.com.ibm.mq.console.MQWebAdminRO UACC(NONE)

    RDEFINE EJBROLE profilePrefix.com.ibm.mq.console.MQWebUser UACC(NONE)

    RDEFINE EJBROLE profilePrefix.com.ibm.mq.rest.MQWebAdmin UACC(NONE)

    RDEFINE EJBROLE profilePrefix.com.ibm.mq.rest.MQWebAdminRO UACC(NONE)

    RDEFINE EJBROLE profilePrefix.com.ibm.mq.rest.MQWebUser UACC(NONE)

    RDEFINE EJBROLE profilePrefix.com.ibm.mq.rest.MFTWebAdmin UACC(NONE)

    RDEFINE EJBROLE profilePrefix.com.ibm.mq.rest.MFTWebAdminRO UACC(NONE)
```


6\. Now we want to specify which users get permission for actually using the predefined security roles in the MQ Console. For example, an MQ administrator should get read access for the EJBROLE AND for the APPLID MQW942.


7\. Run this command as necessary for the appropriate roles and users:


`PERMIT profilePrefix.com.ibm.mq.rest.MQWebAdmin CLASS(EJBROLE) ACCESS(READ) ID(userID)`

#### III. Run the angel process

We will need an angel process run alongside our web server started task. What is that? The Liberty angel process is a started task that allows Liberty servers to use z/OS authorized services. Its long-lived and can be shared among your multiple Liberty servers.

1\. Navigate to SDSF from the ISPF main menu

2\. In the command input line at the top of SDSF, enter the command: **/s mqangel** and press **enter**. For example, it should look like:
    
    COMMAND INPUT ===> /s mqangel 

3\. Now, if you navigate to DA under SDSF and set the prefix to MQ*, you should see the angel task running as the address space MQANGEL  

4\. Now, return to the ISPF main menu. Navigate to option 3.4.

5\. Navigate to SYS1.PROCLIB

6\. From the SYS1.PROCLIB partitioned data set, use the sort command to find MQWEBS. Type e to the left of the MQWEBS member to enter edit mode on the member.

7\. In the JCL, you will notice //STENV and //ZTENV sections. //STENV is getting picked up by the program. ZTENV is not. In general, you can use the STDENV DD statement to specify a z/OS shell script that defines the parameters, environment variables, and other elements needed to start the Java virtual machine (JVM) for an IMS dependent region.

8\. Make sure the line  IBM_JAVA_OPTIONS=-Dcom.ibm.ws.zos.core.angelName=MQANGEL is copied to be in the //STENV section of the JCL after the LIBPATH declaration.

9\. F3 to save and exit the MQWEBS JCL.

#### V. Configure RACF for angel process

At this point, if you try to restart the MQ web console, you will likely be blocked! This is because the web console is running with the userid SYSPROG. We need to give this userid access to the appropriate angel process resources on RACF.

1\. From the ISPF main menu, return to option 6 TSO command line and run the following commands. For the Liberty JVM server to connect to an angel process, create a profile for the angel:

    RDEFINE SERVER BBG.ANGEL UACC(NONE) OWNER(SYSPROG)
    PERMIT BBG.ANGEL CLASS(SERVER) ACCESS(READ) ID(SYSPROG)


2\. Next, define a server profile for the named angel process, corresponding to our profilePrefix:

    RDEFINE SERVER BBG.ANGEL.MQW942 UACC(NONE) OWNER(SYSPROG)
    PERMIT BBG.ANGEL.MQW942 CLASS(SERVER) ACCESS(READ) ID(SYSPROG)


3\. Next, define a server profile for the security profilePrefix (MQW942)

```
RDEFINE SERVER BBG.SECPFX.MQW942 UACC(NONE)
PERMIT BBG.SECPFX.MQW942 CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
```

4\. Define a server profile for the authorized module BBGZSAFM

```
RDEFINE SERVER BBG.AUTHMOD.BBGZSAFM UACC(NONE) OWNER(SYSPROG)
PERMIT BBG.AUTHMOD.BBGZSAFM CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
```

5\. Define a server profile for SAF authorized user registry services and SAF authorization services (SAFCRED)

```
RDEFINE SERVER BBG.AUTHMOD.BBGZSAFM.SAFCRED UACC(NONE)
PERMIT BBG.AUTHMOD.BBGZSAFM.SAFCRED CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
```

6\. Define a server profile and permit WLM services (ZOSWLM)

```
RDEFINE SERVER BBG.AUTHMOD.BBGZSAFM.ZOSWLM UACC(NONE)
PERMIT BBG.AUTHMOD.BBGZSAFM.ZOSWLM CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
```

7\. Define a server profile for RRS transaction services (TXRRS)

```
RDEFINE SERVER BBG.AUTHMOD.BBGZSAFM.TXRRS UACC(NONE)
PERMIT BBG.AUTHMOD.BBGZSAFM.TXRRS CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
```

8\. Define a server profile for SVCDUMP services (ZOSDUMP)

```
RDEFINE SERVER BBG.AUTHMOD.BBGZSAFM.ZOSDUMP UACC(NONE)
PERMIT BBG.AUTHMOD.BBGZSAFM.ZOSDUMP CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
```

9\. Define a server profile for server optimized local adapter services

```
RDEFINE SERVER BBG.AUTHMOD.BBGZSAFM.WOLA UACC(NONE) OWNER(SYSPROG)
PERMIT BBG.AUTHMOD.BBGZSAFM.WOLA CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
RDEFINE SERVER BBG.AUTHMOD.BBGZSAFM.LOCALCOM UACC(NONE) OWNER(SYSPROG)
PERMIT BBG.AUTHMOD.BBGZSAFM.LOCALCOM CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
```

10\. Define a server profile for IFAUSAGE services (PRODMGR)

```
RDEFINE SERVER BBG.AUTHMOD.BBGZSAFM.PRODMGR UACC(NONE) OWNER(SYSPROG)
PERMIT BBG.AUTHMOD.BBGZSAFM.PRODMGR CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
```

11\. Define a server profile for AsyncIO services (ZOSAIO)

```
RDEFINE SERVER BBG.AUTHMOD.BBGZSAFM.ZOSAIO UACC(NONE) OWNER(SYSPROG)
PERMIT BBG.AUTHMOD.BBGZSAFM.ZOSAIO CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
```

12\. Define a server profile for the authorized client module BBGZSCFM

```
RDEFINE SERVER BBG.AUTHMOD.BBGZSCFM UACC(NONE) OWNER(SYSPROG)
PERMIT BBG.AUTHMOD.BBGZSCFM CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
```

13\. Define a server profile for the client optimized local adapter services:

```
RDEFINE SERVER BBG.AUTHMOD.BBGZSCFM.WOLA UACC(NONE) OWNER(SYSPROG)
PERMIT BBG.AUTHMOD.BBGZSCFM.WOLA CLASS(SERVER) ACCESS(READ) ID(SYSPROG)
```
14\. Finally, refresh the in-storage profiles
```
SETROPTS RACLIST(SERVER) REFRESH
```



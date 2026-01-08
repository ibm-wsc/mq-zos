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
III. Run the angel process
II. Run the appropriate RACF commands

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

#### III. Run the angel process

We will need an angel process run alongside our web server started task. What is that? The Liberty angel process is a started task that allows Liberty servers to use z/OS authorized services. Its long-lived and can be shared among your multiple Liberty servers.








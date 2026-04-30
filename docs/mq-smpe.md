# Customizing a new queue manager on IBM MQ for z/OS
### Audience level
Intermediate/Advanced
### Skillset
z/OS Systems Programming, MQ Administration

### Background
The SMP/E process is a standard of installing software on IBM z/OS. MQ is one of multiple software products that takes advantage of SMP/E. This demonstration will show you an example walk through of IBM MQ for z/OS installation. Every environment is different, and this example should be used for educational purposes only.

### Overview of exercise

What needs to be done here: 

1. Retrieve IBM MQ for z/OS 9.4.5 to z/OS from the internet
2. Build SMP/E Environment
3. SMP/E RECEIVE
4. Customize and submit IBM MQ 9.4.1 data set allocation jobs
5. Define the MQ product DDEF entries
6. SMP/E APPLY
7. SMP/E ACCEPT

### Graphic of SMP/E process

[Image of SMP/E process at a high level](assets/SMPE/smpe0.png)

### Demonstration begin

#### I. Retrieve IBM MQ for z/OS 9.4.5 to z/OS from the internet via JCL

1. Each software order will contain SERVINFO DD statement that contains specific information for your order, including credentials and unique identifiers for the product.

2. Use a GIMSMP job to receive software directly from the internet to your global consolidated software inventory (CSI) zone

> NOTE:  CSI is a VSAM data set partitioned into zones, typically including a Global Zone (central repository), Target Zones (for products running on the system), and Distribution Zones (for backup/system maintenance).

2. 

#### I. Retrieve IBM MQ for z/OS 9.4.5 to z/OS from the internet via workstation

1. On shopZ, there are various options for obtaining the media of a z/OS product. Downloading directly to your workstation can be the easiest to avoid having to establish network connections.

> NOTE: Check if this meets your organization's security standards.

2. On your workstation, create a directory structure to accomodate for the format of the SMP/E format. For example,

    ```
    c:\z\shopz\MQ942CD\SMPHOLD
    c:\z\shopz\MQ942CD\SMPPTFIN
    c:\z\shopz\MQ942CD\SMPRELF
    ```

3. Once downloaded, files can be easily moved to a z/OS system using FTP. The package must first be moved to an OMVS file system on z/OS. So, you must have a OMVS file system prepared.

4. Using the mkdir command, create a dedicated mount point in the OMVS root directory in the z/OS system's OMVS directory structure. For example, /u/sysp.

5. Next, mount a very large file system at this mount point.

6. Once, the file system is mounted, create three sub directories to match the SMP/E format:

    ```
    /u/sysp/SMPHOLD
    /u/sysp/SMPPTFIN
    /u/sysp/SMPRELF

> NOTE:
**SMPHOLD** contains exception system modification data, specifically ++HOLD and ++RELEASE statements. A ++HOLD statement is a control record that flags a software modification (SYSMOD, like a PTF) as having a potential issue, preventing its application, while a ++RELEASE statement removes that restriction. Essentially, these statements act as an automated, proactive "stop-and-go" mechanism to ensure software updates don't cause errors or damage the mainframe system.

> NOTE:
**SMPPTFIN** contains Program Temporary Fixes (PTFs), function modification identifiers (FMIDs), and ++ASSIGN statements. ++ASSIGN statements are used to assign specific source identifiers (SOURCEIDs) to SYSMODs (system modifications) during the RECEIVE process. 

> NOTE: 
**SMPRELF** delivers the actual code, macros, and modules of a program being installed via SMP/E. You may also here the contents of SMPRELF referred to simply as REL files

7. Next, using FTP, each file to the z/OS OMVS directory you've created. Ensure the mode is binary mode. For example...

    ```
    ftp zos.image
    cd /u/sysp
    bin
    prompt
    cd SMPHOLD 
    lcd SMPHOLD
    mput *
    cd SMPPTFIN 
    lcd SMPPTFIN
    mput *
    cd SMPRELF 
    lcd SMPRELF
    mput *
    ```

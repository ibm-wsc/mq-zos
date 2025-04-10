# Customizing a new queue manager on IBM MQ for z/OS
### Audience level
Some knowledge of MQ or z/OS 
### Skillset
z/OS Systems Programming, MQ Administration

### Background
Every time a new release of IBM MQ for z/OS is installed, you have the opportunity to create or migrate a new queue manager with the latest capabilities of the IBM MQ release. We will go through the process of creating a new queue manager with IBM MQ for z/OS 9.4.1. IBM MQ for z/OS has been installed on the environment before the lab, so that will installation process will not be in scope of today’s lab. 
To start a new queue manager, JCL procedures need to be copied to a system JCL procedure library and the new queue manager subsystem needs to be defined to MVS. 

### Overview of exercise
What needs to be done here: 
I.	Copy and tailor the sample JCL. Each of these members comes pre-canned in the IBM MQ installation. What your task is as an administrator is to customize these for your specific need.
II.	Run jobs to create the bootstrap data sets and log data sets
III.	Add MSTR and CHIN to SYS1.PROCLIB, the started task library
IV.	Dynamically add MQ subsystem to MVS
V.	Define subsystem security
VI.	Start the queue manager and channel initiator

### Lab begin
#### I. Copy and tailor the sample JCL
1.	We will start with copying the members from the IBM MQ code installation. All the IBM MQ installation will be under the high level qualifier MQ941CD. We are only interested in the sample code here, under SCSQPROC. (*) specifies we want all the members in the SCSQPROC dataset. Go to option =3.3
 
2.	We are making a new queue manager called ZQS3, so we want the new dataset to be referenceable by the high-level qualifier ZQS3. Hit enter.
 
3.	Type a ‘1’ next to option 1. We want the new dataset to have the attribute of MQ941CD.SCSQPROC. Hit enter. 
 
4.	In the top right corner, you should see z/OS confirm that 113 members have been copied to the new dataset you created ZQS3.SCSQPROC. Great! We just need one more thing before we can customize. We are going to steal it from already-existing queue manager ZQS2 in this instance. 

 

5.	QMEDIT is a REXX EXEC that will help us customize our sample code efficiently. We want to name it QMEDIT under our ZQS3 dataset as well. Hit enter and you should see in the top right corner, ‘QMEDIT copied’.
 
6.	Now, from the ISPF main screen, if we enter ‘=3.4’ into the command line and hit enter. We should be able to navigate to our newly created dataset. Copy the below screen and hit enter.
 
7.	Hit enter
8.	Browse the dataset by entering a ‘b’ to the left of the dataset name and hit enter. 
 
9.	We will need to customize the following members of the dataset to effectively create a new queue manager:

    | JCL job    | Description |
    | -------- | ------- | 
    | CSQ4BSDS	| Creates bootstrap data sets |
    | CSQ4CHIN	| Sample Channel Initiator JCL procedure |
    | CSQ4MSTR	| Sample Queue Manager JCL procedure |
    | CSQ4INPX	| Sample commands related to the channel initiator |
    | CSQ4INYG	| Commands to define objects that are normally required |
    | CSQ4PAGE	| Creates page sets for QM storage |
    | CSQ4ZPRM	| Creates the queue manager initiation attributes |

10.	Instead of manually customizing each of these, we will use our QMEDIT to help us customize quickly. Use ‘F8’ to navigate down to QMEDIT from the list of members in ZQS3.SCSQPROC. Place a ‘e’ to the left of QMEDIT and hit enter.
 
11.	Once inside QMEDIT, look through the code and see what the code is customizing. Since this was last used for ZQS2, you will see ZQS2 mentioned a lot. We need to change that.
a.	Enter the command ‘C ‘ZQS2’ ‘ZQS3’ ALL’ on the command line and hit enter. 
b.	Enter the command ‘C ‘MQ933CD’ ‘MQ941CD’ ALL on the command line and hit enter.
c.	Enter the command ‘C ‘1424’ ‘1425’ ALL’ on the command line and hit enter so there isn’t a port number overlap with ZQS2.
12.	By entering the above commands, we’re customizing the REXX exec to the data sets of this particular z/OS environment
 
13.	Now, our REXX exec should be ready to use because it has the correct version of MQ specified, our desired queue manager name, our desired storage areas, and English. Each of those things need to be specified from the original sample code.
14.	We need to activate the QMEDIT code to be able to go through our relevant members and customize them quickly. Use F3, return to the ISPF main menu and enter option 6. Enter this command:

 
15.	Hit enter. With ALTLIB, a user or ISPF application can easily activate and deactivate CLIST and REXX exec libraries as the need arises. We are activating the REXX exec library of QMEDIT here to enable customization.
16.	From the ISPF main menu, navigate back to the ZQS3.SCSQPROC members via option 3.4.
17.	Starting with CSQ4BSDS, we will customize:
a.	CSQ4BSDS
b.	CSQ4CHIN
c.	CSQ4MSTR
d.	CSQ4INPX
e.	CSQ4INYG
f.	CSQ4PAGE
g.	CSQ4ZPRM
18.	Enter an ‘e’ next to CSQ4BSDS and hit enter from the member list. Once inside CSQ4BSDS, enter QMEDIT on the command input line and hit enter.
 
> NOTE: You must execute QMEDIT in the same session as you executed the ALTLIB command from. Otherwise, you will get the error message ‘QMEDIT not found’.

19.	You should notice the changes by looking through CSQ4BSDS, using F7 and F8 to navigate up and down the JCL code. Enter F3 to return to the member list and save your changes to CSQ4BSDS.
20.	Now, navigate back to the ZQS3.SCSQPROC list. Repeat this process for:
a.	CSQ4CHIN
b.	CSQ4MSTR
c.	CSQ4INPX
d.	CSQ4INYG
e.	CSQ4PAGE
f.	CSQ4ZPRM
21.	We are going to make an additional customization on CSQ4PAGE. Navigate to the member using ‘e’ to the left of the member. Once inside, enter the following commands on the command line at the bottom and hit enter. 
a.	c 'VOL=SER' 'STORCLAS' all
b.	c 'VOLUMES' 'STORCLAS' all
22.	We’re using system managed storage devices now instead of volumes, as is the standard on z/OS now
23.	F3 to save the changes. We have to customize the storage here to be appropriate for this z/OS image.
24.	Next, we’re going to modify CSQ4ZPRM. Use QMEDIT like normal, then enter the command: c ‘++HLQ.USERAUTH++’ ‘ZQS1.USERAUTH’ ALL
25.	Last, modify CSQ4CHIN using ‘e’. Use QMEDIT like normal, then we are going to make one more additional customization on CSQ4CHIN. Once inside CSQ4CHIN, enter the command ‘f user exit library’. This will find the appropriate JCL.
 
26.	We want to comment the lines 94 and 119 out. Insert a “*” in the front of the line so that the asterisk lines up with the asterisk on the line below. It should look like this: 

 
27.	F3 out of CSQ4CHIN to save your changes and return to the member list. 
28.	You can enter the command ‘SORT CHANGED’ from the member list panel to ensure you customized all the essential members
 


#### II. Run jobs to create the bootstrap data sets and log data sets
29.	Now, your customization is complete. Enter ‘e’ next to CSQ4BSDS and input ‘SUBMIT’ on the command line. This will create our bootstrap data sets for the new queue manager. 
30.	Repeat this for CSQ4PAGE to set up the page sets for the new queue manager. 
 
31.	Last, submit CSQ4ZPRM using the same process as CSQ4BSDS and CSQ4PAGE
32.	When you return to the main ISPF menu, use option 3.4 to navigate to all the ZQS3 libraries
 
33.	Once you hit enter, you should now see boot strap data set and page set files set up along with our original ZQS3.SCSQPROC data set. If you do not see the new data sets, something has failed in your JCL and you will need to debug. We recommend comparing the BSDS and PAGE JCL to the JCL of a working queue manager, for example, ZQS1.
III. Add MSTR and CHIN to SYS1.PROCLIB, the started task library

34.	Now, we have to edit SYS1.PROCLIB. Navigate to SYS1.PROCLIB using 3.4 on the ISPF menu. SYS1.PROCLIB needs to contain two members for ZQS3, ZQS3MSTR and ZQS3CHIN. We can add these two members by copying our CSQ4MSTR and CSQ4CHIN and renaming them.
35.	From the ISPF main menu, go to 3.3. Here, specify that you would like to copy from ‘ZQS3.SCSQPROC(CSQ4MSTR)’ to ‘SYS1.PROCLIB(ZQS3MSTR)’. This will create a copy of your edited member for SYS1.PROCLIB and it will also rename the member to ZQS3MSTR.
 
 
36.	Repeat this copying process for CSQ4CHIN i.e. ‘ZQS3.SCSQPROC(CSQ4CHIN)’ to ‘SYS1.PROCLIB(ZQS3CHIN)’.
37.	Now, if you navigate to SYS1.PROCLIB using option 3.4, you should see the members ZQS3MSTR and ZQS3CHIN listed as members.
 
#### IV. Dynamically add MQ subsystem to MVS
38.	Now, all the setup is complete, so we just have to start up the queue manager!
39.	The next few commands will all be entered in the MVS command area. Navigate there by entering ‘D’ in the ISPF menu command line to navigate to SDSF. Once in SDSF, enter a slash in your command input and hit enter like so:
40.	Execute command to dynamically define MQ subsystem: 
a.	SETSSI ADD,S=ZQS3,I=CSQ3INI,P='CSQ3EPX,ZQS3,S'
 
> NOTE! None of these dynamic commands will last through an IPL of the system. To make these changes concrete, you will need to modify the LPALST##, IEFSSN## and PROG## members of the LPAR’s SYS1.PARMLIB data set.

#### V. Define subsystem security
41.	F3 back to the main menu, out of SDSF, enter option 6 from the main menu. Here, you will find a TSO command input window:
a.	Turn off security by entering this command: 
 
NOTE! Never turn off security outside of workshop lab environments.

42.	You will see an output like this, indicating this has already been done for you, but this enables you to see how we did it. Obviously, you will not be disabling security in any of your environments, just in our test environment.  

#### VI. Starting your queue manager and channel initiator

43.	Return to the SDSF command window and input the commands into the MVS command line:

a.	Start up our queue manager ZQS3 with the command: ZQS3 START QMGR
b.	Start up the channel initiator with the command: ZQS3 START CHINIT
c.	Start up the listener with the command: ZQS3 start listener TRPTYPE(TCP) Port(1425)
44.	To verify that your queue manager has been set up, you can navigate to MQ Explorer and test the connection. You will use the port number you specified in the REXX exec.
45.	Congrats! You have created a queue manager from scratch! Lab COMPLETE!

### Appendix:
• REXX EXEC is not included with the base product – describe ++ variables 

• Make an error when executing your SETSSI command? Use SETSSI DELETE,S=ZQS3,FORCE to roll back your command.

• Check APF authorized libraries by entering the command /DISPLAY PROG,APF from the SDSF command input then going to the log. 

    APF authorized libraries must be:
        o	MQ941CD.SCSQANLE
        o	MQ941CD.SCSQAUTH
        o	MQ941CD.SCSQMVR1
        •	You may see several LPALST## PROG##, and IEFSSN## members. You want to use the ones specified in the SYS1.PARMLIB(IEASYS##). You can find the IEASYS## member by entering the command /D IPLINFO from the SDSF command input. It will show a screen like this:
 
• Looking to permanently make updates to your LPALST## member?

    o	Add code like this:
    
    •	Looking to permanently make updates to your IEFSSN## member?

    a.	Add code like this:
    
    •	Looking to permanently make updates to your PROG## member?

    o	Add code like this:
    
    • Need to dynamically APF authorize your MQ load libraries?
        o	SETPROG APF,ADD,DSNAME=MQ941CD.SCSQANLE ,SMS   
        o	SETPROG APF,ADD,DSNAME=MQ941CD.SCSQSNLE ,SMS  

    • Need to dynamically add some modules to the LPA (link pack area) of z/OS?
        o	SETPROG LPA,ADD,MODNAME=(CSQ3EPX,CSQ3INI),DSNAME=MQ941CD.SCSQLINK
        o	SETPROG LPA,ADD,MODNAME=(CSQ3ECMX),DSNAME=MQ941CD.SCSQSNLE



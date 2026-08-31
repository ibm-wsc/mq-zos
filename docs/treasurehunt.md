# IBM MQ for z/OS — Treasure Hunt

A hands-on exploration of your sandbox queue manager environment.

**Welcome, explorer.** Each clue below sends you somewhere real in your z/OS sandbox. You'll use the **ISPF Operations & Control Panels**, the **z/OS console**, the **CSQUTIL** batch utility, and raw **MQSC commands** to find answers. Work through the clues in order — each one builds on the previous. Good luck.

**Difficulty key:** 🟢 Easy (observation only) · 🟡 Medium (requires a command) · 🔴 Hard (requires problem-solving)

---

## Part 1 — Finding Your Bearings

### Clue 1 — Who's in charge here? 🟢
> Tags: `ISPF Panels` · `DISPLAY QMGR`

Every z/OS MQ environment starts with a queue manager. Before you can do anything, you need to know what you're working with.

**Your task:** Open the IBM MQ Operations & Control Panels from ISPF (exec `CSQOREXX`) and navigate to the queue manager. Then, from the CSQUTIL COMMAND function or the z/OS console, run:

```mqsc
DISPLAY QMGR DESCR CCSID DEADQ MAXMSGL PLATFORM VERSION
```

Write down the queue manager name, its **dead-letter queue** name (`DEADQ`), and the MQ version number.

> **Hint:** On the z/OS console, prefix MQSC commands with your queue manager's command prefix, e.g. `/+CSQ1 DISPLAY QMGR ALL`. In CSQUTIL JCL, you put the MQSC text in a DD statement and run the COMMAND function.
> 📖 [Administering IBM MQ for z/OS](https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=administering-mq-zos)

**Record:** queue manager name · DEADQ name · VERSION reported

---

### Clue 2 — What queues live here? 🟢
> Tags: `DISPLAY QUEUE` · `SYSTEM queues`

A newly started queue manager comes with a full set of predefined `SYSTEM.*` queues. Let's see what's there — and count them.

**Your task:** Run the following command and count how many queues are returned:

```mqsc
DISPLAY QUEUE(SYSTEM.*) TYPE(QLOCAL)
```

Then find the queue used for inbound MQSC commands and the queue used for command responses. What are their names?

> **Hint:** Look for queues with names like `SYSTEM.COMMAND.*`. The *system-command input queue* is where authorized applications send commands as messages.
> 📖 [MQSC Commands Reference](https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=reference-mqsc-commands)

**Record:** total `SYSTEM.*` local queue count · `COMMAND.INPUT` name · `COMMAND.REPLY` name

---

## Part 2 — Digging into Queues

### Clue 3 — Define your own queue 🟡
> Tags: `DEFINE QLOCAL` · `DISPLAY QSTATUS`

Time to make your mark. Define a local queue that you'll use as a personal scratchpad for the rest of this hunt.

**Your task:**

```mqsc
-- Replace HUNT with your initials, e.g. HUNT.DQ.TEST
DEFINE QLOCAL(HUNT.<INITIALS>.TEST) +
  DESCR('Treasure hunt scratch queue') +
  MAXDEPTH(100) +
  MAXMSGL(4096) +
  DEFPSIST(YES)
```

After defining it, confirm it exists:

```mqsc
DISPLAY QLOCAL(HUNT.*) DESCR MAXDEPTH CURDEPTH DEFPSIST
```

What is the default persistence setting you used, and what does it mean for messages on this queue?

> **Hint:** `DEFPSIST(YES)` means messages default to persistent storage — they survive a restart.
> 📖 [IBM MQ Objects](https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=overview-mq-objects)

**Record:** exact queue name created · DEFPSIST value · what MAXDEPTH controls

---

### Clue 4 — The dead-letter queue — what goes there? 🟡
> Tags: `Dead-letter queue` · `DISPLAY USAGE` · `CSQUDLQH`

You found the dead-letter queue name in Clue 1. Now investigate it properly.

**Your task:** Run these two commands against the dead-letter queue name you found:

```mqsc
DISPLAY QLOCAL(<DLQ-NAME>) ALL
DISPLAY QSTATUS(<DLQ-NAME>) CURDEPTH IPPROCS OPPROCS
```

Answer these questions:

1. What is the current depth (`CURDEPTH`) of the dead-letter queue?
2. What is the maximum message length (`MAXMSGL`) it accepts?
3. What z/OS utility is provided to drain and process the dead-letter queue automatically?

> **Hint:** The utility that processes the dead-letter queue is `CSQUDLQH`.
> 📖 [IBM MQ Utilities on z/OS Reference](https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=reference-mq-utilities-zos)

**Record:** CURDEPTH · MAXMSGL · name of the DLQ handler utility

---

## Part 3 — Channels & Connectivity

### Clue 5 — Channels: who can connect? 🟡
> Tags: `DISPLAY CHANNEL` · `DISPLAY CHINIT`

Channels are how queue managers talk to the outside world. Let's see what's defined.

**Your task:**

```mqsc
-- List all channels of all types
DISPLAY CHANNEL(*) TYPE(ALL) CHLTYPE CONNAME TRPTYPE

-- Check whether the channel initiator is running
DISPLAY CHINIT
```

Answer these questions:

1. How many channels are defined?
2. Is there a **SVRCONN** (server-connection) channel? What is it named?
3. Is the channel initiator active? How many dispatchers are running?

> **Hint:** `DIS CHL(*)` is the short form of `DISPLAY CHANNEL(*)`. `DISPLAY CHINIT` (synonym: `DIS CHI`) shows channel initiator status including listeners and active connections.
> 📖 [Monitoring and controlling channels on z/OS](https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=zos-monitoring-controlling-channels)

**Record:** channel count · SVRCONN channel name (if any) · channel initiator dispatcher count

---

### Clue 6 — Start a listener, check the status 🟡
> Tags: `DEFINE LISTENER` · `START LISTENER` · `DISPLAY CHSTATUS`

A listener sits on a TCP port and accepts incoming channel connections. Let's define one and observe it.

**Your task:**

```mqsc
-- Define a listener on port 1415
DEFINE LISTENER(HUNT.LISTENER) +
  TRPTYPE(TCP) +
  PORT(1415) +
  CONTROL(QMGR)

-- Start it
START LISTENER(HUNT.LISTENER)

-- Check it appeared in CHINIT output
DISPLAY CHINIT
```

What does `CONTROL(QMGR)` mean for the listener's lifecycle?

> **Hint:** `CONTROL(QMGR)` means the listener starts and stops automatically with the queue manager.
> 📖 [Monitoring and controlling channels on z/OS](https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=zos-monitoring-controlling-channels)

**Record:** port used · what `CONTROL(QMGR)` does · listener status shown in `DISPLAY CHINIT`

---

## Part 4 — Storage: Page Sets & Buffer Pools

### Clue 7 — Under the hood: page sets 🟡
> Tags: `DISPLAY USAGE` · `Buffer pools` · `Page sets`

Unlike multiplatform MQ, IBM MQ for z/OS stores messages in **page sets**, managed through **buffer pools** in virtual storage. Understanding this is fundamental.

**Your task:**

```mqsc
-- Display page set usage and their associated buffer pools
DISPLAY USAGE TYPE(PAGESET)
```

Answer these questions:

1. How many page sets are defined?
2. Page set 0 (PSID 0) has a special purpose — what is it used exclusively for?
3. What MQSC command would you use to dynamically resize a buffer pool without restarting?

> **Hint:** PSID 0 holds object definitions (queues, channels, etc.), not messages. Best practice is to keep it in its own buffer pool. The command to resize dynamically is `ALTER BUFFPOOL`.
> 📖 [Buffers and buffer pools for IBM MQ for z/OS](https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=zos-buffers-buffer-pools-mq)

**Record:** page set count · purpose of PSID 0 · command to resize a buffer pool dynamically

---

## Part 5 — Running Commands in Batch with CSQUTIL

### Clue 8 — Batch MQSC with CSQUTIL 🔴
> Tags: `CSQUTIL` · `JCL` · `COMMAND function`

The CSQUTIL batch utility is the workhorse of z/OS MQ administration. It lets you run MQSC commands from a JCL job and capture output to SYSPRINT. Write a JCL job that uses the COMMAND function.

**Your task:** Allocate a new sequential data set (or a PDS member) containing these MQSC commands:

```mqsc
DISPLAY QMGR VERSION DEADQ CCSID
DISPLAY QLOCAL(HUNT.*) CURDEPTH MAXDEPTH
DISPLAY CHANNEL(*) CHLTYPE
```

Then submit JCL that calls CSQUTIL with the COMMAND function pointing to that data set:

```jcl
//MQHUNT  JOB ...
//STEP1   EXEC PGM=CSQUTIL,PARM='<QMGRNAME>'
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
  COMMAND DDNAME(MQCMDS)
/*
//MQCMDS  DD DSN=<your.data.set>,DISP=SHR
```

Check SYSPRINT for the output. What return code did the job end with?

> **Hint:** RC=0 means success. The COMMAND function sends each MQSC statement to the `SYSTEM.COMMAND.INPUT` queue and prints the response.
> 📖 [Using the CSQUTIL Utility for IBM MQ for z/OS](https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=utilities-using-csqutil-utility-mq-zos) · [Invoking the IBM MQ Utility Program on z/OS](https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=zos-invoking-mq-utility-program)

**Record:** job return code · SYSPRINT output snippet confirming commands ran · any warnings

---

## Part 6 — The Final Challenge

### Clue 9 — The log — your safety net 🔴
> Tags: `DISPLAY LOG` · `BSDS` · `Recovery`

The MQ for z/OS active log is your guarantee that nothing is lost. Let's inspect it.

**Your task:**

```mqsc
DISPLAY LOG
```

From the output, answer:

1. What is the name of the **Bootstrap Data Set (BSDS)**? (Look for it in the console output or initialization messages.)
2. Is log compression active?
3. What utility would you use to print the log map and see all active/archive log data sets?

> **Hint:** The BSDS is a VSAM data set that acts as an index of all MQ log data sets. The utility for printing the log map is `CSQJU004`.
> 📖 [DISPLAY LOG on z/OS](https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=reference-display-log-display-log-information-zos) · [Managing IBM MQ Resources on z/OS](https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=zos-managing-mq-resources)

**Record:** BSDS data set name · log compression status · name of the log map utility

---

### Clue 10 — Tidy up — the responsible explorer 🟡
> Tags: `DELETE QLOCAL` · `STOP LISTENER` · `DELETE LISTENER`

A good administrator always cleans up after themselves. Remove the objects you created during this hunt.

**Your task:**

```mqsc
-- Stop and delete your listener
STOP LISTENER(HUNT.LISTENER)
DELETE LISTENER(HUNT.LISTENER)

-- Delete your test queue
DELETE QLOCAL(HUNT.<INITIALS>.TEST)

-- Verify they are gone
DISPLAY LISTENER(HUNT.*) IGNSTATE(YES)
DISPLAY QLOCAL(HUNT.*)
```

What error message do you get when trying to DISPLAY a queue that no longer exists? What does that message number start with?

> **Hint:** IBM MQ for z/OS messages start with the prefix `CSQ`. A "not found" condition will produce a message in the `CSQM...` range. Check your console output or SYSPRINT carefully.
> 📖 [Administering IBM MQ for z/OS](https://www.ibm.com/docs/en/ibm-mq/10.0.x?topic=administering-mq-zos)

**Record:** the `CSQ`-prefixed message ID you received · confirmation that cleanup was successful

---

## 🏆 Hunt Complete!

If you've answered all 10 clues, you've touched the key pillars of IBM MQ for z/OS administration: the queue manager, queues, channels, the channel initiator, page sets, buffer pools, the log, the BSDS, CSQUTIL batch administration, and the ISPF Operations & Control Panels. You're ready to go deeper.

---

## Quick Reference — Key Commands Used

| Command | Short Form | What it does |
|---|---|---|
| `DISPLAY QMGR` | `DIS QMGR` | Show queue manager attributes |
| `DISPLAY QLOCAL` | `DIS QL` | Show local queue definitions |
| `DISPLAY QSTATUS` | `DIS QS` | Show live queue status (depth, open handles) |
| `DISPLAY CHANNEL` | `DIS CHL` | Show channel definitions |
| `DISPLAY CHINIT` | `DIS CHI` | Show channel initiator status |
| `DISPLAY USAGE TYPE(PAGESET)` | `DIS USAGE` | Show page set and buffer pool usage |
| `DISPLAY LOG` | `DIS LOG` | Show active log parameters (z/OS only) |
| `ALTER BUFFPOOL` | — | Dynamically resize a buffer pool |

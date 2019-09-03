# CATMAP

**CATMAP** is a series of tools to be used on z/OS to map out the various datasets on the system. 

## *catmap.rexx* 

the original tool! A rexx script to map out the datasets on the system. 

**Usage:**

```
Arguments:
  -h this help
  -p display product info based on HLQ
  -b brutal mode (gets PDS/PDSE member listings)
  -verbose enables verbose mode
  -f <dataset name> saves output to a file
      (-vol <volume>) Volume for dataset - optional
      (-space <# cylinders>) size of dataset in cylinders
                             100 cyls = 59 MB - optional
```

## *CATMAP3.hlasm*

The new and improved CATMAP. Whats new?

- Written in HLASM so its FAST
- Uses `IGGCSI00` (you'll need access to that to use this tool, usually located in `SYS1.LINKLIB`)
- Lists every non-vsam file 
- ***NEW*** Uses `RACROUTE` to check your access to every DATASET/PDS

This tool *requires* TSO to use, since it uses `TPUT` to output to the screen. For a batch (i.e. JCL) version see CATMAP3J.

Added is using `RACROUTE` to check your access. This couldn't be done in the REXX version without using RACF/ACF2/TopSecret commands. This tool is ESM agnostic and will work in all three environments.

To compile this you can use the following JCL (make sure you change up the JOBCARD though (thats the first three lines). Just put the source in some PDS and call it CATMAP3. This compiles it with symbols, etc for debugging. 

```jcl
//COMPILEG JOB (ASSY),'CATMAP3',CLASS=A,MSGCLASS=Y,
//         NOTIFY=&SYSUID,MSGLEVEL=(1,1)
//         SET FILE=CATMAP3
//ASM      EXEC PROC=HLASMCL,PARM.L=(TEST),PARM.C=(TEST)
//SYSIN    DD   DSN=<some pds>(&FILE),DISP=(SHR)
//L.SYSLMOD DD DSN=<some pds.bin>(&FILE),DISP=(SHR)
//L.SYSLIN DD
//         DD DSN=SYS1.LINKLIB(IGGCSI00),DISP=SHR
//L.SYSPRINT  DD SYSOUT=*
//
```

If the JCL fails, you can use the foreground compiler to compile it and link it for you. Use ISPF option 4 to see forground tools. 

Once compiled you can run it with `CALL <some pds.bin>(CATMAP3)` assuming `some pds.bin` is your load library. E.G. `CALL SOF.LOAD(CATMAP3)`.

## *CATMAP3J.hlasm*

This version is EXACTLY the same as **CATMAP3** except it uses batch to output the results to files instead of printing it to the screen. Compiling it is the same as above. To execute it you can use the following JCL but make sure you change the job card:

```jcl
//CATMAP3J JOB (ASSY),'CATMAPJCL',CLASS=A,MSGCLASS=Y,
//         NOTIFY=&SYSUID,MSGLEVEL=(1,1)
//CATMAP   EXEC PGM=CATMAP3J
//STEPLIB  DD DSN=<where CATMAP3J binary lives>,DISP=SHR
//SYSUDUMP DD SYSOUT=*
//SYSOUT   DD SYSOUT=*
```

## *CATMAP3N.hlasm*

This version adds networking capabilities to CATMAP3. Why? Because frequently the amount of data generates is in the hundreds of megs. And the job can take a while to run. Passing it over the network make it faster to get the information you need. 

Compiling this program is different from the previous two, you need to specify the library where it can find some of the programs we use, specify `IGGCSI00` and `EZACIC04` are used and typical reside (but not always) in `SYS1.LINKLIB` and `TCPIP.SEZACMAC`.

```jcl
//COMPILEL JOB (SOF),'Compile and Link',CLASS=A,MSGCLASS=Y,
//         NOTIFY=&SYSUID,MSGLEVEL=(1,1)
//         SET FILE=CATMAP3N
//ASSEMBLY EXEC PGM=ASMA90,PARM=OBJECT
//SYSPRINT DD  SYSOUT=A
//SYSTERM  DD  SYSOUT=A
//SYSIN    DD  DSN=<some folder/pds>(&FILE),DISP=SHR
//SYSLIB   DD  DSN=SYS1.MACLIB,DISP=SHR
//         DD  DSN=TCPIP.SEZACMAC,DISP=SHR
//SYSUT1   DD  DSN=&&SYSUT1,SPACE=(16384,(120,120),,,ROUND),
//             UNIT=SYSALLDA,BUFNO=1
//SYSLIN   DD  DSN=&&OBJ,SPACE=(3040,(40,40),,,ROUND),
//             UNIT=SYSALLDA,DISP=(MOD,PASS),
//             BLKSIZE=3040,LRECL=80,RECFM=FB,BUFNO=1
//LINK     EXEC PGM=HEWL,PARM='MAP,LET,LIST',COND=(8,LT,ASSEMBLY)
//*
//SYSLIN   DD  DSN=&&OBJ,DISP=(OLD,DELETE)
//         DD  DDNAME=SYSIN
//SYSLMOD  DD DSN=<some folder/pds>(&FILE),DISP=SHR
//SYSUT1   DD  DSN=&&SYSUT1,SPACE=(1024,(120,120),,,ROUND),
//             UNIT=SYSALLDA,BUFNO=1
//SYSLIB   DD  DSN=SYS1.LINKLIB,DISP=SHR
//         DD  DSN=TCPIP.SEZATCP,DISP=SHR
//SYSPRINT DD  SYSOUT=*
//*
```

To use this version you'll need to change line 360 to add your IP address. Follow the comments how to do that. Then open up a listener on port 4444 on your machine and run the program either in `IKJEFT01` or TSO with `CALL <some folder>(CATMAP3N)`. 

### CATMAP2

What happened to CATMAP2? It's garbage and basically didn't work, so we skipped over it. 

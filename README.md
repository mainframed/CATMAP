# CATMAP

CATMAP is a collection of z/OS tools that walks the master catalog and
connected user catalogs to map the system's non-VSAM data sets.

## Implementations

- `catmap.rexx` — the original TSO REXX implementation.
- `CATMAP3.hlasm` — HLASM implementation that writes to the TSO terminal.
- `CATMAP3J.hlasm` — batch HLASM implementation that writes to `CATOUT`.
- `CATMAP3N.hlasm` — networked HLASM implementation that sends ASCII over TCP.

The three CATMAP3 variants share the same catalog and access-checking logic:

1. Obtain the master catalog from z/OS control blocks.
2. Use the Catalog Search Interface (`IGGCSI00`) to discover user catalogs.
3. Search each catalog for non-VSAM data sets and their volume serials.
4. Use `RACROUTE REQUEST=AUTH,STATUS=ACCESS` to report the caller's access.

The catalog table is allocated dynamically, the CSI work area is reused, and
individual catalog errors do not stop the complete scan. CSI entries marked as
errors or without returned field data are skipped.

## Output format

CATMAP3 output records are 80 bytes:

```text
ACCESS  VOLSER DATA.SET.NAME
```

The fields are seven bytes of access, one blank, six bytes of VOLSER, one
blank, 44 bytes of data set name, and 21 trailing blanks. Possible access
values include `NONE`, `READ`, `UPDATE`, `CONTROL`, `ALTER`, `NOTPROT`, and
`ERROR`.

CATMAP3J uses `RECFM=FB,LRECL=80`, allowing `CATOUT` to reference a standard
FB 80 sequential data set or a member of an FB 80 PDS:

```jcl
//CATOUT DD DSN=YOUR.CATMAP.OUTPUT,DISP=OLD
```

```jcl
//CATOUT DD DSN=YOUR.FB80.PDS(CATMAP),DISP=SHR
```

CATMAP3N sends each 80-byte record followed by an ASCII line feed.

## catmap.rexx

The original REXX implementation uses TSO commands such as `LISTCAT`,
`LISTDSI`, and `LISTDS`.

```text
Arguments:
  -h                 display help
  -p                 display product information based on HLQ
  -b                 list PDS/PDSE members ("brutal" mode)
  -verbose           enable verbose output
  -f <data set>      save output to a new data set
  -vol <volume>      output volume (optional)
  -space <cylinders> output allocation size (optional)
```

Brutal mode can generate large numbers of access violations. Use it with
care.

## CATMAP3 (TSO)

CATMAP3 requires a TSO environment because it uses `TPUT`:

```text
CALL 'YOUR.LOADLIB(CATMAP3)'
```

Use `compile_catmap3.jcl` to assemble and link it. Update its job card and data
set names for your environment. The link-edit must include `IGGCSI00`, normally
from `SYS1.LINKLIB`.

The following is a complete assemble-and-link example:

```jcl
//CATMAP3C JOB (ACCT),'COMPILE CATMAP3',CLASS=A,MSGCLASS=Y,
//         NOTIFY=&SYSUID,MSGLEVEL=(1,1)
//ASSEMBLY EXEC PGM=ASMA90,PARM='OBJECT,NODECK'
//SYSPRINT DD  SYSOUT=*
//SYSTERM  DD  SYSOUT=*
//SYSIN    DD  DSN=YOUR.SOURCE(CATMAP3),DISP=SHR
//SYSLIB   DD  DSN=SYS1.MACLIB,DISP=SHR
//SYSUT1   DD  UNIT=SYSALLDA,SPACE=(CYL,(1,1))
//SYSLIN   DD  DSN=&&OBJ,DISP=(NEW,PASS),
//             UNIT=SYSALLDA,SPACE=(CYL,(1,1)),
//             DCB=(RECFM=FB,LRECL=80,BLKSIZE=3200)
//LINK     EXEC PGM=HEWL,PARM='MAP,LET,LIST',
//             COND=(8,LT,ASSEMBLY)
//SYSLIN   DD  DSN=&&OBJ,DISP=(OLD,DELETE)
//         DD  DDNAME=SYSIN
//SYSLMOD  DD  DSN=YOUR.LOAD(CATMAP3),DISP=SHR
//SYSUT1   DD  UNIT=SYSALLDA,SPACE=(CYL,(1,1))
//SYSLIB   DD  DSN=SYS1.LINKLIB,DISP=SHR
//SYSPRINT DD  SYSOUT=*
//SYSIN    DD  *
 INCLUDE SYSLIB(IGGCSI00)
 NAME CATMAP3(R)
/*
```

It can also be called from a batch TSO step:

```jcl
//CATMAP3R JOB (ACCT),'RUN CATMAP3',CLASS=A,MSGCLASS=Y,
//         NOTIFY=&SYSUID
//TSO      EXEC PGM=IKJEFT01,DYNAMNBR=20
//STEPLIB  DD  DSN=YOUR.LOAD,DISP=SHR
//SYSTSPRT DD  SYSOUT=*
//SYSTSIN  DD  *
 CALL 'YOUR.LOAD(CATMAP3)'
/*
```

## CATMAP3J (batch)

CATMAP3J writes its results through the `CATOUT` DD. Use
`compile_catmap3j.jcl` to build it and `run_catmap3j.jcl` as an execution
example. To build it manually, use the CATMAP3 compile example above and
replace every `CATMAP3` member or module name with `CATMAP3J`.

```jcl
//CATMAPJR JOB (ACCT),'RUN CATMAP3J',CLASS=A,MSGCLASS=Y,
//         NOTIFY=&SYSUID
//CATMAP   EXEC PGM=CATMAP3J
//STEPLIB  DD  DSN=YOUR.LOAD,DISP=SHR
//SYSUDUMP DD  SYSOUT=*
//CATOUT   DD  DSN=YOUR.CATMAP.OUTPUT,
//             DISP=(NEW,CATLG,DELETE),
//             UNIT=SYSALLDA,SPACE=(CYL,(2,2),RLSE),
//             DCB=(RECFM=FB,LRECL=80,BLKSIZE=0)
```

To write to an existing FB 80 PDS member instead:

```jcl
//CATOUT   DD  DSN=YOUR.FB80.PDS(CATMAP),DISP=SHR
```

`CATMAP3J.clist` allocates an FB 80 sequential output data set, then calls
`&SYSUID..CATMAP.LOADLIB(CATMAP3J)`. Output defaults to
`userid.CATMAP.OUTPUT`; use `HLQ(prefix)` to select another output prefix.
The output data set must not already exist.

## CATMAP3N (TCP)

CATMAP3N accepts two required parameters:

1. An IPv4 address in dotted-decimal form (`###.###.###.###`).
2. An integer port from 1 through 65535.

Host names and IPv6 addresses are not accepted.

Start a listener on a host reachable from z/OS:

```sh
ncat -nlvp 4444
```

Then call CATMAP3N from TSO:

```text
CALL 'YOUR.LOADLIB(CATMAP3N)' '192.0.2.10 4444'
```

Invalid parameters return code 8. Network initialization or connection errors
return code 12.

Use `compile_catmapn.jcl` to build CATMAP3N. In addition to the normal HLASM
macros and `IGGCSI00`, it requires the z/OS Communications Server socket
macros and conversion routine, commonly found in:

- `TCPIP.SEZACMAC`
- `TCPIP.SEZATCP`

The complete network build differs by adding the TCP/IP macro and link
libraries:

```jcl
//CATMAPNC JOB (ACCT),'COMPILE CATMAP3N',CLASS=A,MSGCLASS=Y,
//         NOTIFY=&SYSUID,MSGLEVEL=(1,1)
//ASSEMBLY EXEC PGM=ASMA90,PARM='OBJECT,NODECK'
//SYSPRINT DD  SYSOUT=*
//SYSTERM  DD  SYSOUT=*
//SYSIN    DD  DSN=YOUR.SOURCE(CATMAP3N),DISP=SHR
//SYSLIB   DD  DSN=SYS1.MACLIB,DISP=SHR
//         DD  DSN=TCPIP.SEZACMAC,DISP=SHR
//SYSUT1   DD  UNIT=SYSALLDA,SPACE=(CYL,(1,1))
//SYSLIN   DD  DSN=&&OBJ,DISP=(NEW,PASS),
//             UNIT=SYSALLDA,SPACE=(CYL,(1,1)),
//             DCB=(RECFM=FB,LRECL=80,BLKSIZE=3200)
//LINK     EXEC PGM=HEWL,PARM='MAP,LET,LIST',
//             COND=(8,LT,ASSEMBLY)
//SYSLIN   DD  DSN=&&OBJ,DISP=(OLD,DELETE)
//         DD  DDNAME=SYSIN
//SYSLMOD  DD  DSN=YOUR.LOAD(CATMAP3N),DISP=SHR
//SYSUT1   DD  UNIT=SYSALLDA,SPACE=(CYL,(1,1))
//SYSLIB   DD  DSN=SYS1.LINKLIB,DISP=SHR
//         DD  DSN=TCPIP.SEZATCP,DISP=SHR
//SYSPRINT DD  SYSOUT=*
//SYSIN    DD  *
 INCLUDE SYSLIB(IGGCSI00)
 NAME CATMAP3N(R)
/*
```

`run_catmap3n.jcl` contains a batch TSO example. The equivalent JCL is:

```jcl
//CATMAPNR JOB (ACCT),'RUN CATMAP3N',CLASS=A,MSGCLASS=Y,
//         NOTIFY=&SYSUID
//TSO      EXEC PGM=IKJEFT01,DYNAMNBR=20
//STEPLIB  DD  DSN=YOUR.LOAD,DISP=SHR
//SYSTSPRT DD  SYSOUT=*
//SYSTSIN  DD  *
 CALL 'YOUR.LOAD(CATMAP3N)' '192.0.2.10 4444'
/*
```

## ACF2 note

The SAF interface is portable, but `STATUS=ACCESS` has an ACF2-specific
authorization requirement. ACF2 treats this request as an unaudited
interrogation of security definitions and can issue system abend S047 when a
non-APF-authorized program makes the call.

No S047 recovery handler is currently implemented in CATMAP. On ACF2 systems,
the security administrator can define a narrowly scoped GSO SAFDEF with
`NOAPFCHK` for the CATMAP program and its
`REQUEST=AUTH,CLASS=DATASET,STATUS=ACCESS` call. The other option is to run an
appropriately reviewed build from an APF-authorized environment. Consult the
site security administrator before either change.

## CATMAP2

CATMAP2 was an unsuccessful intermediate implementation and is not included.

## Binary release installation

`CATMAP.XMI` is a binary TSO TRANSMIT file containing `CATMAP.XMILIB`:

- `LOADLIB`: nested XMIT of the three assembled load modules (`CATMAP3`,
  `CATMAP3J`, `CATMAP3N`).
- `JCLLIB`: nested XMIT containing `RUN3`, `RUN3J`, `RUN3N`, `RUNREXX`,
  the `CATMAP3J` CLIST, and the original `CATMAP` REXX exec.
- `SOURCE`: nested XMIT of the three HLASM sources and their compile JCL
  (`COMPILE3`, `COMPILEJ`, `COMPILEN`).
- `INSTALL`: CLIST that receives all three libraries under the executing user's
  `&SYSUID`.

Upload `CATMAP.XMI` **in binary mode** to a sequential FB 80 data set. For
example, with Zowe configured for your system (replace `YOURID`):

```sh
zowe zos-files create ps YOURID.CATMAP.XMI --rf FB --rl 80 --bs 27920 --ps 5 --ss 2
zowe zos-files upload file-to-data-set CATMAP.XMI YOURID.CATMAP.XMI --binary
```

At the TSO READY prompt (or ISPF option 6), issue:

```text
RECEIVE INDATASET('YOURID.CATMAP.XMI')
```

When RECEIVE asks for restore parameters, enter:

```text
DATASET('YOURID.CATMAP.XMILIB')
```

Then execute:

```text
EX 'YOURID.CATMAP.XMILIB(INSTALL)'
```

INSTALL creates `YOURID.CATMAP.LOADLIB`, `YOURID.CATMAP.JCLLIB`, and
`YOURID.CATMAP.SOURCE`.
It refuses to replace existing libraries; rename those before installing.
The run JCL and CLIST use `&SYSUID` for the load-library name. The TSO job
input uses `SYMBOLS=EXECSYS` so symbols in CALL commands are resolved too.
Edit the sample JOB accounting information and classes for your site.
`RUN3J` writes FB 80 records to spool; the CLIST writes a sequential data set.
Set the listener address and port in `RUN3N` before submitting it.

The REXX source remains interpreted; the three HLASM variants are prebuilt.
The ACF2 authorization note above still applies to the prebuilt modules.

For rebuilding an installed release, customize and submit the SOURCE members
`COMPILE3` (CATMAP3), `COMPILEJ` (CATMAP3J), and `COMPILEN` (CATMAP3N).
They read the adjacent HLASM source members and link into the installed
`&SYSUID..CATMAP.LOADLIB`. The matching local files are `compile_catmap3.jcl`,
`compile_catmap3j.jcl`, and `compile_catmapn.jcl`.
`release/package.jcl` packages populated LOADLIB, JCLLIB, and SOURCE datasets into an
existing FB 80 XMILIB PDS (with `release/INSTALL.clist` uploaded as INSTALL),
then creates the outer sequential XMI. The outer XMI must not already exist.

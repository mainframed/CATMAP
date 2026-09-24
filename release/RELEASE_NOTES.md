Prebuilt CATMAP tools for z/OS, distributed as a nested TSO XMIT package.

- LOADLIB contains CATMAP3 (TSO), CATMAP3J (batch), and CATMAP3N (TCP).
- JCLLIB contains run JCL for all four implementations, the CATMAP3J CLIST,
  and the original CATMAP REXX exec.
- Run JCL and CLIST resolve the load library using the executing user's
  `&SYSUID`.
- SOURCE contains all three HLASM sources and their separate compile jobs:
  COMPILE3, COMPILEJ, and COMPILEN.
- XMILIB includes an INSTALL CLIST to receive all three libraries.

### Installation

1. Upload `CATMAP.XMI` in **binary mode** to `YOURID.CATMAP.XMI`, allocated
   as sequential FB 80 (for example, BLKSIZE 27920).
2. In TSO, enter `RECEIVE INDATASET('YOURID.CATMAP.XMI')`.
3. At the restore prompt, enter `DATASET('YOURID.CATMAP.XMILIB')`.
4. Run `EX 'YOURID.CATMAP.XMILIB(INSTALL)'`.

Replace YOURID with your TSO user ID. INSTALL creates YOURID.CATMAP.LOADLIB,
YOURID.CATMAP.JCLLIB, and YOURID.CATMAP.SOURCE and refuses to replace
existing libraries.
Customize the JOB cards for your site. Set a reachable listener address and
port in RUN3N before running the TCP variant. RUN3J writes output to spool;
the CATMAP3J CLIST creates userid.CATMAP.OUTPUT.

The assembler variants use RACROUTE STATUS=ACCESS. On ACF2 systems, the
site-specific authorization requirement described in the README applies.

### Validation

All three assembler/link jobs completed with CC 0000. The downloaded XMIT
was uploaded again, received, and installed successfully; every restored
member matched its original byte for byte. Live TCP and runtime catalog
scans were not tested as part of this packaging run.

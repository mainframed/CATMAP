# Release validation

Built on 2026-09-24 using Zowe against the requested z/OS host.

| Job | Purpose | Result |
| --- | --- | --- |
| JOB01481 | Assemble/link CATMAP3 | CC 0000 |
| JOB01482 | Assemble/link CATMAP3J | CC 0000 |
| JOB01483 | Assemble/link CATMAP3N | CC 0000 |
| JOB01494 | XMIT LOADLIB/JCLLIB/SOURCE, then XMILIB | CC 0000 |
| JOB01495 | Receive the re-uploaded CATMAP.XMI and run INSTALL | CC 0000 |

The restored LOADLIB (3 members), JCLLIB (6 members), SOURCE (6 members),
and XMILIB (4 members) match their original members byte for byte, including the load
modules. All JCL and CLIST source records fit FB80. Runtime catalog scans
and a live TCP connection were not part of this packaging validation.

CATMAP.XMI is 252,640 bytes (3,158 records of 80 bytes). SHA-256:

```text
6919407cecb9e03e186e4301d032137d84ed64b4711d11448f5e9cd9c050ce6e
```

The installer uses DATA PROMPT/ENDDATA to supply RECEIVE's restore
parameters; see [IBM's CLIST reference](https://publib.boulder.ibm.com/epubs/pdf/ikj4b840.pdf).
The TSO run jobs use SYMBOLS=EXECSYS for in-stream substitution; see
[IBM's SYMBOLS syntax](https://www.ibm.com/docs/en/zos/3.1.0?topic=sp-syntax-6).

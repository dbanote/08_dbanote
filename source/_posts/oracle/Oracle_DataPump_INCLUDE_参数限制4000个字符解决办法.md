---
title: Oracle expdp中INCLUDE参数限制4000个字符的解决办法
tags:
- oracle
- expdp
---

使用expdp的include参数，当需要包含大量表时，一旦字符超4000时就会报错
``` perl
expdp DADM/passwd DIRECTORY=DMP_DIR SCHEMAS=DADM
DUMPFILE=dadm_tables_data.dmp CONTENT=DATA_ONLY 
INCLUDE=TABLE:"IN ('CBTRFCC','CBTFCCN','CGTGCDD','CGDSALC',
...
'CGTRSGR','CBTFCCE','CGTREPD','CPDORPG')"

UDE-00014 : invalid value for parameter, 'include'.
```

使用以下技巧可以解决
``` perl
CREATE TABLE list_of_tables ( tbl_name VARCHAR2(30) );
INSERT INTO list_of_tables ( 'CBTRFCC' );
INSERT INTO list_of_tables ( 'CBTFCCN' );
...
INSERT INTO list_of_tables ( 'CPDORPG' );
COMMIT;

expdp DADM/passwd DIRECTORY=DMP_DIR SCHEMAS=DADM DUMPFILE=dadm_data.dmp 
CONTENT=DATA_ONLY include=TABLE:"IN (SELECT tbl_name FROM list_of_tables)"
```
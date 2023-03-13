---
title: Hive 명령어
categories: [hive]
tags: [hive]
comments : true
---

# Hive 명령어

# Hive 접속

Hive 마스터 노드에서

```bash
hive
```

를 통해서 접속!

# Hive LOAD DATA

```bash
LOAD DATA [LOCAL] INPATH 'filepath' [OVERWRITE] 
INTO TABLE tablename [PARTITION (partcol1=val1, partcol2=val2 ...)] 
[INPUTFORMAT 'inputformat' SERDE 'serde']
```

예제)

```bash
LOAD DATA INPATH '/data/DZ_BVD_ID_AND_NAME/BvD_ID_and_Name.txt'
INTO TABLE bigdata.DZ_BVD_ID_AND_NAME
```
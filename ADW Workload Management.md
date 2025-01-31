- Created: `=dateformat(this.file.ctime, "DDDD, HH:mm")`
- Last Modified: `=dateformat(this.file.mtime, "DDDD, HH:mm")`
- Tags: #AutonomousDB #ADW 
---
# Intro
- 문서의 목표
    - ADW workload management의 이해
        - 핵심은 parallelism: auto DOP & parallel statement queueing → **"Just load and run!"** → **Really?**
    - Scope
        - Not ADB, but ADW
        - Not ADW-D but ADW-S 
    - 방향: 상대적으로 잘 정립된 것 → 상대적으로 베일에 쌓인 것 
        - Non-ADB (일반 Orale DB) → ADW OCPU 모델 → ADW ECPU 모델
        - 19c → 23ai

- 테스트 환경: CPU 자원을 동등하게 가져간 5 개 데이터베이스 이용
    - Non-ADB (Base DB): `CDB19`: 16 OCPU → `PDB19`: 8 OCPU
    - ADW OCPU: `ADW19O`: 8 OCPU, `ADW23O`: 8 OCPU
    - ADW ECPU: ($OCPU : ECPU = 1 : 4$를 감안하여) `ADW19E`: 32 ECPU, `ADW23E`: 32 ECPU

# Non-ADB 셋업
- 목표: ADW provisioning에 최대한 비슷한 절차를 통해 ADW-alike한 DB instance 생성 & 셋업
    - (1) CDB-level resource plan 설정: "among PDBs"
    - (2) PDB 생성
    - (3) PDB-level resource plan 설정: "inside PDB, among consumer groups"
    - (4) 주요 parameter 설정
## CDB Resouce Plan 셋업
- CDB resouce plan `COMMON_PDB_PLAN` 생성
```sql
declare
    l_plan varchar2(30) := 'COMMON_PDB_PLAN';

begin
    dbms_resource_manager.clear_pending_area;
    dbms_resource_manager.create_pending_area;

    dbms_resource_manager.create_cdb_plan(plan => l_plan);

    dbms_resource_manager.validate_pending_area;
    dbms_resource_manager.submit_pending_area;
end;
/
```

- CDB plan directive 설정
    - CDB plan directive의 유형에 따라 방식이 달라짐
        - 세가지 유형
            - **PDB plan**: 특정 PDB에 적용 (one plan directive per PDB)
                - `dbms_resource_manager.create_cdb_plan_directive`로 plan directive를 생성하면서 PDB를 지정
            - **CDB default plan**: 모든 PDB에 공통 적용 (one plan directive for all PDBs) 
                - `dbms_resource_manager.update_cdb_default_directive`로 세팅
            - **PDB performance profile**: 어떤 workload 특성을 공유하는 PDB들에 대해 공통 적용 (one plan directive for a set of PDBs)
                - `dbms_resource_manager.create_cdb_profile_directive`로 plan directive와 함께 profile을 생성 
                - 이후 PDB 생성 후 해당 PDB의 `db_performance_profile` 파라미터 값으로 이 profile을 지정
        - **ADW는 CDB default plan 방식**  → 여기서도 이 방식 이용
            - 그리고 모든 directive들도 그냥 default 값 이용
     - 핵심 CDB plan directive
        - **SHARES**
            - 대상 PDB에게 할당되는 CPU 자원의 상대적인 ratio (natural number)
            - PDB의 `cpu_count`를 명시적으로 지정하는 경우 (이것이 일반적인 경우) 최종 share는 그 값에 의해 결정 
        - **UTILIZATION_LIMIT**
            - 대상 PDB가 사용하는 자원의 절대적인 상한 (%) 
            - PDB의 `cpu_count`를 명시적으로 지정하는 경우 최종 share는 그 값에 의해 결정
        - **PARALLEL_SERVER_LIMIT**
            - `parallel_servers_target` 파라미터
                - Parallel statement queueing 환경에서 active하게 사용할 수 있는 PX 서버들의 총 갯수 → 이 값을 넘기게 되는 statement는 queueing이 적용; "queueing point"
            - Parallel server limit은 PDB의 `parallel_servers_target`의 CDB 레벨의 `parallel_servers_target`에 대해 가져갈 수 있는 비율 (%)
            - PDB의 `cpu_count`를 명시적으로 지정하는 경우 `parallel_servers_target`은 다시 그 `cpu_count`에 의해 결정. 따라서 최종적인 parallel server limit은 `parallel_servers_target`에 의해 결정 
    - 결론
        - 일반적으로 **결국 PDB resource management의 핵심은 PDB의 `cpu_count`**

- 생성된 `COMMON_PDB_PLAN`의 (default) plan directive 확인
```sql
select plan,                    -- plan 이름
       pluggable_database,      -- 대상 PDB
       shares,                  -- 핵심 plan directive (1)
       utilization_limit,       -- 핵심 plan directive (2)
       parallel_server_limit    -- 핵심 plan directive (3)
  from dba_cdb_rsrc_plan_directives
 where plan = 'COMMON_PDB_PLAN';
```

```
PLAN                           PLUGGABLE_DATABASE            SHARES UTILIZATION_LIMIT PARALLEL_SERVER_LIMIT
------------------------------ ------------------------- ---------- ----------------- ---------------------
COMMON_PDB_PLAN                ORA$DEFAULT_PDB_DIRECTIVE          1
COMMON_PDB_PLAN                ORA$AUTOTASK                                        90
```
- Notes
    - `ORA$DEFAULT_PDB_DIRECTIVE`: 모든 PDB를 나타내는 이름
    - CDB default plan의 default 값
        - $SHARES = 1$
        - $UTILIZATION_LIMIT = 100$ (위 쿼리에서는 null로 나타남)
        - $PARALLEL_SERVER_LIMIT = 100$ (위 쿼리에서는 null로 나타남)

- `COMMON_PDB_PLAN` enable: `resource_manager_plan` 파라미터의 설정
```sql
alter system set resource_manager_plan = 'COMMON_PDB_PLAN';
```
## PDB 생성
- SQL command 이용
```sql
create pluggable database pdb19
    admin user admin identified by "Welcome123__" roles=(dba)    -- admin user password 및 role. 여기서는 그냥 dba만 지정. 실제 ADW provision 시에는 세부적인 다양한 role들이 부여됨
    storage (maxsize 1T)                                         -- storage 용량. (CPU 용량은 생성 후 cpu_count 파라미터 설정에 의해 이루어짐)
;
```
## PDB Resource Plan 셋업
- 서비스 셋업
```bash
srvctl add service -db cdb19_bgg_icn -service PDB19_HIGH -pdb PDB19
srvctl start service -db cdb19_bgg_icn -service PDB19_HIGH

srvctl add service -db cdb19_bgg_icn -service PDB19_MEDIUM -pdb PDB19
srvctl start service -db cdb19_bgg_icn -service PDB19_MEDIUM

srvctl add service -db cdb19_bgg_icn -service PDB19_LOW -pdb PDB19
srvctl start service -db cdb19_bgg_icn -service PDB19_LOW
```

- 서비스 셋업 결과 확인
```bash
srvctl status service -db cdb19_bgg_icn
```

```
Service cdb19_pdb19.paas.oracle.com is running on instance(s) cdb19
Service pdb19_high is running on instance(s) cdb19
Service pdb19_low is running on instance(s) cdb19
Service pdb19_medium is running on instance(s) cdb19
```

- 핵심 PDB resource plan directives
    - **SHARES**: 대상 consumer group에게 할당되는 CPU 자원의 상대적인 ratio (natural number)
        - 한편 parallel statement queue는 consumer group 당 하나씩 존재. 각각의 queue는 FIFO 방식이지만, 각 queue들 사이에서는 `shares`를 통해 dequeue 확률을 계산
    - **PARALLEL_DEGREE_LIMIT**
        - 대상 consumer group이 쓸 수 있는 DOP의 상한
    - **PARALLEL_SERVER_LIMIT**
        - PDB의 총 `parallel_servers_target` 개 PX 서버들 중 대상 consumer group이 사용할 수 있는 상한 (%)        

- PDB resource plan `DWCS_PLAN`(실제 ADW에서 사용하는 이름) 및 directive들 생성
```sql
declare
    l_plan varchar2(30) := 'DWCS_PLAN';

begin
    dbms_resource_manager.clear_pending_area;
    dbms_resource_manager.create_pending_area;

    dbms_resource_manager.create_plan(plan => l_plan);

    dbms_resource_manager.create_consumer_group(consumer_group => 'HIGH');
    dbms_resource_manager.create_consumer_group(consumer_group => 'MEDIUM');
    dbms_resource_manager.create_consumer_group(consumer_group => 'LOW');

    dbms_resource_manager.set_consumer_group_mapping(
        attribute => dbms_resource_manager.service_name,    -- 접속 서비스 기준의 mapping
        value => '%_HIGH%',
        consumer_group => 'HIGH'
    );
    dbms_resource_manager.set_consumer_group_mapping(
        attribute => dbms_resource_manager.service_name,
        value => '%_MEDIUM%',
        consumer_group => 'MEDIUM'
    );
    dbms_resource_manager.set_consumer_group_mapping(
        attribute => dbms_resource_manager.service_name,
        value => '%_LOW%',
        consumer_group => 'LOW'
    );

    -- 이하 다양한 magic number들을 사용. 모두 같은 자원의 ADW OCPU 모델에서 따온 것이며, 각각의 의미는 나중에 설명
    
    dbms_resource_manager.create_plan_directive(
        plan => l_plan,
        group_or_subplan => 'HIGH',
        shares => 4,                      
        parallel_degree_limit_p1 => 8,    
        parallel_server_limit => 50       
    );
    dbms_resource_manager.create_plan_directive(
        plan => l_plan,
        group_or_subplan => 'MEDIUM',
        shares => 2,
        parallel_degree_limit_p1 => 4,
        parallel_server_limit => 84
    );
    dbms_resource_manager.create_plan_directive(
        plan => l_plan,
        group_or_subplan => 'LOW',
        shares => 1,
        parallel_degree_limit_p1 => 1
    );
    dbms_resource_manager.create_plan_directive(
        plan => l_plan,
        group_or_subplan => 'OTHER_GROUPS',
        shares => 1,
        parallel_degree_limit_p1 => 1
    );

    dbms_resource_manager.validate_pending_area();
    dbms_resource_manager.submit_pending_area();
end;
/
```

- `DWCS_PLAN` enable
```sql
alter system set resource_manager_plan = 'DWCS_PLAN';
```

- `cpu_count` 설정 → **instance caging on**
```sql
alter system set cpu_count = 16;
```
- Notes
    - OCPU 모델의 경우 항상 $cpu\_count = 2 \times OCPUs$

- Consumer group mapping 설정 확인
```sql
select attribute, value, consumer_group
  from dba_rsrc_group_mappings;
```

```
ATTRIBUTE       VALUE                CONSUMER_GROUP
--------------- -------------------- ----------------------------------------
SERVICE_NAME    %_HIGH%              HIGH
SERVICE_NAME    %_LOW%               LOW
SERVICE_NAME    %_MEDIUM%            MEDIUM
ORACLE_USER     SYS                  SYS_GROUP
ORACLE_USER     SYSTEM               SYS_GROUP
ORACLE_FUNCTION BACKUP               BATCH_GROUP
ORACLE_FUNCTION COPY                 BATCH_GROUP
ORACLE_FUNCTION DATALOAD             ETL_GROUP
ORACLE_FUNCTION INMEMORY             ORA$AUTOTASK
```

- Consumer group mapping 동작 확인
```sql
connect admin/WElcome123__@pdb19_medium
@whoami
```

```
       SID    SERIAL#     OS PID USERNAME SCHEMA   MACHINE          PROGRAM          CDB              CON_NAME           INSTANCE SERVICE          MODULE           CGROUP
---------- ---------- ---------- -------- -------- ---------------- ---------------- ---------------- ---------------- ---------- ---------------- ---------------- ----------------
      3324      45283      10347 ADMIN    ADMIN    SANGLEE-PF1FSXNM sqlplus@SANGLEE- cdb19            PDB19                     1 PDB19_MEDIUM     SQL*Plus         MEDIUM
                                                                    PF1FSXNM (TNS V1
                                                                    -V3)
```
## 핵심 Parameter들 설정
- `cpu_count`: CPU 자원
    - CPU 자원의 할당은 OCPU나 ECPU로 지정. 하지만 DB 내에서는 결국 `cpu_count`로 표현되며, 다양한 CPU 관련 설정들이 이를 기준으로 조율됨
    - 전 단계에서 $2 \times OCPUs$로 설정

- `parallel_degree_policy`
    - Parallelism을 위한 가장 기본적인 설정
        - (1) 병렬 처리를 할 것인지 말 것인지
        - (2) 병렬 처리를 한다면 DOP는 어떻게 설정할 것인지
    - 주요 값
        - `MANUAL` (default)
            - Manual parallelism
            - Semi auto dop: manual parallelize, but auto dop
                - Degree를 지정하지 않는 parallel 힌트 (`/*+ parallel */`)
                - **Manual 환경에서 performance tuning에 상당히 유용한 기능**
        - `AUTO`
            - Auto parallelism: auto parallelize and auto dop
            - Parallel statement queueing
    - 설정
        - `alter system set parallel_degree_policy = AUTO;`

- `parallel_servers_target`
    - Rule: $active\ PX\ servers \le parallel\_servers\_target$
        - e.g. 
            - 어떤 병렬 statement를 DOP 4로 요청
            - 현재 `parallel servers target` 대비 가용한 PX 서버의 갯수는 2개만 남음
            - Then, 이 병렬 statement는 queued, instead of DOP downgrade
                - **Parallel statement queueing과 DOP downgrade는 mutual exclusive**
    - Default 값
        - CDB: 16 OCPU에서 1280 → $parallel\_servers\_target = parallel\_max\_servers$  
        - PDB: 8 OCPU로 변경 후 512 → $parallel\_servers\_target \lt parallel\_max\_servers$ (하지만 `parallel_max_servers`의 값은 `cpu_count`를 다르게 설정했는데도 그대로 inherit)
    - 설정: 8 OCPU ADW에 맞춤
        - `alter system set parallel_servers_target = 96;`
## Non-ADB의 최종 Resource Plan 확인
- PDB resource plan
```sql
select * from pt(q'[select * from v$rsrc_plan]');
```

```
COLUMN                         VALUE
------------------------------ ------------------------------------------------------------
ID                             75044
NAME                           DWCS_PLAN
IS_TOP_PLAN                    TRUE                 -- PDB는 원래 subplan을 지원하지 않음
CPU_MANAGED                    ON                   -- CPU 자원이 관리됨
INSTANCE_CAGING                ON                   -- Instance caging이 적용됨
PARALLEL_SERVERS_ACTIVE        0                    -- 현재 active한 PX 서버 갯수
PARALLEL_SERVERS_TOTAL         96                   -- = parallel_servers_target
PARALLEL_EXECUTION_MANAGED     FULL                 -- parallel statement queueing이 적용됨
CON_ID                         4
DIRECTIVE_TYPE                 DEFAULT_DIRECTIVE    -- CDB plan 유형
SHARES                         1                    -- CDB default plan 값 그대로. 하지만 doesn't matter
UTILIZATION_LIMIT              50                   -- cpu_count에 의해 16/32로 정확히 계산되어 나타남
PARALLEL_SERVER_LIMIT                               -- CDB default plan 값은 100%이고 null로 표현. 하지만 doesn't matter
MEMORY_MIN                                          -- Exadata Smart Flash Cache 및 PMEM Cache의 최소 사용량 (%)
MEMORY_LIMIT                                        -- Exadata Smart Flash Cache 및 PMEM Cache의 최대 사용량 (%)
PROFILE                                             -- = db_performance_profile
```
- Notes
    - `v$rsrc_plan`은 현재 enable된 PDB plan의 정보를 보여 주지만, 동시에 CDB plan의 설정 내용을 엿볼 수 있음
        - 일반적으로 PDB에서 직접 `dba_cdb_rsrc_plan_directives`를 통해 CDB plan을 볼 수는 없음

- PDB resource plan directive
```sql
select plan,
       group_or_subplan,
       type,
       mgmt_p1,                    -- = share
       active_sess_pool_p1,        -- session queueing은 사용하지 않음을 명시적으로 보이기 위해 포함
       parallel_degree_limit_p1,
       parallel_server_limit
  from dba_rsrc_plan_directives
 where plan = 'DWCS_PLAN';
```

```
PLAN       GROUP_OR_SUBPLAN TYPE              MGMT_P1 ACTIVE_SESS_POOL_P1 PARALLEL_DEGREE_LIMIT_P1 PARALLEL_SERVER_LIMIT
---------- ---------------- -------------- ---------- ------------------- ------------------------ ---------------------
DWCS_PLAN  HIGH             CONSUMER_GROUP          4                                            8                    50
DWCS_PLAN  MEDIUM           CONSUMER_GROUP          2                                            4                    84
DWCS_PLAN  LOW              CONSUMER_GROUP          1                                            1
DWCS_PLAN  OTHER_GROUPS     CONSUMER_GROUP          1                                            1
```

# ADW의 설정 확인
## 핵심 Parameter들
- Intro
    - 세가지 핵심 parameter
        - `cpu_count`: \#OCPU 또는 \#ECPU에 의해 결정 → how?
        - `parallel_degree_policy` = `AUTO`
        - `parallel_servers_target`:  `cpu_count`에 의해 결정 → how?
    
- 이를 확인하기 위해 SDK를 이용 점진적으로 scale-up을 하면서 각각의 parameter 값 조사. 먼저 OCPU 모델의 경우:
```
  OCPU/ECPU    INSTANCE_COUNT    CPU_COUNT    PARALLEL_SERVERS_TARGET
-----------  ----------------  -----------  -------------------------
          1                 1            2                         12
          2                 1            4                         24
          4                 1            8                         48
          6                 1           12                         72
          8                 1           16                         96    -- 현재 값 
         10                 1           20                        120
         12                 1           24                        144
         14                 1           28                        168
         16                 2           16                         96    -- 이후 RAC
         18                 2           18                        108
         20                 2           20                        120
```
- Observation
    - $cpu\_count = 2 \times OCPUs$
    - $parallel\_servers\_target = 6 \times cpu\_count$
        - 이것은 non-ADB의 공식과는 다름. 그리고 계산된 값 역시 non_ADB에 비해 훨씬 작음 

- 다음은 ECPU 모델의 경우
```
  OCPU/ECPU    INSTANCE_COUNT    CPU_COUNT    PARALLEL_SERVERS_TARGET
-----------  ----------------  -----------  -------------------------
          2                 1            1                          6
          4                 1            2                         12
          6                 1            3                         18
          8                 1            4                         24
         10                 1            5                         30
         12                 1            6                         36
         14                 1            7                         42
         16                 1            8                         48
         18                 1            9                         54
         20                 1           10                         60
         22                 1           11                         66
         24                 1           12                         72
         26                 1           13                         78
         28                 1           14                         84
         30                 1           15                         90
         32                 1           16                         96    -- 현재 값   
         ...
         64                 2           16                         96    -- 이후 RAC
```
- Observation
    - $cpu\_count = 0.5 \times ECPUs$
        - 그런데 여기서 $OCPU : ECPU = 1 : 4$ 비율을 적용하여 계산을 더 해 보면
            - $cpu\_count = 0.5 \times ECPUs = 0.5 \times (4 \times OCPUs) = 2 \times OCPUs$ → OCPU 모델에서의 계산식과 equivalent
    - $parallel\_servers\_target = 6 \times cpu\_count$
 
- 결론
    - **ADW에서 동등한 자원의 OCPU 모델과 ECPU 모델은 같은 `cpu_count`, 같은 `parallel_servers_target`를 갖도록 세팅**
        - OCPU 모델에서 ECPU 모델로의 transition은 자원 설정 면에서는 매우 seamless할 것 (MOS 2998742.1)
    - 한편 auto scaling을 enable하면 `cpu_count`의 값은 즉시 3배로 증가
        - 즉 auto scaling은 즉각적인 것. (Billing은 실제 사용량을 근거로 하겠지만)
## PDB Resource Plan
- Non-ADB와 동일한 쿼리 이용
```sql
select * from pt(q'[select * from v$rsrc_plan]');
```

- OCPU 모델
```
COLUMN                         VALUE
------------------------------ ------------------------------------------------------------
ID                             41820
NAME                           DWCS_PLAN
IS_TOP_PLAN                    TRUE                 
CPU_MANAGED                    ON                   
INSTANCE_CAGING                ON                   
PARALLEL_SERVERS_ACTIVE        0                    
PARALLEL_SERVERS_TOTAL         96                   -- = parallel_servers_target
PARALLEL_EXECUTION_MANAGED     FULL                 
CON_ID                         103
DIRECTIVE_TYPE                 DEFAULT_DIRECTIVE    
SHARES                         1600                 -- = cpu_count × 100. 이 값은 natural number이면 되므로 cpu_count가 반영되는 한 그럴듯한 설정이라고 볼 수 있음 
UTILIZATION_LIMIT              16                   -- = cpu_count. 이 값은 percentage이므로 이렇게 설정되면 안 됨. 그냥 편의적 설정으로 보임.
PARALLEL_SERVER_LIMIT                               
MEMORY_MIN                                          
MEMORY_LIMIT                                        
PROFILE                                             
```

- ECPU 모델
```
COLUMN                         VALUE
------------------------------ ------------------------------------------------------------
ID                             41820
NAME                           DWCS_PLAN
IS_TOP_PLAN                    TRUE
CPU_MANAGED                    ON
INSTANCE_CAGING                ON
PARALLEL_SERVERS_ACTIVE        0
PARALLEL_SERVERS_TOTAL         96
PARALLEL_EXECUTION_MANAGED     FULL
CON_ID                         71
DIRECTIVE_TYPE                 DEFAULT_DIRECTIVE
SHARES                         1600
UTILIZATION_LIMIT              16
PARALLEL_SERVER_LIMIT
MEMORY_MIN
MEMORY_LIMIT
PROFILE
```

- Notes
    - 두 compute 모델은 정확히 동일한 설정을 보여줌
    - Non-ADB와의 차이점
        - Non-ADB에서는 $parallel\_max\_servers \gt parallel\_servers\_target$, 하지만 ADB에서는 두 모델 모두 $parallel\_max\_servers = parallel\_servers\_target$. 
            - `parallel_servers_target`이 단순한 queueing point가 아니라 parallelism의 상한 역할
                - Non-ADB에서는 parallel statement queueing이 몇가지 예외를 허용함. 예를 들어 `alter session set parallel_degree_policy = manual;`이 허용되므로 `parallel_servers_target` 이상의 PX 서버들을 사용하는 경우가 발생. 하지만 ADW는 그렇지 않음 
## PDB Resource Plan Directive
- 역시 non-ADB와 동일한 쿼리 이용
```sql
select plan,
       group_or_subplan,
       type,
       mgmt_p1,
       active_sess_pool_p1,
       parallel_degree_limit_p1,
       parallel_server_limit
  from dba_rsrc_plan_directives
 where plan = 'DWCS_PLAN';
```

- OCPU 모델
```
PLAN       GROUP_OR_SUBPLAN TYPE              MGMT_P1 ACTIVE_SESS_POOL_P1 PARALLEL_DEGREE_LIMIT_P1 PARALLEL_SERVER_LIMIT
---------- ---------------- -------------- ---------- ------------------- ------------------------ ---------------------
DWCS_PLAN  HIGH             CONSUMER_GROUP          4                                            8                    50
DWCS_PLAN  MEDIUM           CONSUMER_GROUP          2                                            4                    84
DWCS_PLAN  LOW              CONSUMER_GROUP          1                                            1
DWCS_PLAN  OTHER_GROUPS     CONSUMER_GROUP          1                                            1
```

- ECPU 모델
```
PLAN       GROUP_OR_SUBPLAN TYPE              MGMT_P1 ACTIVE_SESS_POOL_P1 PARALLEL_DEGREE_LIMIT_P1 PARALLEL_SERVER_LIMIT
---------- ---------------- -------------- ---------- ------------------- ------------------------ ---------------------
DWCS_PLAN  HIGH             CONSUMER_GROUP          4                                           16                   100
DWCS_PLAN  MEDIUM           CONSUMER_GROUP          2                                            4                    67
DWCS_PLAN  LOW              CONSUMER_GROUP          1                                            1
DWCS_PLAN  OTHER_GROUPS     CONSUMER_GROUP          1                                            1
```

- 설정만 보면 앞서 non-ADB에서의 설정과 동일

# ADW Service Table
## 개요
- ADW workload management의 요약: doc. ([Database Service Names for Autonomous Database](https://docs.oracle.com/en/cloud/paas/autonomous-database/serverless/adbsb/predefined-database-services-names.html))

- Terminologies
    - **DOP (Degree Of Parallelism)**
        - 병렬 처리는 다수의 PX 서버들이 하나의 **PSS (Parallel Server Set)** 을 이루어 같은 operation들을 병렬로 처리하는 것
            - 하나의 PSS가 한번에 처리하는 병렬 처리 operation들의 집합을 **DFO(Data Flow Operation)**라고 함
                - Serial 처리의 (row source) operation에 대응되는 개념
            - 그리고 DOP는 **하나의 PSS가 갖는 PX 프로세스의 갯수**로 정의됨
        - **명목 DOP (requested DOP)** vs. **실질 DOP (allocated PXs)**
            - 일반적으로 $Real\ DOP \gt Nomimal\ DOP$
                - 지극히 간단한 statement가 아니라면 보통 두개의 PSS가 한쌍을 이루어 **producer-consumer model**로 동작
                    - e.g Sorting (`select * from t1 order by c1;`)
                        - 이 작업은 `t1`의 읽기와 읽은 데이터의 정렬이라는 두 가지 operation으로 구성
                        - 단일 PSS를 사용한다고 가정
                            - 이 PSS가 읽기만 병렬로 처리한다면 정렬은 QC(Query Coordinator)가 단독으로 처리해야 함
                            - 이 PSS가 읽기와 정렬을 동시에 병렬로 처리한다면 그 중간 결과는 각 PX 서버가 읽은 데이터에 대한 정렬 결과에 불과. 여전히 QC 혼자 재정렬을 해야 함
                        - 실제 동작: PSS pair를 이용
                            - PSS1이 읽기, PSS2가 정렬을 담당. (PSS1이 producer, PSS2가 consumer) 이때 QC는 PSS2의 각 PX 서버 별로 담당해야 할 `c1` 값의 range를 미리 할당  
                            - 이제 PSS1이 `t1` 데이터를 읽음. 그리고 PSS1의 각 PX 서버가 읽어들인 `c1`의 값을 보고 PSS2의 담당 PX 서버에게 전송
                            - PSS2의 각 PX 프로세스들은 미리 지정된 `c1` 값 범위에 해당하는 데이터를 받아 정렬. 그 결과 해당 범위 내에서 완전히 정렬된 결과를 얻음. 이를 QC에게 전송
                            - QC는 그저 PSS2로부터 받은 중간 result set을 `c1` 범위 순서대로 client에게 반환하면 됨
                    - 하나의 병렬 statement를 처리하기 위해서는 하나의 PSS pair가 연쇄적으로 producer-consumer 역할을 교대 수행하며 진행해야 함. 이는 전체적으로 **DFO tree**를 이루게 됨
                        - 일반 serial 실행 계획도 tree 구조임. 즉 DFO tree란 일반 (serial) operation tree에 대응되는 개념
                - 따라서
                    - 하나의 PSS pair로 병렬 처리를 하는 경우 $Real\ DOP = 2 \times Nominal\ DOP$가 성림 → 이것이 **가장 일반적인 경우**
                    - 경우에 따라 DFO tree가 여러 개, 따라서 PSS pair가 여러 개 필요한 statement들이 있음. 이 경우 $Real\ DOP = 2 \times Nominal\ DOP \times Number\ of\ DFO\ trees$가 성립 
            - 결론: **DOP는 기본적으로 명목 DOP를 얘기하지만, 실제 사용하는 PX 갯수의 맥락에서는 반드시 실질 DOP를 따져야 함**
    - **Resource Share**
        - PDB plan directive의 그 **share**와 동일한 의미
    - **Concurrency**
        - 일반적 의미는 동시 접속 세션의 수 (`sessions`) → 한계 초과 시 접속 불가
            - 한편 이를 session queueing으로 처리할 수도 있음 (`active_session_pool`) → 하지만 ADW에서는 이를 사용하지 않음
        - Parallel statement queueing 환경에서는 **동시 실행 가능한 병렬 statement의 갯수** → 한계 초과 시 queued
            - Edge case: 만일 동시 접속 세션 수 한계에 이미 도달했다면 parallel statement 역시 queueing 대신 접속 불가로 처리
## Service Table의 내용
### OCPU 모델
- OCPU 모델의 service table

| Service Name | DOP     | Resource Share | Concurrency                | Concurrency w/ compute auto scaling (upto 3 times except LOW) |
| ------------ | ------- | -------------- | -------------------------- | ------------------------------------------------------------- |
| HIGH         | $OCPUs$ | 4              | 3                          | 9                                                             |
| MEDIUM       | 4       | 2              | $trunc(1.26 \times OCPUs)$ | $trunc(3.78) \times OCPUs$                                    |
| LOW          | 1       | 1              | $300 \times OCPUs$         | $300 \times OCPUs$                                            |
- 추가 설명
    - HIGH의 DOP
        - $OCPUs = 0.5 \times cpu\_count$
    - MEDIUM DOP의 정확한 공식
        - $*OCPUs$, when $2 \le OCPUs \lt 4$ (OCPU 갯수가 2 이상일 때만 parallelism)
        - $4$, when $OCPUs \ge 4$ → 일반적
    - 참고로 MEDIUM concurrency에 대해 document bug가 있음. 위 service table의 내용이 맞는 내용임

- DOP와 concurrency의 보다 정확한 의미
    - DOP는 PDB plan directive의 `PARALLEL_DEGREE_LIMIT` 값과 동일. 하지만 ADW에서 이 값은 **DOP의 상한이 아니라 fixed DOP임**
    - Fixed DOP를 사용하는 ADW 환경에서 concurrency는 간단한 규칙에 의해 계산됨
        - $Concurrency = Available\ PX\ servers \div Real\ DOP$

- 이상을 종합하면 다음과 같이 OCPU 모델의 service table equation을 얻을 수 있음. 먼저 HIGH의 경우:

$$
Concurrency = \frac {parallel\_servers\_target \times parallel\_server\_limit} {Real\ DOP} = \frac{(6 \times cpu\_count) \times 0.5} {2 \times (0.5 \times cpu\_count)} = 3
$$
- 다음은 MEDIUM의 경우:
$$
Concurrency = \frac {parallel\_servers\_target \times parallel\_server\_limit} {Real\ DOP} = \frac{(6 \times cpu\_count) \times 0.84} {2 \times 4} = 0.63 \times cpu\_count = 0.63 \times (2 \times OCPUs) = 1.26 \times OCPUs
$$
- 참고
    - Auto scaling이 enable되는 경우 `cpu_count`는 즉시 3배로 증가함. 이를 위 equation들에 적용하면 오직 분자에 있는 `cpu_count`만 3배가 되는 것임. 이에 따라 concurrency만 역시 3배로 증가하게 됨
    - Documentation에 다음과 같은 문장이 있음: "Note that these degree of parallelism values may be doubled for simple queries like a query on a single table."
        - e.g
            - 현재 (nominal) DOP가 4인 MEDIUM 서비스에서 `select * from t;` 문장을 수행하면 DOP 8로 수행됨 
                - 그런데 이때 필요한 operation은 읽기 하나 뿐이므로 PSS도 하나만 필요. 
                - 하지만 DOP가 2배가 되므로 최종적으로 사용하는 PX 서버의 갯수는 8
                - 결국 같은 MEDIUM 서비스에서 non-trivial statement와 동일한 갯수의 PX 서버를 사용하는 셈
### ECPU 모델
- ECPU 모델의 service table

| Service Name | DOP                       | Resource Share | Concurrency                   | Concurrency w/ compute auto scaling (upto 3 times except LOW) |
| ------------ | ------------------------- | -------------- | ----------------------------- | ------------------------------------------------------------- |
| HIGH         | $trunc(0.5 \times ECPUs)$ | 4              | 3                             | 9                                                             |
| MEDIUM       | 4                         | 2              | $trunc(0.25125 \times ECPUs)$ | $trunc(0.75375 \times ECPUs)$                                 |
| LOW          | 1                         | 1              | $75 \times ECPUs$             | $75 \times ECPUs$                                             |
- 추가 설명
    - HIGH의 DOP
        - $0.5 \times ECPUs = 0.5 \times (2 \times cpu\_count) = cpu\_count$ 
    - MEDIUM DOP의 정확한 공식
        - $trunc(0.5 \times ECPUs)$, when $4 \le OCPUs \lt 8$ (ECPU 갯수가 4 이상일 때만 parallelism)
        - $4$, when $OCPUs \ge 8$ → 일반적

- OCPU 때와 마찬가지로 equation을 정리해 보자. 먼저 HIGH:
$$
Concurrency = \frac {parallel\_servers\_target \times parallel\_server\_limit} {Real\ DOP} = \frac{(6 \times cpu\_count) \times 1} {2 \times cpu\_count} = 3
$$
- 다음은 MEDIUM:
$$
Concurrency = \frac {parallel\_servers\_target \times parallel\_server\_limit} {Real\ DOP} = \frac{(6 \times cpu\_count) \times 0.67} {2 \times 4} = 0.5025 \times cpu\_count = 0.5025 \times (0.5 \times ECPUs) = 0.25125 \times ECPUs
$$

## Service Table의 현재 설정 조회
- 다음 쿼리 이용
```sql
select consumer_group, shares, concurrency_limit as concurrency, degree_of_parallelism as dop
  from cs_resource_manager.list_current_rules();
```
- Notes
    - ADW-S에서는 `dbms_resource_manager`는 lockdown(PDB lockdown profile) 되어 있고 대신 `cs_resource_manager`를 통해 일부 기능만 open.
        - 하지만 **ADW-D에서는 엉뚱하게도 `dbms_resource_manager`가 open되어 있음.**

- OCPU 모델 w/o compute auto scaling
```
CONSUMER_GROUP                     SHARES CONCURRENCY        DOP
------------------------------ ---------- ----------- ----------
HIGH                                    4           3          8
MEDIUM                                  2          10          4
LOW                                     1        2400          1
```

- OCPU 모델 w/ compute auto scaling → concurrency만 증가
```
CONSUMER_GROUP                     SHARES CONCURRENCY        DOP
------------------------------ ---------- ----------- ----------
HIGH                                    4           9          8
MEDIUM                                  2          30          4
LOW                                     1        2400          1
```

- ECPU 모델 w/o compute auto scaling
```
CONSUMER_GROUP                     SHARES CONCURRENCY        DOP
------------------------------ ---------- ----------- ----------
HIGH                                    4           3         16
MEDIUM                                  2           8          4
LOW                                     1        2400          1
```

- ECPU 모델 w/ compute auto scaling → concurrency만 증가
```
CONSUMER_GROUP                     SHARES CONCURRENCY        DOP
------------------------------ ---------- ----------- ----------
HIGH                                    4           9         16
MEDIUM                                  2          24          4
LOW                                     1        2400          1
```

# 동작 확인
## Auto DOP
- Auto DOP(또는 Auto Parallelism)의 본래 의미 
    - (1) Parallelize 여부를 자동으로 결정
        - Manual 방식에서는 CBO에게 optimization 과정에서 parallelism을 고려해도 좋다는 허락이 필요 e.g. `parallel` 힌트, object의 `parallel` 속성, `alter session force parallel query|dml|ddl ...;`
        - 하지만 auto DOP에서는 아무런 directive가 없어도 알아서 병렬 처리 여부를 결정. 이 기준은,
            - $Estimated\ elapsed\ time \ge parallel\_min\_time\_threshold\ (default\ 10s)$ ⇒ parallize
    - (2) DOP도 optimizer가 자동으로 결정
        - Object statistics에 근거하여 계산

- 하지만 **ADW는 엄밀한 의미의 auto DOP를 구현하지 않음**
    - Parallelize 여부는 자동으로 결정하되, DOP는 지정된 값을 이용 → ADW → **plan directive `parallel_degree_limit`이 상한이 아닌 고정 DOP로 작용**
### Non-ADB
- 테스트 용 테이블 생성
```sql
create table t
pctfree 0
as
select rownum as id, rpad('*',84,'*') as pad
  from dual
connect by level <= 10000;
```
- Note
    - 건수는 10,000, block 갯수는 128 개의 작은 테이블
    - `t`에 parallel 속성도 부여되지 않음

- MEDIUM으로 접속하여 `parallel` 힌트도 없는 간단한 쿼리의 실행 계획 확인
```sql
@estb
select * from t order by id;
@este
```

```
Note
-----
   - automatic DOP: Computed Degree of Parallelism is 1 because of parallel threshold
```
- Notes
    - 예상 수행 시간이 `parallel_min_time_threshold`를 넘지 않으므로 no parallelism

- 통계 정보를 직접 설정하여 CBO를 속여 보자:
```sql
begin
    dbms_stats.set_table_stats(
        ownname => user,
        tabname => 'T',
        numrows => 4000000,
        numblks => 544000
     );
end;
/

@estb
select * from t order by id;
@este
```

```
Automatic Degree of Parallelism Information:
--------------------------------------------

   - Degree of Parallelism of 2 is derived from scan of object ADMIN.T

Note
-----
   - automatic DOP: Computed Degree of Parallelism is 2
```

- 통계 정보를 더 크게 설정
```sql
begin
    dbms_stats.set_table_stats(
        ownname => user,
        tabname => 'T',
        numrows => 8000000,
        numblks => 1088000
     );
end;
/

@estb
select * from t order by id;
@este
```

```
Automatic Degree of Parallelism Information:
--------------------------------------------

   - Degree of Parallelism of 4 is derived from scan of object ADMIN.T

Note
-----
   - automatic DOP: Computed Degree of Parallelism is 4
```

- 통계 정보를 훨씬 더 크게 설정
```sql
begin
    dbms_stats.set_table_stats(
        ownname => user,
        tabname => 'T',
        numrows => 16000000,
        numblks => 2176000
     );
end;
/

@estb
select * from t order by id;
@este
```

```
Note
-----
   - automatic DOP: Computed Degree of Parallelism is 4 because of degree limit
```
- Notes
    - 이 object 통계는 resource manager 설정이 없는 경우 DOP가 8로 계산되지만, resource plan directive `parallel_degree_limit`에 의해 4로 제한된 것
### ADW
- 동일한 테이블을 생성하고, 아무 통계 조작도 하지 않은 상태에서 - 즉 small table인 상태 그대로 MEDIUM으로 접속하여 실행 계획 확인
```sql
@estb
select * from t order by id;
@este
```

```
Note
-----
   - automatic DOP: Computed Degree of Parallelism is 4 because of degree limit
```

- 이와 같은 fixed DOP 적용은 OCPU/ECPU 모델에 공통
## Parallel Statement Queueing
### 준비
- 정확한 동작 확인을 위해 ADW의 result cache는 off
```sql
alter system set result_cache_mode = manual;
```

- 데이터
```sql
@drop_table numbers

create table numbers
as
select level as n,
       trunc(dbms_random.value(1, 1000), 4) as rand_val
  from dual
connect by level <= 1000;
```

- 쿼리 정의: q1.sql
```sql
select n1.n as group_id,
       count(*) as group_size,
       round(sum(power(n2.n, 3) * n3.rand_val)/1000000, 2) as weighted_cubic_millions,
       round(avg(power(n2.n * n3.rand_val, 2)), 2) as weighted_square_avg,
       round(stddev(n2.n * n3.rand_val), 2) as weighted_std_dev,
       round(sum(case when mod(n2.n + n3.n, 2) = 0 then power(n2.rand_val * n3.rand_val, 2)
                 else sqrt(abs(n2.rand_val - n3.rand_val)) end), 2) as conditional_sum
      from numbers n1
cross join numbers n2
cross join numbers n3
where n1.n <= 30
group by n1.n
order by n1.n;
```
- Notes
    - 이 쿼리는 단일 DFO tree 형태, 즉 한쌍의 PSS를 사용하는 가장 일반적인 케이스

- 동시 수행 job 정의: OCPU 모델의 MEDIUM 서비스에서 수행. (이후 다른 job들도 비슷한 방식으로 준비하여 테스트함)
```bash
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_medium q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_medium q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_medium q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_medium q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_medium q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_medium q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_medium q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_medium q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_medium q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_medium q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_medium q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_medium q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_medium q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_medium q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_medium q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_medium q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_medium q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_medium q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_medium q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_medium q1.sql
```

- 동시 수행: cwork.sh에 -n 옵션을 주지 않음 → job 내의 모든 라인이 동시에 수행 
```bash
cwork.sh /home/glpi/work/adw_test/a19om.job | tee -a $log_file &
```

- 종합적인 모니터링 스크립트 준비: px_mon.sql
```sql
with p as (  -- parallel_servers_target 값 
    select to_number(value) as parallel_servers_target
      from v$parameter
     where name = 'parallel_servers_target'
),
pp as (      -- System-wide한, 즉 CDB 레벨에서의 PX 서버 자원 현황; PX 프로세스 pool은 CDB 레벨에서 하나가 존재
    select *
      from (select status, count(*) as cnt
              from gv$px_process
             group by status)
    pivot (
        max(cnt)
        for status in ('IN USE' as system_in_use, 'AVAILABLE' as system_available)
    )
),
rsi as (     -- 현재 수행 중인 statement들의 명목 DOP 및 실질 DOP
    select sum(dop) as total_dop, sum(pq_servers) total_px
      from gv$rsrc_session_info
     where current_consumer_group = '&&1'
       and dop != 0
       and state != 'PQ QUEUED'
),
sm1 as (     -- 현재 수행 중인 statement들의 갯수와 queue에 들어간 statement들의 갯수
    select *
      from (select status, count(*) as cnt
              from gv$sql_monitor
             where rm_consumer_group = '&&1'
               and username = '&&2'
               and process_name = 'ora'
               and (status = 'EXECUTING' or status = 'QUEUED')
            group by status)
    pivot (
        max(cnt)
        for status in ('EXECUTING' as executing, 'QUEUED' as queued)
    )
),
sm2 as (     -- DOP downgrade 여부
    select *
      from (select px_status, count(*) as cnt
              from (select case when px_servers_allocated < px_servers_requested then 'DOWNGRADED'
                        else 'NORMAL' end px_status
                      from gv$sql_monitor
                     where rm_consumer_group = '&&1'
                       and username = '&&2'
                       and process_name = 'ora'
                       and status = 'EXECUTING')
            group by px_status)
        pivot (
            max(cnt)
            for px_status in ('DOWNGRADED' as downgraded, 'NORMAL' as normal)
        )
)
select nvl(executing, 0) as executing,
       nvl(downgraded, 0) as downgraded,
       nvl(queued, 0) as queued,
       nvl(total_dop, 0) as total_dop,
       nvl(total_px, 0) as totl_px,
       parallel_servers_target,
       nvl(system_in_use) as system_in_use,
       nvl(system_available) as system_available
  from sm1, sm2, rsi, p, pp;
```
- Notes
    - 오라클은 dynamic performance view에 대한 조회에 read consistency를 보장하지 않음. 따라서 이런 모니터링 스크립트는 가끔 inconsistent한 결과를 보일 수도 있음에 주의
### 수행
#### OCPU 모델
- MEDIUM에서의 px_mon.sql 결과 (px_mon.sql을 5초 단위로 반복 수행하여 얻은 로그)
    - 환경: concyrrency = 10, DOP = 4, tasks = 20
```
  EXECUTING DOWNGRADED     QUEUED  TOTAL_DOP    TOTL_PX PARALLEL_SERVERS_TARGET SYSTEM_IN_USE SYSTEM_AVAILABLE
 ---------- ---------- ---------- ---------- ---------- ----------------------- ------------- ----------------
         0          0          0          0          0                      96             1              708
        10          0         10         40         80                      96            87              622
        10          0         10         40         80                      96            88              621
        10          0         10         40         80                      96            86              623
        10          0         10         40         80                      96            80              629
        10          0         10         40         80                      96            81              628
        10          0         10         40         80                      96            83              626
        10          0         10         40         80                      96            80              629
        10          0         10         40         80                      96            90              619
        10          0         10         40         80                      96            80              629
        10          0         10         40         80                      96            80              629
        10          0          8         40         80                      96            80              629
        10          0          3         40         80                      96            80              629
        10          0          1         40         80                      96            80              629
        10          0          0         40         80                      96            80              629
        10          0          0         40         80                      96            80              629
        10          0          0         40         80                      96            81              628
        10          0          0         40         80                      96            86              623
        10          0          0         40         80                      96            86              623
        10          0          0         40         80                      96            86              623
        10          0          0         40         80                      96            86              623
        10          0          0         40         80                      96            86              623
        10          0          0         40         80                      96            86              623
         8          0          0         32         64                      96            64              645
         6          0          0         20         40                      96            48              661
         0          0          0          0          0                      96             8              701
         0          0          0          0          0                      96             0              709
```
- Notes
    - 정상 수행
    - $max(total\_px) \div parallel\_servers\_target \simeq 0.8333... \simeq 0.84 = parallel\_server\_limit$

- 실제 수행 로그
```
(base) glpi@SANGLEE-PF1FSXNM:~/work/adw_test$ ./a19om.sh
JOB: adw_test, TASK: q1.sql, START_DT: 20250119_143559.796655447, END_DT: 20250119_143647.759509607, ELAPSED: 00:00:47.96, ROWS: 30
JOB: adw_test, TASK: q1.sql, START_DT: 20250119_143559.835804747, END_DT: 20250119_143648.843201329, ELAPSED: 00:00:49.01, ROWS: 30
JOB: adw_test, TASK: q1.sql, START_DT: 20250119_143559.818439547, END_DT: 20250119_143657.731159962, ELAPSED: 00:00:57.91, ROWS: 30
JOB: adw_test, TASK: q1.sql, START_DT: 20250119_143600.011040746, END_DT: 20250119_143657.731157862, ELAPSED: 00:00:57.72, ROWS: 30
JOB: adw_test, TASK: q1.sql, START_DT: 20250119_143559.792451147, END_DT: 20250119_143657.732432462, ELAPSED: 00:00:57.94, ROWS: 30
JOB: adw_test, TASK: q1.sql, START_DT: 20250119_143559.848767847, END_DT: 20250119_143657.732916462, ELAPSED: 00:00:57.88, ROWS: 30
JOB: adw_test, TASK: q1.sql, START_DT: 20250119_143559.936704046, END_DT: 20250119_143657.973693663, ELAPSED: 00:00:58.03, ROWS: 30
JOB: adw_test, TASK: q1.sql, START_DT: 20250119_143559.860449047, END_DT: 20250119_143658.064307263, ELAPSED: 00:00:58.20, ROWS: 30
JOB: adw_test, TASK: q1.sql, START_DT: 20250119_143559.938057546, END_DT: 20250119_143658.249103664, ELAPSED: 00:00:58.31, ROWS: 30
JOB: adw_test, TASK: q1.sql, START_DT: 20250119_143559.887871347, END_DT: 20250119_143658.997991667, ELAPSED: 00:00:59.11, ROWS: 30
JOB: adw_test, TASK: q1.sql, START_DT: 20250119_143600.091060645, END_DT: 20250119_143745.218148407, ELAPSED: 00:01:45.12, ROWS: 30
JOB: adw_test, TASK: q1.sql, START_DT: 20250119_143600.002038246, END_DT: 20250119_143748.535889618, ELAPSED: 00:01:48.54, ROWS: 30
JOB: adw_test, TASK: q1.sql, START_DT: 20250119_143600.007939746, END_DT: 20250119_143748.600290918, ELAPSED: 00:01:48.59, ROWS: 30
JOB: adw_test, TASK: q1.sql, START_DT: 20250119_143600.194574645, END_DT: 20250119_143748.690741119, ELAPSED: 00:01:48.49, ROWS: 30
JOB: adw_test, TASK: q1.sql, START_DT: 20250119_143600.089684145, END_DT: 20250119_143748.904992819, ELAPSED: 00:01:48.81, ROWS: 30
JOB: adw_test, TASK: q1.sql, START_DT: 20250119_143600.148396945, END_DT: 20250119_143749.033940420, ELAPSED: 00:01:48.88, ROWS: 30
JOB: adw_test, TASK: q1.sql, START_DT: 20250119_143600.024702046, END_DT: 20250119_143749.081858520, ELAPSED: 00:01:49.07, ROWS: 30
JOB: adw_test, TASK: q1.sql, START_DT: 20250119_143600.146255145, END_DT: 20250119_143749.804034822, ELAPSED: 00:01:49.65, ROWS: 30
JOB: adw_test, TASK: q1.sql, START_DT: 20250119_143600.195538045, END_DT: 20250119_143751.070740627, ELAPSED: 00:01:50.87, ROWS: 30
JOB: adw_test, TASK: q1.sql, START_DT: 20250119_143600.187770345, END_DT: 20250119_143751.079267027, ELAPSED: 00:01:50.89, ROWS: 30
SCENARIO: a19om, START_DT: 20250119_143559.633076148, END_DT: 20250119_143751.099423627, ELAPSED: 00:01:51.46
```
- Notes
    - 10개씩 실행되므로 처음 10개의 수행 시간과 그 다음 10개의 수행 시간에 차이가 발생 → 후반 10개의 수행 시간에는 queue 대기 시간도 포함
        - 지금까지 parallel statement queueing의 경험이 별로 없었던 이유: 거의 모든 BMT/PoC에서 전체 job 뿐만 아니라 개별 task의 수행 시간도 측정했기 때문
            - 따라서 `cwork.sh -n` 옵션을 주로 이용 (shell process queueing)

- HIGH에서의 px_mon.sql 결과
    - 환경: concyrrency = 3, DOP = 8, tasks = 10
```
 EXECUTING DOWNGRADED     QUEUED  TOTAL_DOP    TOTL_PX PARALLEL_SERVERS_TARGET SYSTEM_IN_USE SYSTEM_AVAILABLE
---------- ---------- ---------- ---------- ---------- ----------------------- ------------- ----------------
        0          0          0          0          0                      96             0              708
        3          0          7         24         48                      96            59              649
        3          0          7         24         48                      96            48              660
        3          0          7         24         48                      96            48              660
        3          0          7         24         48                      96            48              662
        3          0          7         24         48                      96            48              662
        3          0          7         24         48                      96            49              661
        3          0          7         24         48                      96            48              662
        2          0          6         24         48                      96            48              662
        3          0          4         24         48                      96            49              661
        3          0          4         24         48                      96            56              654
        3          0          4         24         48                      96            56              654
        3          0          4         24         48                      96            54              656
        3          0          4         24         48                      96            48              662
        3          0          4         24         48                      96            50              660
        3          0          4         24         48                      96            49              661
        3          0          2         24         48                      96            48              662
        3          0          1         24         48                      96            48              662
        3          0          1         24         48                      96            48              662
        3          0          1         24         48                      96            48              662
        3          0          1         24         48                      96            48              662
        3          0          1         24         48                      96            48              662
        3          0          1         24         48                      96            48              662
        3          0          1         24         48                      96            49              661
        2          0          0          8         16                      96            16              694
        1          0          0          8         16                      96            16              694
        1          0          0          8         16                      96            16              694
        1          0          0          8         16                      96            16              694
        1          0          0          8         16                      96            16              694
        1          0          0          8         16                      96            16              694
        1          0          0          8         16                      96            16              694
        0          0          0          0          0                      96             0              710
```
- Notes
    - 정상 수행
    - $max(total\_px) \div parallel\_servers\_target = 0.5 = parallel\_server\_limit$
#### ECPU 모델
- MEDIUM에서의 px_mon.sql 결과
    - 환경: concyrrency = 8, DOP = 4, tasks = 20
```
 EXECUTING DOWNGRADED     QUEUED  TOTAL_DOP    TOTL_PX PARALLEL_SERVERS_TARGET SYSTEM_IN_USE SYSTEM_AVAILABLE
---------- ---------- ---------- ---------- ---------- ----------------------- ------------- ----------------
        0          0          0          0          0                      96             0              707
        8          0         12         32         64                      96            64              643
        8          0         12         32         64                      96            64              643
        8          0         12         32         64                      96            64              643
        8          0         12         32         64                      96            65              642
        8          0         12         32         64                      96            70              637
        8          0         12         32         64                      96            70              637
        8          0         12         32         64                      96            70              637
        8          0         12         32         64                      96            64              643
        8          0         12         32         64                      96            65              642
        8          0         10         32         64                      96            64              643
        8          0          8         32         64                      96            65              642
        8          0          4         32         64                      96            74              633
        8          0          4         32         64                      96            64              643
        8          0          4         32         64                      96            64              643
        8          0          4         32         64                      96            64              643
        8          0          4         32         64                      96            74              633
        8          0          4         32         64                      96            64              643
        8          0          4         32         64                      96            64              643
        8          0          4         32         64                      96            65              642
        8          0          4         32         64                      96            64              643
        8          0          2         32         64                      96            64              643
        4          0          0         16         32                      96            33              674
        4          0          0         16         32                      96            32              675
        4          0          0         16         32                      96            32              675
        4          0          0         16         32                      96            32              675
        4          0          0         16         32                      96            32              675
        4          0          0         16         32                      96            32              675
        4          0          0         16         32                      96            32              675
        4          0          0         16         32                      96            32              675
        0          0          0          0          0                      96             0              707
```
- Notes
    - 정상 수행
    - $max(total\_px) \div parallel\_servers\_target \simeq 0.6666... \simeq 0.67 = parallel\_server\_limit$

- HIGH에서의 px_mon.sql 결과
    - 환경: concyrrency = 3, DOP = 16, tasks = 10
```
 EXECUTING DOWNGRADED     QUEUED  TOTAL_DOP    TOTL_PX PARALLEL_SERVERS_TARGET SYSTEM_IN_USE SYSTEM_AVAILABLE
---------- ---------- ---------- ---------- ---------- ----------------------- ------------- ----------------
        0          0          0          0          0                      96             7              701
        3          0          7         48         96                      96           106              602
        3          0          7         48         96                      96            97              611
        3          0          7         48         96                      96            96              612
        3          0          7         48         96                      96            96              612
        3          0          7         48         96                      96            96              612
        3          0          7         48         96                      96            96              612
        3          0          7         48         96                      96            96              612
        2          0          5         48         96                      96            97              611
        3          0          4         48         96                      96            96              612
        3          0          4         48         96                      96            96              612
        3          0          4         48         96                      96            96              612
        3          0          4         48         96                      96            96              612
        3          0          4         48         96                      96            96              612
        3          0          4         48         96                      96            96              612
        3          0          4         48         96                      96            96              612
        3          0          2         48         96                      96            97              611
        3          0          2         48         96                      96            97              611
        3          0          1         48         96                      96            96              612
        3          0          1         48         96                      96            96              612
        3          0          1         48         96                      96            96              612
        3          0          1         48         96                      96            97              611
        3          0          1         48         96                      96           107              601
        3          0          1         48         96                      96           103              605
        3          0          1         48         96                      96           102              606
        2          0          0         32         64                      96            64              644
        1          0          0         16         32                      96            32              676
        1          0          0         16         32                      96            32              676
        1          0          0         16         32                      96            33              675
        1          0          0         16         32                      96            32              676
        1          0          0         16         32                      96            32              676
        1          0          0         16         32                      96            33              675
        1          0          0         16         32                      96            32              676
        0          0          0          0          0                      96             1              707
```
- Notes
    - 정상 수행
    - $max(total\_px) \div parallel\_servers\_target = 1 = parallel\_server\_limit$
### 특수 케이스
#### Multiple DFO Trees
- 쿼리: q2 (SSB sample query 중 하나)
```sql
select c_nation, s_nation, d_year, sum(lo_revenue) as revenue
  from ssb.customer, ssb.lineorder, ssb.supplier, ssb.dwdate
 where lo_custkey = c_custkey
   and lo_suppkey = s_suppkey
   and lo_orderdate = d_datekey
   and c_region = 'ASIA' and s_region = 'ASIA'
   and d_year >= 1992 and d_year <= 1997
group by c_nation, s_nation, d_year
order by d_year asc, revenue desc;
```

- 이 쿼리의 실행 계획
```sql
------------------------------------------------------------------------------------
| Id  | Operation                                    | Name                        |
------------------------------------------------------------------------------------
|   0 | SELECT STATEMENT                             |                             |
|   1 |  TEMP TABLE TRANSFORMATION                   |                             |
|   2 |   LOAD AS SELECT                             | SYS_TEMP_0FD9E00B2_541D9738 |
|   3 |    PX COORDINATOR                            |                             |
|   4 |     PX SEND QC (RANDOM)                      | :TQ10001                    |
|   5 |      HASH GROUP BY                           |                             |
|   6 |       PX RECEIVE                             |                             |
|   7 |        PX SEND HASH                          | :TQ10000                    |
|   8 |         KEY VECTOR CREATE BUFFERED           | :KV0000                     |
|   9 |          PX BLOCK ITERATOR                   |                             |
|* 10 |           TABLE ACCESS STORAGE FULL          | CUSTOMER                    |
|  11 |   LOAD AS SELECT                             | SYS_TEMP_0FD9E00B3_541D9738 |
|  12 |    PX COORDINATOR                            |                             |
|  13 |     PX SEND QC (RANDOM)                      | :TQ20001                    |
|  14 |      HASH GROUP BY                           |                             |
|  15 |       PX RECEIVE                             |                             |
|  16 |        PX SEND HASH                          | :TQ20000                    |
|  17 |         KEY VECTOR CREATE BUFFERED           | :KV0001                     |
|  18 |          PX BLOCK ITERATOR                   |                             |
|* 19 |           TABLE ACCESS STORAGE FULL          | SUPPLIER                    |
|  20 |   LOAD AS SELECT                             | SYS_TEMP_0FD9E00B4_541D9738 |
|  21 |    PX COORDINATOR                            |                             |
|  22 |     PX SEND QC (RANDOM)                      | :TQ30001                    |
|  23 |      HASH GROUP BY                           |                             |
|  24 |       PX RECEIVE                             |                             |
|  25 |        PX SEND HASH                          | :TQ30000                    |
|  26 |         KEY VECTOR CREATE BUFFERED           | :KV0002                     |
|  27 |          PX BLOCK ITERATOR                   |                             |
|* 28 |           TABLE ACCESS STORAGE FULL          | DWDATE                      |
|  29 |   PX COORDINATOR                             |                             |
|  30 |    PX SEND QC (ORDER)                        | :TQ40003                    |
|  31 |     SORT ORDER BY                            |                             |
|  32 |      PX RECEIVE                              |                             |
|  33 |       PX SEND RANGE                          | :TQ40002                    |
|* 34 |        HASH JOIN                             |                             |
|  35 |         PX RECEIVE                           |                             |
|  36 |          PX SEND BROADCAST                   | :TQ40000                    |
|  37 |           PX BLOCK ITERATOR                  |                             |
|  38 |            TABLE ACCESS STORAGE FULL         | SYS_TEMP_0FD9E00B3_541D9738 |
|* 39 |         HASH JOIN                            |                             |
|  40 |          TABLE ACCESS STORAGE FULL           | SYS_TEMP_0FD9E00B2_541D9738 |
|* 41 |          HASH JOIN                           |                             |
|  42 |           TABLE ACCESS STORAGE FULL          | SYS_TEMP_0FD9E00B4_541D9738 |
|  43 |           VIEW                               | VW_VT_846B3E5D              |
|  44 |            HASH GROUP BY                     |                             |
|  45 |             PX RECEIVE                       |                             |
|  46 |              PX SEND HASH                    | :TQ40001                    |
|  47 |               VECTOR GROUP BY                |                             |
|  48 |                HASH GROUP BY                 |                             |
|  49 |                 KEY VECTOR USE               | :KV0002                     |
|  50 |                  KEY VECTOR USE              | :KV0000                     |
|  51 |                   KEY VECTOR USE             | :KV0001                     |
|  52 |                    PX BLOCK ITERATOR         |                             |
|* 53 |                     TABLE ACCESS STORAGE FULL| LINEORDER                   |
------------------------------------------------------------------------------------
```
- Notes
    - Factored subquery의 materialization이 포함된 statement가 대표적인 multiple DFO tree의 예. 
        - 각각의 materailization은 별도의 DFO tree를 통해 수행되며 각각 QC가 coordinate 한다. 위 플랜에서는 그런 것이 3개이고, 그 다음 main 쿼리가 다시 독자의 DFO tree를 통해 수행되어 최종적으로 QC에 의해 취합. 따라서 이 쿼리는 총 4개의 DFO tree로 구성됨
        - 이 경우 $Real\ DOP = 2 \times Nominal\ DOP \times 4 = 8 \times Nominal\ DOP$의 관계가 성립
        - 이런 경우에 parallel statement queuing은 어떻게 동작할까? 너무 큰 real DOP에 의해 concurrency가 service table에 제시된 기준보다 줄어들지 않을까?

- 먼저 OCPU 모델에서 job을 구성한 후 HIGH 서비스를 이용하여 수행. 먼저 SQL Monitoring 결과:
```
STATUS          START    END      DURATION SQL_ID          SQL_EXEC_ID        PHV USER     PROGRAM          MODULE           ACTION           DOP      
--------------- -------- -------- -------- --------------- ----------- ---------- -------- ---------------- ---------------- ---------------- --------
...
QUEUED          15:08:01                   bfvvuj9fck45n      33554437 3373953146 ADMIN    sqlplus@SANGLEE- adw_test         q2.sql                
QUEUED          15:08:01                   bfvvuj9fck45n      33554439 3373953146 ADMIN    sqlplus@SANGLEE- adw_test         q2.sql                
QUEUED          15:08:01                   bfvvuj9fck45n      33554440 3373953146 ADMIN    sqlplus@SANGLEE- adw_test         q2.sql                
QUEUED          15:08:01                   bfvvuj9fck45n      33554441 3373953146 ADMIN    sqlplus@SANGLEE- adw_test         q2.sql                
DONE (ALL ROWS) 15:08:01 15:08:26 00:00:25 bfvvuj9fck45n      33554433 3373953146 ADMIN    sqlplus@SANGLEE- adw_test         q2.sql           64(1)
DONE (ALL ROWS) 15:08:01 15:08:27 00:00:26 bfvvuj9fck45n      33554432 3373953146 ADMIN    sqlplus@SANGLEE- adw_test         q2.sql           64(1)
DONE (ALL ROWS) 15:08:01 15:08:27 00:00:26 bfvvuj9fck45n      33554436 3373953146 ADMIN    sqlplus@SANGLEE- adw_test         q2.sql           64(1)
EXECUTING       15:08:01          00:00:29 bfvvuj9fck45n      33554435 3373953146 ADMIN    sqlplus@SANGLEE- adw_test         q2.sql           64(1)
EXECUTING       15:08:01          00:00:30 bfvvuj9fck45n      33554434 3373953146 ADMIN    sqlplus@SANGLEE- adw_test         q2.sql           64(1)
EXECUTING       15:08:01          00:00:30 bfvvuj9fck45n      33554438 3373953146 ADMIN    sqlplus@SANGLEE- adw_test         q2.sql           64(1)
```
- Notes
    - 여기서 DOP는 real DOP로 nominal DOP 8의 8 배인 64로 관측
    - 만일 64개의 PX 서버들을 동시에 필요로 한다면 concurrency의 감소가 불가피 → 하지만 그렇지 않다!

- 이제 px_mon.sql의 결과를 보면:
```
 EXECUTING DOWNGRADED     QUEUED  TOTAL_DOP    TOTL_PX PARALLEL_SERVERS_TARGET SYSTEM_IN_USE SYSTEM_AVAILABLE
---------- ---------- ---------- ---------- ---------- ----------------------- ------------- ----------------
        0          0          0          0          0                      96             8              702
        3          0          7         24         48                      96            51              659
        3          0          7         24         48                      96            49              661
        3          0          7         24         48                      96            49              661
        3          0          7         24         48                      96            49              661
        3          0          7         24         48                      96            49              661
        3          0          4         24         48                      96            49              661
        3          0          4         24         48                      96            49              661
        3          0          4         24         48                      96            48              662
        3          0          4         24         48                      96            49              661
        3          0          4         24         48                      96            48              662
        3          0          1         24         48                      96            48              662
        3          0          1         24         48                      96            48              660
        3          0          1         24         48                      96            48              660
        3          0          1         24         48                      96            48              660
        3          0          1         24         48                      96            48              660
        1          0          0          8         16                      96            16              692
        1          0          0          8         16                      96            16              692
        1          0          0          8         16                      96            16              692
        1          0          0          8         16                      96            16              692
        0          0          0          0          0                      96             1              707
```
- Notes
    - 실제 수행은 no problem. 각각의 DFO tree가 순차적으로 수행되어 동시에 수행되는 것이 아니므로 PX 서버 사용량이 뻥튀기 되지 않는 것.
        - SQL monitor에서는 runtime 값이 아닌 optimization 결과를 보여준 것으로 해석해야 할 것

- 다음으로 ECPU 모델에서 동일한 방식으로 수행. 먼저 SQL Monitoring 결과:
```
STATUS          START    END      DURATION SQL_ID          SQL_EXEC_ID        PHV USER     PROGRAM          MODULE           ACTION           DOP     
--------------- -------- -------- -------- --------------- ----------- ---------- -------- ---------------- ---------------- ---------------- --------
...
QUEUED          15:20:55                   bfvvuj9fck45n      33554450 3373953146 ADMIN    sqlplus@SANGLEE- adw_test         q2.sql                   
QUEUED          15:20:55                   bfvvuj9fck45n      33554444 3373953146 ADMIN    sqlplus@SANGLEE- adw_test         q2.sql                   
QUEUED          15:20:55                   bfvvuj9fck45n      33554446 3373953146 ADMIN    sqlplus@SANGLEE- adw_test         q2.sql                   
QUEUED          15:20:55                   bfvvuj9fck45n      33554449 3373953146 ADMIN    sqlplus@SANGLEE- adw_test         q2.sql                   
QUEUED          15:20:55                   bfvvuj9fck45n      33554451 3373953146 ADMIN    sqlplus@SANGLEE- adw_test         q2.sql                   
QUEUED          15:20:55                   bfvvuj9fck45n      33554448 3373953146 ADMIN    sqlplus@SANGLEE- adw_test         q2.sql                   
QUEUED          15:20:55                   bfvvuj9fck45n      33554447 3373953146 ADMIN    sqlplus@SANGLEE- adw_test         q2.sql                   
EXECUTING       15:20:55          00:00:09 bfvvuj9fck45n      33554445 3373953146 ADMIN    sqlplus@SANGLEE- adw_test         q2.sql           128(1)  
EXECUTING       15:20:55          00:00:09 bfvvuj9fck45n      33554443 3373953146 ADMIN    sqlplus@SANGLEE- adw_test         q2.sql           128(1)  
EXECUTING       15:20:55          00:00:11 bfvvuj9fck45n      33554442 3373953146 ADMIN    sqlplus@SANGLEE- adw_test         q2.sql           128(1)  
```
- Notes
    - 역시 SQL Monitor에서는 real DOP가 nominal DOP 16의 8배인 128로 나타남

- 다음은 px_mon.sql의 결과:
```
 EXECUTING DOWNGRADED     QUEUED  TOTAL_DOP    TOTL_PX PARALLEL_SERVERS_TARGET SYSTEM_IN_USE SYSTEM_AVAILABLE
---------- ---------- ---------- ---------- ---------- ----------------------- ------------- ----------------
        0          0          0          0          0                      96             0              709
        3          0          7         48         96                      96            96              613
        3          0          7         48         96                      96            96              613
        3          0          7         48         96                      96            96              613
        3          0          7         48         96                      96            97              612
        3          0          7         48         96                      96            96              613
        3          0          7         48         96                      96           112              597
        3          0          4         48         96                      96            99              610
        3          0          4         48         96                      96            96              613
        3          0          4         48         96                      96            96              613
        3          0          4         48         96                      96            96              613
        3          0          4         48         96                      96            97              612
        3          0          1         48         96                      96           102              607
        3          0          1         48         96                      96            96              613
        3          0          1         48         96                      96            98              611
        3          0          1         48         96                      96            96              613
        3          0          1         48         96                      96            96              613
        3          0          1         48         96                      96            75              635
        1          0          0         16         32                      96            39              671
        1          0          0         16         32                      96            33              677
        0          0          0          0          0                      96             7              703
```
- Notes
    - 역시 문제없이 동작
#### 복합 서비스
- 이번에는 MEDIUM과 HIGH를 동시에 사용하는 job을 구성. 이 경우 각 서비스 별로 service table에서 제시하는 만큼의 concurrency를 제공하지 못할 거라는 예상이 가능

- 먼저 OCPU 모델에 대해 복합 서비스 작업 수행. 다음은 HIGH와 MEDIUM에 대해 비슷한 시간 대의 px_mon.sql 모니터링 레코드:
```
# HIGH

 EXECUTING DOWNGRADED     QUEUED  TOTAL_DOP    TOTL_PX PARALLEL_SERVERS_TARGET SYSTEM_IN_USE SYSTEM_AVAILABLE
---------- ---------- ---------- ---------- ---------- ----------------------- ------------- ----------------
        3          0          2         24         48                      96            96              612

# MEDIUM

 EXECUTING DOWNGRADED     QUEUED  TOTAL_DOP    TOTL_PX PARALLEL_SERVERS_TARGET SYSTEM_IN_USE SYSTEM_AVAILABLE
---------- ---------- ---------- ---------- ---------- ----------------------- ------------- ----------------
        6          0         14         24         48                      96            96              612
```
- Notes
    - 위 시점을 기준으로 보면,
        - 예상대로 concurrency 저하 발생 가능. 단 전반적으로 MEDIUM의 concurrency가 더 많이 관찰됨. 이는 물론 `share`의 작용 때문.
        - HIGH가 PX 서버를 48개, MEDIUM이 48개를 사용하여 총 96개를 사용함으로써 `parallel_servers_target`을 꽉 채움.
            - OCPU 모델의 `parllel_server_limit` 설정 상 각 서비스 단독으로는 자원의 100%를 쓰지 못하지만, 이렇게 동시에 사용할 때에는 보다 작업을 꽉꽉 채워 넣는 것이 가능

- 같은 작업을 이번에는 ECPU 모델에서 수행. 다음은 역시 HIGH와 MEDIUM에 대해 비슷한 시간 대의 px_mon.sql 모니터링 레코드:
```
# HIGH

 EXECUTING DOWNGRADED     QUEUED  TOTAL_DOP    TOTL_PX PARALLEL_SERVERS_TARGET SYSTEM_IN_USE SYSTEM_AVAILABLE
---------- ---------- ---------- ---------- ---------- ----------------------- ------------- ----------------
        3          0         15         48         96                      96            96              613

# MEDIUM

 EXECUTING DOWNGRADED     QUEUED  TOTAL_DOP    TOTL_PX PARALLEL_SERVERS_TARGET SYSTEM_IN_USE SYSTEM_AVAILABLE
---------- ---------- ---------- ---------- ---------- ----------------------- ------------- ----------------
        0          0         17          0          0                      96            96              613
```
- Notes
    - 전반적으로 OCPU 모델과 같은 양상. 다만 조금 다른 현상도 간간히 관찰됨. ECPU 모델의 HIGH는 `parallel_serverㄴ_limit`이 100%. 따라서 HIGH가 더 높은 `share`에 의해 dequeue가 좀 더 잘 된 어떤 시점에서 보면 HIGH 만으로 `parallel_servers_target`을 가득 채우게 되며, 이때 MEDIUM은 작업을 아예 수행하지 못하는 경우가 발생할 수도 있음

- 결론
    - Concurrency는 DOP의 함수값에 불과 

# 유연성의 보충 방안
- **ADW는 유연성이 부족**; 접속 서비스 별로 고정된 DOP와 그에 따른 고정된 concurrency
    - 보완책
        - MEDIUM 서비스에 한해 설정 변경 가능
        - 한 세션 내에서 DOP를 조정 가능
            - Consumer group의 switch
            - `parallel` 힌트의 사용

> 그 외 consumer group 간의 `share`를 변경하는 것도 지원되지만 실용성이 그리 높지 않아 본 문서에서는 다루지 않음
## MEDIUM 서비스의 DOP 및 Concurrency 조정
### 프로시저
- 다음 프로시저를 이용하여 MEDIUM의 DOP 및 concurrency를 customize할 수 있음:
```sql
begin
    cs_resource_manager.update_plan_directive(
        consumer_group => 'MEDIUM',
        concurrency_limit => <N>
    );
end;
/
```

- 원래의 설정으로 돌아오고 싶을 때에는 다음 프로시저를 이용한다.
```sql
begin
    cs_resource_manager.revert_to_default_values(
        consumer_group => 'MEDIUM',
        concurrency_limit => true
    );
end;
/
```
### Rules
#### Ground Rule
- **MEDIUM 서비스의 설정에 변경을 가하는 경우 `parallel_server_limit`은 즉시 100%로 변경**됨. 아래는 OCPU 모델의 예이며, 이는 ECPU에서도 동일함
```
-- 현재 service table

CONSUMER_GROUP                     SHARES CONCURRENCY        DOP
------------------------------ ---------- ----------- ----------
HIGH                                    4           3          8
MEDIUM                                  2          10          4
LOW                                     1        2400          1

-- 현재 plan directive

PLAN       GROUP_OR_SUBPLAN TYPE              MGMT_P1 ACTIVE_SESS_POOL_P1 PARALLEL_DEGREE_LIMIT_P1 PARALLEL_SERVER_LIMIT
---------- ---------------- -------------- ---------- ------------------- ------------------------ ---------------------
DWCS_PLAN  HIGH             CONSUMER_GROUP          4                                            8                    50
DWCS_PLAN  MEDIUM           CONSUMER_GROUP          2                                            4                    84    -- 변경 전
DWCS_PLAN  LOW              CONSUMER_GROUP          1                                            1
DWCS_PLAN  OTHER_GROUPS     CONSUMER_GROUP          1                                            1

-- 설정 변경

begin
    cs_resource_manager.update_plan_directive(
        consumer_group => 'MEDIUM',
        concurrency_limit => 5
    );
end;
/

-- 변경된 service table

CONSUMER_GROUP                     SHARES CONCURRENCY        DOP
------------------------------ ---------- ----------- ----------
HIGH                                    4           3          8
MEDIUM                                  2           5          9
LOW                                     1        2400          1

-- 변경된 plan directive

PLAN       GROUP_OR_SUBPLAN TYPE              MGMT_P1 ACTIVE_SESS_POOL_P1 PARALLEL_DEGREE_LIMIT_P1 PARALLEL_SERVER_LIMIT
---------- ---------------- -------------- ---------- ------------------- ------------------------ ---------------------
DWCS_PLAN  HIGH             CONSUMER_GROUP          4                                            8                    50
DWCS_PLAN  MEDIUM           CONSUMER_GROUP          2                                            9                   100    -- 변경 후
DWCS_PLAN  LOW              CONSUMER_GROUP          1                                            1
DWCS_PLAN  OTHER_GROUPS     CONSUMER_GROUP          1                                            1
```
#### 변경 가능 범위
> Documentation에서 제시. 하지만 ECPU 모델에 대해 documentation bug가 있으며, 아래 정리된 값들이 맞음

- OCPU 모델
    - $OCPUs \ge 2$인 경우에만 가능
    - Concurrency 변경 범위
        - w/o auto scaling: $1 \le concurrency \le 3 \times OCPUs$
        - w/ auto scaling: $1 \le concurrency \le 9 \times OCPUs$
    - DOP 변경 범위
        - w/o auto scaling: $2 \le DOP \le 2 \times OCPUs$
        - w/ auto scaling: $2 \le DOP \le 6 \times OCPUs$

- ECPU 모델
     - $ECPUs \ge 4$인 경우에만 가능
    - Concurrency 변경 범위
        - w/o auto scaling: $1 \le concurrency \le 0.75 \times ECPUs = 0.75 \times (4 \times OCPUs) = 3 \times OCPUs$
        - w/ auto scaling: $1 \le concurrency \le 2.25 \times ECPUs = 2.25 \times (4 \times OCPUs) = 9 \times OCPUs$
    - DOP 변경 범위
        - w/o auto scaling: $2 \le DOP \le 0.5 \times ECPUs = 0.5 \times (4 \times OCPUs) = 2 \times OCPUs$
        - w/ auto scaling: $2 \le DOP \le 1.5 \times ECPUs = 1.5 \times (4 \times OCPUs) = 6 \times OCPUs$

- Notes
    - 위에서 살펴본 바와 같이 **동등한 자원의 OCPU 모델과 ECPU 모델은 MEDIUM 서비스의 concurrency 및 DOP의 변경 범위가 완전히 동일**
#### Calculation Rules
- 전술한 바와 같이 fixed DOP를 사용하는 ADW 환경에서 concurrency는 다음 공식에 의해 계산
    - $Concurrency = Available\ PX\ servers \div Real\ DOP$ → concurrency는 DOP의 함수
    - 따라서 `cs_resource_manager.update_plan_directive`가 concurrency를 argument로 받지만 내부적으로는 다음 2 단계를 통해 처리함
        - 먼저 argument로 받은 concurrency에 의거하여 DOP를 계산
        - 계산된 DOP로 다시 concurrency를 보정 → 이에 따라 concurrency로 지정한 값과 다른 값을 얻을 수 있음

- 이 rule을 검증하기 위해 다음과 같은 Python code를 작성 (코드의 일부)
```python
    headers = [
        'CONC_IN',      # cs_resource_manager.update_plan_directive에 argument로 입력하는 concurrency
        'CONC',         # 실제 산출된 concurrency
        'DOP',          # 실제 산출된 DOP
        'DOP_CALC',     # 이 코드에 의해 계산된 DOP
        'CONC_CALC',    # 이 코드에 의해 계산된 concurrency
        'MAX_PX'        # 실제 사용할 수 있는 최대 PX 갯수 = concurrency * 2 * DOP = parallel_servers_target
    ]

    result = []

    # parallel_servers_target의 값 조회
    with oracledb.connect(**adb_connect_info) as connection:
        with connection.cursor() as cursor:
            cursor.execute("select value from v$parameter where name = 'parallel_servers_target'")
            parallel_servers_target = int(cursor.fetchone()[0])

    # 주어진 ADW의 compute model, compute_count (OCPU로 환산), auto scaling 여부 조회
    adb_info = database_client.get_autonomous_database(adb_ocid)
    compute_model = adb_info.data.compute_model
    compute_count = int(adb_info.data.cpu_core_count) if compute_model == "OCPU" else int(adb_info.data.compute_count / 4)
    is_auto_scaling_enabled = adb_info.data.is_auto_scaling_enabled

    # 변경 가능한 concurrency 및 DOP의 범위 정의
    min_conc = 1
    max_conc = 3 * compute_count if not is_auto_scaling_enabled else 9 * compute_count
    min_dop = 2
    max_dop = 2 * compute_count if not is_auto_scaling_enabled else 6 * compute_count

    with oracledb.connect(**adb_connect_info) as connection:
        with connection.cursor() as cursor:
            # Concurrency를 하나씩 증가시키며 loop
            for conc_in in range(min_conc, max_conc + 1):
                # DOP 계산: concurrency = available_PXs / real_DOP
                dop_calc = math.trunc(parallel_servers_target / (2 * conc_in))
                # 계산된 DOP를 min, max 범위에 cap
                dop_calc = max_dop if dop_calc > max_dop else dop_calc
                dop_calc = min_dop if dop_calc < min_dop else dop_calc
                # Concurrency를 min, max 범위에 맞게 재보정
                conc_calc = math.trunc(parallel_servers_target / (dop_calc * 2)) if dop_calc < max_dop else conc_in

                # 실제 MEDIUM 서비스를 변경한 후 결과값 조회
                cursor.execute(f"""
                    begin
                       cs_resource_manager.update_plan_directive(
                           consumer_group => 'MEDIUM',
                           concurrency_limit => {conc_in}
                       );
                    end;
                """)
                cursor.execute("""
                   select concurrency_limit, degree_of_parallelism
                     from cs_resource_manager.list_current_rules()
                    where consumer_group = 'MEDIUM'
                """)
                conc, dop = cursor.fetchone()

                # 모든 값을 하나의 레코드로 취합
                result.append([conc_in, conc, dop, dop_calc, conc_calc, conc * 2 * dop])

    # 전체 값들을 테이블 형태로 출력
    print(tabulate(result, headers=headers, tablefmt='pretty'))
```

- 수행 결과: no auto scaling
```
                     OCPU                                                        ECPU    
+---------+------+-----+----------+-----------+--------+    +---------+------+-----+----------+-----------+--------+
| CONC_IN | CONC | DOP | DOP_CALC | CONC_CALC | MAX_PX |    | CONC_IN | CONC | DOP | DOP_CALC | CONC_CALC | MAX_PX |
+---------+------+-----+----------+-----------+--------+    +---------+------+-----+----------+-----------+--------+
|    1    |  1   | 16  |    16    |     1     |   32   |    |    1    |  1   | 16  |    16    |     1     |   32   |
|    2    |  2   | 16  |    16    |     2     |   64   |    |    2    |  2   | 16  |    16    |     2     |   64   |
|    3    |  3   | 16  |    16    |     3     |   96   |    |    3    |  3   | 16  |    16    |     3     |   96   |
|    4    |  4   | 12  |    12    |     4     |   96   |    |    4    |  4   | 12  |    12    |     4     |   96   |
|    5    |  5   |  9  |    9     |     5     |   90   |    |    5    |  5   |  9  |    9     |     5     |   90   |
|    6    |  6   |  8  |    8     |     6     |   96   |    |    6    |  6   |  8  |    8     |     6     |   96   |
|    7    |  8   |  6  |    6     |     8     |   96   |    |    7    |  8   |  6  |    6     |     8     |   96   |
|    8    |  8   |  6  |    6     |     8     |   96   |    |    8    |  8   |  6  |    6     |     8     |   96   |
|    9    |  9   |  5  |    5     |     9     |   90   |    |    9    |  9   |  5  |    5     |     9     |   90   |
|   10    |  12  |  4  |    4     |    12     |   96   |    |   10    |  12  |  4  |    4     |    12     |   96   |
|   11    |  12  |  4  |    4     |    12     |   96   |    |   11    |  12  |  4  |    4     |    12     |   96   |
|   12    |  12  |  4  |    4     |    12     |   96   |    |   12    |  12  |  4  |    4     |    12     |   96   |
|   13    |  16  |  3  |    3     |    16     |   96   |    |   13    |  16  |  3  |    3     |    16     |   96   |
|   14    |  16  |  3  |    3     |    16     |   96   |    |   14    |  16  |  3  |    3     |    16     |   96   |
|   15    |  16  |  3  |    3     |    16     |   96   |    |   15    |  16  |  3  |    3     |    16     |   96   |
|   16    |  16  |  3  |    3     |    16     |   96   |    |   16    |  16  |  3  |    3     |    16     |   96   |
|   17    |  24  |  2  |    2     |    24     |   96   |    |   17    |  24  |  2  |    2     |    24     |   96   |
|   18    |  24  |  2  |    2     |    24     |   96   |    |   18    |  24  |  2  |    2     |    24     |   96   |
|   19    |  24  |  2  |    2     |    24     |   96   |    |   19    |  24  |  2  |    2     |    24     |   96   |
|   20    |  24  |  2  |    2     |    24     |   96   |    |   20    |  24  |  2  |    2     |    24     |   96   |
|   21    |  24  |  2  |    2     |    24     |   96   |    |   21    |  24  |  2  |    2     |    24     |   96   |
|   22    |  24  |  2  |    2     |    24     |   96   |    |   22    |  24  |  2  |    2     |    24     |   96   |
|   23    |  24  |  2  |    2     |    24     |   96   |    |   23    |  24  |  2  |    2     |    24     |   96   |
|   24    |  24  |  2  |    2     |    24     |   96   |    |   24    |  24  |  2  |    2     |    24     |   96   |
+---------+------+-----+----------+-----------+--------+    +---------+------+-----+----------+-----------+--------+
```
- Notes
    - 실제 값과 계산한 값이 완뵥히 동일 → 정확한 rule을 파악함
    - 같은 사양의 OCPU와 ECPU 모델의 결과는 완전히 동일
    - 가끔 `parallel_servers_target`을 가득 채우지 못하는 경우도 가능

- 수행 결과: auto scaling
```
                     OCPU                                                        ECPU    
+---------+------+-----+----------+-----------+--------+    +---------+------+-----+----------+-----------+--------+
| CONC_IN | CONC | DOP | DOP_CALC | CONC_CALC | MAX_PX |    | CONC_IN | CONC | DOP | DOP_CALC | CONC_CALC | MAX_PX |
+---------+------+-----+----------+-----------+--------+    +---------+------+-----+----------+-----------+--------+
|    1    |  1   | 48  |    48    |     1     |   96   |    |    1    |  1   | 48  |    48    |     1     |   96   |
|    2    |  2   | 48  |    48    |     2     |  192   |    |    2    |  2   | 48  |    48    |     2     |  192   |
|    3    |  3   | 48  |    48    |     3     |  288   |    |    3    |  3   | 48  |    48    |     3     |  288   |
|    4    |  4   | 36  |    36    |     4     |  288   |    |    4    |  4   | 36  |    36    |     4     |  288   |
|    5    |  5   | 28  |    28    |     5     |  280   |    |    5    |  5   | 28  |    28    |     5     |  280   |
|    6    |  6   | 24  |    24    |     6     |  288   |    |    6    |  6   | 24  |    24    |     6     |  288   |
|    7    |  7   | 20  |    20    |     7     |  280   |    |    7    |  7   | 20  |    20    |     7     |  280   |
|    8    |  8   | 18  |    18    |     8     |  288   |    |    8    |  8   | 18  |    18    |     8     |  288   |
|    9    |  9   | 16  |    16    |     9     |  288   |    |    9    |  9   | 16  |    16    |     9     |  288   |
|   10    |  10  | 14  |    14    |    10     |  280   |    |   10    |  10  | 14  |    14    |    10     |  280   |
|   11    |  11  | 13  |    13    |    11     |  286   |    |   11    |  11  | 13  |    13    |    11     |  286   |
|   12    |  12  | 12  |    12    |    12     |  288   |    |   12    |  12  | 12  |    12    |    12     |  288   |
|   13    |  13  | 11  |    11    |    13     |  286   |    |   13    |  13  | 11  |    11    |    13     |  286   |
|   14    |  14  | 10  |    10    |    14     |  280   |    |   14    |  14  | 10  |    10    |    14     |  280   |
|   15    |  16  |  9  |    9     |    16     |  288   |    |   15    |  16  |  9  |    9     |    16     |  288   |
|   16    |  16  |  9  |    9     |    16     |  288   |    |   16    |  16  |  9  |    9     |    16     |  288   |
|   17    |  18  |  8  |    8     |    18     |  288   |    |   17    |  18  |  8  |    8     |    18     |  288   |
|   18    |  18  |  8  |    8     |    18     |  288   |    |   18    |  18  |  8  |    8     |    18     |  288   |
|   19    |  20  |  7  |    7     |    20     |  280   |    |   19    |  20  |  7  |    7     |    20     |  280   |
|   20    |  20  |  7  |    7     |    20     |  280   |    |   20    |  20  |  7  |    7     |    20     |  280   |
|   21    |  24  |  6  |    6     |    24     |  288   |    |   21    |  24  |  6  |    6     |    24     |  288   |
|   22    |  24  |  6  |    6     |    24     |  288   |    |   22    |  24  |  6  |    6     |    24     |  288   |
|   23    |  24  |  6  |    6     |    24     |  288   |    |   23    |  24  |  6  |    6     |    24     |  288   |
|   24    |  24  |  6  |    6     |    24     |  288   |    |   24    |  24  |  6  |    6     |    24     |  288   |
|   25    |  28  |  5  |    5     |    28     |  280   |    |   25    |  28  |  5  |    5     |    28     |  280   |
|   26    |  28  |  5  |    5     |    28     |  280   |    |   26    |  28  |  5  |    5     |    28     |  280   |
|   27    |  28  |  5  |    5     |    28     |  280   |    |   27    |  28  |  5  |    5     |    28     |  280   |
|   28    |  28  |  5  |    5     |    28     |  280   |    |   28    |  28  |  5  |    5     |    28     |  280   |
|   29    |  36  |  4  |    4     |    36     |  288   |    |   29    |  36  |  4  |    4     |    36     |  288   |
|   30    |  36  |  4  |    4     |    36     |  288   |    |   30    |  36  |  4  |    4     |    36     |  288   |
|   31    |  36  |  4  |    4     |    36     |  288   |    |   31    |  36  |  4  |    4     |    36     |  288   |
|   32    |  36  |  4  |    4     |    36     |  288   |    |   32    |  36  |  4  |    4     |    36     |  288   |
|   33    |  36  |  4  |    4     |    36     |  288   |    |   33    |  36  |  4  |    4     |    36     |  288   |
|   34    |  36  |  4  |    4     |    36     |  288   |    |   34    |  36  |  4  |    4     |    36     |  288   |
|   35    |  36  |  4  |    4     |    36     |  288   |    |   35    |  36  |  4  |    4     |    36     |  288   |
|   36    |  36  |  4  |    4     |    36     |  288   |    |   36    |  36  |  4  |    4     |    36     |  288   |
|   37    |  48  |  3  |    3     |    48     |  288   |    |   37    |  48  |  3  |    3     |    48     |  288   |
|   38    |  48  |  3  |    3     |    48     |  288   |    |   38    |  48  |  3  |    3     |    48     |  288   |
|   39    |  48  |  3  |    3     |    48     |  288   |    |   39    |  48  |  3  |    3     |    48     |  288   |
|   40    |  48  |  3  |    3     |    48     |  288   |    |   40    |  48  |  3  |    3     |    48     |  288   |
|   41    |  48  |  3  |    3     |    48     |  288   |    |   41    |  48  |  3  |    3     |    48     |  288   |
|   42    |  48  |  3  |    3     |    48     |  288   |    |   42    |  48  |  3  |    3     |    48     |  288   |
|   43    |  48  |  3  |    3     |    48     |  288   |    |   43    |  48  |  3  |    3     |    48     |  288   |
|   44    |  48  |  3  |    3     |    48     |  288   |    |   44    |  48  |  3  |    3     |    48     |  288   |
|   45    |  48  |  3  |    3     |    48     |  288   |    |   45    |  48  |  3  |    3     |    48     |  288   |
|   46    |  48  |  3  |    3     |    48     |  288   |    |   46    |  48  |  3  |    3     |    48     |  288   |
|   47    |  48  |  3  |    3     |    48     |  288   |    |   47    |  48  |  3  |    3     |    48     |  288   |
|   48    |  48  |  3  |    3     |    48     |  288   |    |   48    |  48  |  3  |    3     |    48     |  288   |
|   49    |  72  |  2  |    2     |    72     |  288   |    |   49    |  72  |  2  |    2     |    72     |  288   |
|   50    |  72  |  2  |    2     |    72     |  288   |    |   50    |  72  |  2  |    2     |    72     |  288   |
|   51    |  72  |  2  |    2     |    72     |  288   |    |   51    |  72  |  2  |    2     |    72     |  288   |
|   52    |  72  |  2  |    2     |    72     |  288   |    |   52    |  72  |  2  |    2     |    72     |  288   |
|   53    |  72  |  2  |    2     |    72     |  288   |    |   53    |  72  |  2  |    2     |    72     |  288   |
|   54    |  72  |  2  |    2     |    72     |  288   |    |   54    |  72  |  2  |    2     |    72     |  288   |
|   55    |  72  |  2  |    2     |    72     |  288   |    |   55    |  72  |  2  |    2     |    72     |  288   |
|   56    |  72  |  2  |    2     |    72     |  288   |    |   56    |  72  |  2  |    2     |    72     |  288   |
|   57    |  72  |  2  |    2     |    72     |  288   |    |   57    |  72  |  2  |    2     |    72     |  288   |
|   58    |  72  |  2  |    2     |    72     |  288   |    |   58    |  72  |  2  |    2     |    72     |  288   |
|   59    |  72  |  2  |    2     |    72     |  288   |    |   59    |  72  |  2  |    2     |    72     |  288   |
|   60    |  72  |  2  |    2     |    72     |  288   |    |   60    |  72  |  2  |    2     |    72     |  288   |
|   61    |  72  |  2  |    2     |    72     |  288   |    |   61    |  72  |  2  |    2     |    72     |  288   |
|   62    |  72  |  2  |    2     |    72     |  288   |    |   62    |  72  |  2  |    2     |    72     |  288   |
|   63    |  72  |  2  |    2     |    72     |  288   |    |   63    |  72  |  2  |    2     |    72     |  288   |
|   64    |  72  |  2  |    2     |    72     |  288   |    |   64    |  72  |  2  |    2     |    72     |  288   |
|   65    |  72  |  2  |    2     |    72     |  288   |    |   65    |  72  |  2  |    2     |    72     |  288   |
|   66    |  72  |  2  |    2     |    72     |  288   |    |   66    |  72  |  2  |    2     |    72     |  288   |
|   67    |  72  |  2  |    2     |    72     |  288   |    |   67    |  72  |  2  |    2     |    72     |  288   |
|   68    |  72  |  2  |    2     |    72     |  288   |    |   68    |  72  |  2  |    2     |    72     |  288   |
|   69    |  72  |  2  |    2     |    72     |  288   |    |   69    |  72  |  2  |    2     |    72     |  288   |
|   70    |  72  |  2  |    2     |    72     |  288   |    |   70    |  72  |  2  |    2     |    72     |  288   |
|   71    |  72  |  2  |    2     |    72     |  288   |    |   71    |  72  |  2  |    2     |    72     |  288   |
|   72    |  72  |  2  |    2     |    72     |  288   |    |   72    |  72  |  2  |    2     |    72     |  288   |
+---------+------+-----+----------+-----------+--------+    +---------+------+-----+----------+-----------+--------+
```
- Notes
    - Auto scaling의 경우에도 정확히 rule을 따름
### MEDIUM 변경 후 동시 수행 
- 끝으로 MEDIUM 변경 후의 동시 수행 결과에도 이상이 없는지를 확인

- OCPU 모델에서의 px_mon.sql 결과
    - Auto scaling disable, tasks = 20
    - Concurreny argument = 7 → 결과 concurrency = 8,  DOP = 6
```
 EXECUTING DOWNGRADED     QUEUED  TOTAL_DOP    TOTL_PX PARALLEL_SERVERS_TARGET SYSTEM_IN_USE SYSTEM_AVAILABLE
---------- ---------- ---------- ---------- ---------- ----------------------- ------------- ----------------
        8          0         12         48         96                      96            96              611
```

- ECPU 모델에서의 px_mon.sql 결과
    - Auto scaling enable, tasks = 8
    - Concurreny argument = 4 → 결과 concurrency = 4,  DOP = 36
```
 EXECUTING DOWNGRADED     QUEUED  TOTAL_DOP    TOTL_PX PARALLEL_SERVERS_TARGET SYSTEM_IN_USE SYSTEM_AVAILABLE
---------- ---------- ---------- ---------- ---------- ----------------------- ------------- ----------------
        4          0          4        144        288                     288           289              420
```

- 모든 것이 규칙적으로 수행됨
## 한 세션 내에서 DOP의 조정
- 하나의 세션이 단 하나의 statement만 실행하고 접속을 끊는 경우는 거의 없으며 대부분 다수의 statement들을 순차적으로 수행하게 됨. e.g. 다수의 statement 들을 수행하는 배치 프로그램
- 이때 각 statement들의 자원 사용량은 얼마든지 다를 수 있음. 하지만 이미 특정 서비스에 접속해 있다면 그 서비스가 제공하는 고정된 DOP에 묶이게 됨.  
- ADW는 이 제한을 회피할 수 있는 두가지 방법을 제공
    - 동적으로 concumer group을 원하는 그룹으로 switch
    - `Parallel` 힌트의 사용
### Consumer Group의 Switching
- 먼저 OCPU 모델의 HIGH 서비스에 접속
```sql
connect admin/WElcome123__@adw19o_high
@whoami
```

```

       SID    SERIAL#     OS PID USERNAME SCHEMA   MACHINE          PROGRAM          CDB              CON_NAME           INSTANCE SERVICE          MODULE           CGROUP
---------- ---------- ---------- -------- -------- ---------------- ---------------- ---------------- ---------------- ---------- ---------------- ---------------- ----------------
     34864      45324     384416 ADMIN    ADMIN    SANGLEE-PF1FSXNM sqlplus@SANGLEE- e491pod          YH0OLYBN5PQCE4N_          2 YH0OLYBN5PQCE4N_ SQL*Plus         HIGH
                                                                    PF1FSXNM (TNS V1                  ADW19O                      ADW19O_high.adb.
                                                                    -V3)                                                          oraclecloud.com
```

- 세션 작업 수행 중 어떤 다음 task에 HIGH 서비스 만큼의 자원이 필요하지 않다고 가정.  이때 다음 방법을 통해 세션 재접속 없이 다른 consumer group으로 switch 가능
```sql
begin
    cs_session.switch_service('MEDIUM');
end;
/

@whoami
```

```
       SID    SERIAL#     OS PID USERNAME SCHEMA   MACHINE          PROGRAM          CDB              CON_NAME           INSTANCE SERVICE          MODULE           CGROUP
---------- ---------- ---------- -------- -------- ---------------- ---------------- ---------------- ---------------- ---------- ---------------- ---------------- ----------------
     34864      45324     384416 ADMIN    ADMIN    SANGLEE-PF1FSXNM sqlplus@SANGLEE- e491pod          YH0OLYBN5PQCE4N_          2 YH0OLYBN5PQCE4N_ SQL*Plus         MEDIUM
                                                                    PF1FSXNM (TNS V1                  ADW19O                      ADW19O_medium.ad
                                                                    -V3)                                                          b.oraclecloud.co
                                                                                                                                  m
```

- 이 방법은 ECPU 모델에서도 동일하게 동작
```sql
connect admin/WElcome123__@adw19e_low
@whoami
```

```
       SID    SERIAL#     OS PID USERNAME SCHEMA   MACHINE          PROGRAM          CDB              CON_NAME           INSTANCE SERVICE          MODULE           CGROUP
---------- ---------- ---------- -------- -------- ---------------- ---------------- ---------------- ---------------- ---------- ---------------- ---------------- ----------------
     30057      14754      45541 ADMIN    ADMIN    SANGLEE-PF1FSXNM sqlplus@SANGLEE- e491pod          YH0OLYBN5PQCE4N_          2 YH0OLYBN5PQCE4N_ SQL*Plus         LOW
                                                                    PF1FSXNM (TNS V1                  ADW19E                      ADW19E_low.adb.o
                                                                    -V3)                                                          raclecloud.com
```

```sql
begin
    cs_session.switch_service('MEDIUM');
end;
/

@whoami
```

```

       SID    SERIAL#     OS PID USERNAME SCHEMA   MACHINE          PROGRAM          CDB              CON_NAME           INSTANCE SERVICE          MODULE           CGROUP
---------- ---------- ---------- -------- -------- ---------------- ---------------- ---------------- ---------------- ---------- ---------------- ---------------- ----------------
     30057      14754      45541 ADMIN    ADMIN    SANGLEE-PF1FSXNM sqlplus@SANGLEE- e491pod          YH0OLYBN5PQCE4N_          2 YH0OLYBN5PQCE4N_ SQL*Plus         MEDIUM
                                                                    PF1FSXNM (TNS V1                  ADW19E                      ADW19E_medium.ad
                                                                    -V3)                                                          b.oraclecloud.co
                                                                                                                                  m
```

- 응용
    - 다수의 statement들을 수행하는 배치 프로그램 중간 중간에 `cs_session.switch_service`에 대한 call을 포함시키면 자원의 최대한 효율적인 사용을 도모할 수 있음
### Parallel Hint의 사용
- ADW에서는 `parallel` 힌트를 통해 statement 단위로 fixed DOP가 아닌 다른 DOP를 쓸 수 있음. **단 접속 서비스의 fixed DOP보다 낮은 degree만 가능**

- 먼저 `parallel` 힌트를 사용할 수 있도록 설정 필요:
```sql
alter system set optimizer_ignore_parallel_hints = false;
```
- Notes
    - 이 설정은 세션 레벨도 가능

- 우선 OCPU 모델에서 테스트. 앞서 사용한 q1.sql에 아래와 같이 힌트를 줌. 이때 degree는 HIGH DOP 8의 절반인 4를 주고 HIGH에 접속한 후 먼저 예상 실행 계획을 확인:
```sql
@estb
select /*+ parallel(4) */
       n1.n as group_id,
       count(*) as group_size,
       round(sum(power(n2.n, 3) * n3.rand_val)/1000000, 2) as weighted_cubic_millions,
       round(avg(power(n2.n * n3.rand_val, 2)), 2) as weighted_square_avg,
       round(stddev(n2.n * n3.rand_val), 2) as weighted_std_dev,
       round(sum(case when mod(n2.n + n3.n, 2) = 0 then power(n2.rand_val * n3.rand_val, 2)
                 else sqrt(abs(n2.rand_val - n3.rand_val)) end), 2) as conditional_sum
      from numbers n1
cross join numbers n2
cross join numbers n3
where n1.n <= 30
group by n1.n
order by n1.n;
@este
```

```
Note
-----
   - Degree of Parallelism is 4 because of hint
```

- 실제 실행 역시 DOP 4로 수행됨
```
STATUS          START    END      DURATION SQL_ID          SQL_EXEC_ID        PHV USER     PROGRAM          MODULE           ACTION           DOP      SQL_TEXT
--------------- -------- -------- -------- --------------- ----------- ---------- -------- ---------------- ---------------- ---------------- -------- -------------------------------
...
EXECUTING       16:15:26                   b7g08avnaus2m      33554432  842082734 ADMIN    sqlplus@SANGLEE- SQL*Plus                          8(1)     select /*+ parallel(4) */
```

- 이번에는 degree를 HIGH의 fixed DOP 8보다 큰 16으로 지정한 후 예상 실행 계획을 확인하면 → 역시 힌트가 잘 듣는 것처럼 보임:
```
Note
-----
   - Degree of Parallelism is 16 because of hint
```

- 하지만 실제 실행을 해보면 DOP는 결국 HIGH의 fixed DOP인 8로 downgrade:
```
STATUS          START    END      DURATION SQL_ID          SQL_EXEC_ID        PHV USER     PROGRAM          MODULE           ACTION           DOP      SQL_TEXT
--------------- -------- -------- -------- --------------- ----------- ---------- -------- ---------------- ---------------- ---------------- -------- -------------------------------
EXECUTING       16:17:17                   46t1qkjv24ac7      33554434  842082734 ADMIN    sqlplus@SANGLEE- SQL*Plus                          16(1-D)  select /*+ parallel(16) */
```

- 이상의 동작은 ECPU 모델에서도 정확히 동일
```sql
-- 예상 실행 계획

Note
-----
   - Degree of Parallelism is 32 because of hint


-- 실제 실행 시 SQL Monitor

STATUS          START    END      DURATION SQL_ID          SQL_EXEC_ID        PHV USER     PROGRAM          MODULE           ACTION           DOP      SQL_TEXT
--------------- -------- -------- -------- --------------- ----------- ---------- -------- ---------------- ---------------- ---------------- -------- ------------------------------------
EXECUTING       16:33:00                   10vsx1wvbfddh      33554433  842082734 ADMIN    sqlplus@SANGLEE- SQL*Plus                          32(1-D)  select /*+ parallel(32) */
```

#### Serial 처리의 경우
- 한편 아예 serial 처리가 더 효율적인 stetement들도 있음. PX coordinating의 overhead가 statement 자체의 처리보다 클 수 있기 때문. 이를 위한 `noparallel` 힌트도 쓸 수 있을까? 먼저 `noparallel` 힌트를 쓴 statement의 예상 실행 계획은 다음과 같음:
```
Hint Report (identified by operation id / Query Block Name / Object Alias):
Total hints for statement: 1 (U - Unused (1))
---------------------------------------------------------------------------

   0 -  STATEMENT
         U -  noparallel

Note
-----
   - Degree of Parallelism is 1 because of hint     
```
- Notes
    - 약간 모순되는 output 

- 하지만 실제 실행을 하면 serial로 잘 처리됨됨
```
STATUS          START    END      DURATION SQL_ID          SQL_EXEC_ID        PHV USER     PROGRAM          MODULE           ACTION           DOP      SQL_TEXT
--------------- -------- -------- -------- --------------- ----------- ---------- -------- ---------------- ---------------- ---------------- -------- ---------------------------------------
...
EXECUTING       16:44:05          00:00:10 fmdas5jxu3tda      33554433  745804083 ADMIN    sqlplus@SANGLEE- SQL*Plus                                   select /*+ noparallel */
```

- `noparallel`힌트에 대한 실행 계획 및 실제 실행에 관해 OCPU와 ECPU 모델은  모두 동일한 동작을 보여줌

#### Parallel Hint의 사용과 Concurrency
- 혹시 `parallel` hint에 의해 concurrency가 영향을 받을 수 있을까? 우선 OCPU 모델의 HIGH에 접속하여 동시 수행 작업을 돌리되, 힌트를 통해 DOP를 fixed 값인 8의 절반인 4로 낮춘 버전을 사용. 다음은 px_mon.sql의 결과:
```
 EXECUTING DOWNGRADED     QUEUED  TOTAL_DOP    TOTL_PX PARALLEL_SERVERS_TARGET SYSTEM_IN_USE SYSTEM_AVAILABLE
---------- ---------- ---------- ---------- ---------- ----------------------- ------------- ----------------
        6          0          4         24         48                      96            58              654
```
- Notes
    - **힌트를 통하여 degree를 대상 서비스의 fixed DOP보다 낮게 주면 concurrency가 증가**
    - Concurrency는 역시 DOP에 의해 결정됨을 확인할 수 있음. 위 결과를 보면 PX 서버 갯수는 48로 `parallel_servers_target`의 50%. 즉 `parallel_servers_limit`은 변함없이 준수되고 있음

- ECPU 모델에 대해 같은 테스트를 해도 결과는 동일
```
 EXECUTING DOWNGRADED     QUEUED  TOTAL_DOP    TOTL_PX PARALLEL_SERVERS_TARGET SYSTEM_IN_USE SYSTEM_AVAILABLE
---------- ---------- ---------- ---------- ---------- ----------------------- ------------- ----------------
        6          0          4         48         96                      96           104              607
```

- 끝으로 `noparallel` 힌트를 테스트해보자. OCPU 모델의 HIGH 서비스를 이용하는 job을 다음과 같이 정의
```
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_high q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_high q1_serial.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_high q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_high q1_serial.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_high q1.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_high q1_serial.sql
runsql.sh -j adw_test -c admin/WElcome123__@adw19o_high q1.sql
```
- Notes
    - q1_serial.sql은 기존의 q1.sql에 `noparallel` 힌트를 준 것
    - Parallel task 4개, serial task 3개

- Job을 수행
```
-- SQL Monitor 결과

STATUS          START    END      DURATION SQL_ID          SQL_EXEC_ID        PHV USER     PROGRAM          MODULE           ACTION           DOP      SQL_TEXT
--------------- -------- -------- -------- --------------- ----------- ---------- -------- ---------------- ---------------- ---------------- -------- -----------------------------------------
QUEUED          17:14:08                   36qxdga5srwt6      33554439  842082734 ADMIN    sqlplus@SANGLEE- adw_test         q1.sql                    select
EXECUTING       17:14:08          00:00:01 36qxdga5srwt6      33554436  842082734 ADMIN    sqlplus@SANGLEE- adw_test         q1.sql           16(1)    select
EXECUTING       17:14:08          00:00:01 36qxdga5srwt6      33554438  842082734 ADMIN    sqlplus@SANGLEE- adw_test         q1.sql           16(1)    select
EXECUTING       17:14:08          00:00:01 36qxdga5srwt6      33554437  842082734 ADMIN    sqlplus@SANGLEE- adw_test         q1.sql           16(1)    select
EXECUTING       17:14:08          00:00:14 fmdas5jxu3tda      33554435  745804083 ADMIN    sqlplus@SANGLEE- adw_test         q1_serial.sql             select /*+ noparallel */
EXECUTING       17:14:08          00:00:14 fmdas5jxu3tda      33554436  745804083 ADMIN    sqlplus@SANGLEE- adw_test         q1_serial.sql             select /*+ noparallel */
EXECUTING       17:14:08          00:00:14 fmdas5jxu3tda      33554437  745804083 ADMIN    sqlplus@SANGLEE- adw_test         q1_serial.sql             select /*+ noparallel */

-- px_mon.sql 결과

 EXECUTING DOWNGRADED     QUEUED  TOTAL_DOP    TOTL_PX PARALLEL_SERVERS_TARGET SYSTEM_IN_USE SYSTEM_AVAILABLE
---------- ---------- ---------- ---------- ---------- ----------------------- ------------- ----------------
        6          0          1         24         48                      96            48              663
```
- Notes
    - Documentation에는 serail 처리가 되는 LOW 서비스는 queue와 무관하다고 기술. 이는 serial 처리 시 PX 서버를 전혀 사용하지 않으므로 당연한 것. **`noparallel` 힌트도 정확히 LOW와 같이 동작**
        - 4 개의 parallel task가 동시에 요청되어 3개가 실행, 1개가 queued
        - 3개의 serial task들은 queueing과 상관없이 진행
    - 이 behavior는 ECPU 모델에서도 그대로임

# Summary
> 현재 19c 버전까지만 테스트하였으므로 아래는 잠정적인 결론임

- **ADW 19c의 workload management는 대체로 document에 제시된 대로 잘 동작하는 것으로 보임**
    - 하지만 단 한번 parallel statement queueing이 무너지고 DOP downgrade가 발생하는 현상을 관찰한 적이 있음

- **하지만 ADW의 근본적인 약점은 full auto DOP가 아니라는 점.** 즉 서비스마다 fixed DOP, 그 결과 fixed concurrency를 사용해야 한다는 것
    - MEDIUM 서비스의 변경, consumer group의 switch, `parallel` 힌트의 사용 등의 보완책이 있지만 이 모두가 manual 개입을 필요로 하므로 **충분히 autonomous하지 않음**
        - **오히려 autonomous라는 용어를 사용하지도 않는 Big Query가 훨씬 더 autonomous한 특징들을 갖고 있는 것으로 보임.**
            - 비단 Big Query 뿐 아니라 다른 경쟁 서비스들도 대체로 비슷

- 바라는 방향 (IMHO)
    - **Full-auto DOP를 적용**하여 job들을 아무 생각없이 던져도 유연하고 원활하게 수행할 수 있도록 (진정한 "load and run")
    - **HIGH 및 MEDIUM 서비스의 설정을 동등하게**
        - 동일한 `parallel_degree_limit`
        - 둘 다 `parallel_server_limit`을 100으로 설정하여 `parallel_servers_target`(= `parallel_max_servers`)를 모두 사용할 수 있도록
        - 이렇게 하더라도 둘 사이의 우선 순위 조율은 여전히 `share`에 의해 가능
    - 이렇게 되면 유연성 보완을 위한 MEDIUM 서비스의 변경 등과 같은 기능은 불필요

- (현재 기능이 유지된다는 전제 하에) 향후 deal 또는 POC/BMT에 대한 견해 (IMHO)
    - **현재의 ADW는 적어도 기술적으로 Big Query와 같은 유연한 서비스를 대체하기 쉽지 않은 듯 함**
        - 그래서 SSG.com 사례처럼 타 서비스의 workload를 "그대로 가져와 수행"하는 형태의 PoC/BMT는 애초에 무리수였다고 봄
            - 우리가 최선을 다해 수행을 마쳤다 하더라도 우리의 형태, 우리의 방식에 대해 고객이 accept하기를 기대하기는 어렵다고 봄
                - e.g.
                    - 시나리오마다 MEDIUM 설정을 바꾸기
                    - `parallel` 힌트의 사용
                        - 일반적인 힌트는 SQL tuning 관점에서 납득이 가는 일이지만 workload management마저도 힌트에 의존해야 한다?
    - 따라서 향후에는 **기술적/영업적 사전 communication에 의해 우리의 모습을 사전에 적정 수준에서 open하는 것이 우선. 그리고 그 상태에서 고객을 충분히 설득**하는 과정이 선행되어야 하지 않을까... 
        - 그렇게 되면 PoC/BMT를 하더라도 우리에게 적합한 "정제된 workload"를 받고, 수행 역시 우리에게 적합한 방식을 사용할 수 있는 가능성이 열림

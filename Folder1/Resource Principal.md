- Created: `=dateformat(this.file.ctime, "DDDD, HH:mm")`
- Last Modified: `=dateformat(this.file.mtime, "DDDD, HH:mm")`
- Tags: #AutonomousDB #ADW 
---
#### Task 3-2: Resource Principal 설정
- Object Storage **vec_bucket** 접근을 위해 필요
    - 사전 일괄 작업의 편의를 위해서임. (Resource principal을 쓰지 않는 경우 각 OCI 사용자 별로 각 ADW 안에 credential을 만들어 주어야 함) 

- 아래 두 IAM 설정은 이미 해 둔 상태
    - Resource principal 사용을 위한 dynamic group **oci-bootcamp-dynamic-group** 생성. Matching rule:
        - `ALL {resource.type = 'autonomousdatabase', resource.compartment.id = '<oci-bootcamp compartment OCID>'}`
    - **oci-bootcamp-dynamic-group**에 대한 다음 policy statement를 **oci-bootcamp-policy**에 추가
        - `allow dynamic-group oci-bootcamp-dynamic-group to manage object-family in compartment oci-bootcamp`

- 각 ADW에 접속하여 다음과 같이 resource principal enable
```sql
BEGIN
    dbms_cloud_admin.enable_resource_principal();
    dbms_cloud_admin.enable_resource_principal(username => 'VECTOR');
END;
/

--
-- 결과 확인

SELECT owner, credential_name
  FROM dba_credentials
 WHERE credential_name = 'OCI$RESOURCE_PRINCIPAL'
   AND owner = 'ADMIN';

SELECT grantee, table_name, grantor
  FROM all_tab_privs
 WHERE grantee = 'VECTOR'
   AND table_name = 'OCI$RESOURCE_PRINCIPAL';
```

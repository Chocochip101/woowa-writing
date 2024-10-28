# 커버링 인덱스로 100만 건 데이터 환경에서 API 응답 속도 90% 개선하기

팀 크루루는 서비스 성장을 가정하고, 데이터베이스에 대량의 데이터를 삽입하여 성능을 개선하는 작업을 진행하고 있습니다. 팀 내에서 대량의 데이터 환경에서 API 응답 속도를 더 빠르게 처리하기 위해 접근했던 방법과 이를 해결하기 위해 커버링 인덱스를 적용하고 성능을 개선한 경험담을 공유하려고 합니다.
## 예상 독자

1. API의 병목 현상에 어떻게 접근하고 해결해야 할지 궁금한 분들
2. Join이 포함된 쿼리에서 성능 최적화를 고민하고 계신 분들
3. N+1 문제와 Lazy Loading으로 인해 발생하는 비효율을 해결하고 싶은 분들
4. MySQL에서 인덱스를 적용해 쿼리 성능을 개선하고자 하는 분들
5. 100만 건 데이터 처리 중 쿼리 개수 및 처리 시간을 줄이기 위한 최적화 방법을 찾고 계신 분들
6. 커버링 인덱스와 복합 인덱스가 실제 성능에 얼마나 영향을 미치는지 궁금한 분들

# 배경
현재 팀 크루루는 서비스 성장을 가정하고, 데이터베이스에 대량의 더미 데이터를 삽입하여 기능이 원활하게 동작할 수 있도록 성능을 개선하는 작업을 진행하고 있습니다. 대규모 데이터를 처리하는 환경에서 성능을 최적화하기 위해, 데이터를 구성하고 테스트를 진행했습니다.
## 선행 작업

테스트를 위해 다음과 같은 테이블 구성과 레코드 수를 기준으로 데이터를 준비했습니다.


| 테이블 | 레코드 수 | 참고 |
| --- | --- | --- |
| MEMBER | 500 | - |
| CLUB | 500 | - |
| DASHBOARD | 1,500 | Club 하나당 3개의 Dashboard |
| APPLYFORM | 1,500 | - |
| PROCESS | 6,000 | Dashboard 하나당 4개의 Process |
| APPLICANT | 200,000 | - |
| QUESTION | 7,500 | - |
| CHOICE | 15,000 | - |
| ANSWER | 1,000,000 | - |
| EVALUATION | 1,000,000 | - |

위 세 가지 방법 중 LOAD DATA LOCAL INFILE 명령어를 사용하여 CSV 파일을 임포트하는 방법을 선택했습니다. 이 방법을 통해 100만 건의 데이터를 단 6초 만에 삽입할 수 있었습니다.
## 대량 데이터 삽입 방법
데이터베이스에 대량 데이터를 삽입하는 두 가지 방법을 고려하였습니다.

### 엑셀로 랜덤 데이터 생성 후 INSERT문을 통한 삽입
엑셀을 이용해 랜덤 데이터를 생성한 후, 이를 INSERT문으로 변환하여 하나씩 데이터를 삽입하는 방법입니다. 엑셀의 랜덤 함수와 텍스트 결합 기능을 이용해 대량 데이터를 생성할 수 있습니다.
엑셀 함수로 원하는 형태의 데이터를 세밀하게 생성할 수 있습니다. 수작업으로 INSERT문을 생성하고 실행해야 하므로, 대량 데이터를 삽입하는 데 시간이 너무 오래 걸립니다. 100만 건을 삽입하는 데 몇 시간이 소요될 수 있습니다.

### Bulk INSERT 사용
여러 행의 데이터를 한 번에 삽입하는 Bulk INSERT를 사용하면, INSERT문을 여러 번 사용하지 않고도 데이터를 빠르게 넣을 수 있습니다. INSERT문을 여러 번 사용하지 않고도 대량 데이터를 한 번에 삽입할 수 있습니다. 직접 쿼리문을 생성해야 하며, 쿼리 작성 비용이 큽니다. 대량 데이터를 다룰 때 여전히 비효율적입니다.

### MySQL에서 LOAD DATA LOCAL INFILE 사용
MySQL의 LOAD DATA LOCAL INFILE 명령어는 CSV 파일을 매우 빠르게 임포트할 수 있는 방법입니다. MySQL Workbench에서 이 구문을 실행하면, 파일에서 테이블로 대량 데이터를 효율적으로 삽입할 수 있습니다. 다른 방법에 비해 매우 빠르게 대량 데이터를 삽입할 수 있습니다.


위 세 가지 방법 중, LOAD DATA 명령어를 사용하여 CSV 파일을 임포트하는 방법을 선택하였고, 이를 통해 100만 건의 데이터를 6초만에 삽입했습니다.

# API 실행 시간 관측

API의 실행 시간을 Grafana 모니터링 툴로 관측해보겠습니다.

![image.png](https://github.com/user-attachments/assets/80826272-b0a8-4a84-b8e4-9987d9baabaf)


![image](https://github.com/user-attachments/assets/61c5967f-e3e8-42a3-8288-0b9a63f057c9)

실행 시간이 1초를 넘는 API가 있습니다!  `PATCH /v1/processes/{processes}`, `GET /v1/processes` API를 튜닝해 보겠습니다.

---

# 쿼리 성능 개선

먼저 `GET /v1/processes` 에서 실행되는 쿼리를 살펴봅시다.

```sql
select * from member m1_0 where m1_0.email=?
select d1_0.dashboard_id,d1_0.club_id from dashboard d1_0 where d1_0.dashboard_id=?
select * from club c1_0 where c1_0.club_id=?
select * from apply_form af1_0 where af1_0.dashboard_id=?
select * from process p1_0 where p1_0.dashboard_id=?
select * from applicant a1_0 where a1_0.process_id=?
select count(e1_0.evaluation_id) from evaluation e1_0 where e1_0.applicant_id=? and e1_0.process_id=?
select * from evaluation e1_0 where e1_0.process_id=? and e1_0.applicant_id=?
select count(e1_0.evaluation_id) from evaluation e1_0 where e1_0.applicant_id=? and e1_0.process_id=?
select * from evaluation e1_0 where e1_0.process_id=? and e1_0.applicant_id=?
// ...
select count(e1_0.evaluation_id) from evaluation e1_0 where e1_0.applicant_id=? and e1_0.process_id=?
select * from evaluation e1_0 where e1_0.process_id=? and e1_0.applicant_id=?
select count(e1_0.evaluation_id) from evaluation e1_0 where e1_0.applicant_id=? and e1_0.process_id=?
select * from evaluation e1_0 where e1_0.process_id=? and e1_0.applicant_id=?
```

총 200여 개의 쿼리가 실행되었습니다.

Lazy Loading으로 인한 N+1 쿼리 발생과 로직의 비효율성 때문에 `select count(e1_0.evaluation_id)` 및 `select e1_0.evaluation_id, ...` 쿼리가 많이 반복되어 실행되는 것을 알 수 있습니다. 

먼저 인덱스를 적용하여 반복되는 단건 조회 쿼리의 처리 속도를 향상해 보겠습니다.

## 인덱스를 적용하여 단건 조회 쿼리의 처리 속도 개선하기

200개의 쿼리들의 공통점은 **where에 걸린 조건**입니다.

```sql
where e1_0.applicant_id=? and e1_0.process_id=?
```

`applicant_id` 와  `process_id` 칼럼을 조건으로 지정하고 있으므로 복합 인덱스를 적용하기로 했습니다. 

그러면 여기서 인덱스의 칼럼 순서는 어떻게 결정해야 할까요? 

 이를 알아보기 위해 (process_id, applicant_id) 인덱스와 (applicant_id, process_id) 인덱스를 생성하여 비교하였습니다.


![image](https://github.com/user-attachments/assets/0c2b1b8f-e661-4ab7-a976-7b48e932cd80)

### 인덱스 1. (process_id, applicant_id)

`(process_id, applicant_id)` 인덱스를 적용한 후 실행 계획을 살펴봅시다.


![image](https://github.com/user-attachments/assets/c60d92d2-6063-4faf-8ca3-3471279ebb74)

주목해야할 칼럼은 possible_keys입니다. possible_keys는 옵티마이저가 최적의 실행 계획을 수립하기 위해 고려한 여러 접근 방법 중에서 사용할 수 있는 인덱스들의 목록입니다 possible_keys가 NULL이기 때문에 옵티마이저가 어떠한 인덱스도 고려하지 않았다는 것을 알 수 있습니다.

따라서 해당 쿼리는 테이블을 처음부터 끝까지 읽는 Full Table Scan으로 실행됩니다.

### 인덱스 2. (applicant_id, process_id)

이번에는 `(applicant_id, process_id)` 인덱스를 적용하여 실행 계획을 살펴보겠습니다.

![image](https://github.com/user-attachments/assets/19c443fe-c650-4121-acd7-a983b96dcea9)

이전 실행 계획과는 다르게 type, key가 변경되었습니다. type이 ref로 변경되어 인덱스를 사용했으며, 여러 인덱스 중에서 저희가 생성한 `(applicant_id, process_id)` 인덱스를 사용하였음을 알 수 있습니다. 


![image](https://github.com/user-attachments/assets/22dc0b21-957c-417e-b0be-825d7007fad0)

실행 시간이 1초 대에서 약 350ms로 많이 줄었습니다. 하지만 해당 관측 결과는 **네트워크 통신 속도를 제외한 결과**이기 때문에 실제 실행 시간은 더 오래 걸릴 수 있습니다.

따라서 최적화가 더 필요하지만, 단일 쿼리 최적화로는 더 이상 속도를 줄이지 못한다고 판단했습니다.

## API 로직 튜닝

API 하나당에서 실행되는 쿼리의 수를 줄이기 위해 로직을 수정했습니다.

이전 로직은 다음과 같습니다.

1. dashboardId로 process 조회
2. process마다 존재하는 applicant 조회 (N+1)
3. applicant마다 evaluation 평균 점수 조회

N+1 문제로 쿼리가 많이 발생하고, 평균 점수를 반복해서 조회하기 때문에 쿼리의 개수가 기하급수적으로 증가합니다.

따라서 로직을 다음과 같이 수정했습니다.

1. **평균 점수 및 개수**를 projection으로 변환
2. **Fetch Join**을 활용하여 process, applicant 테이블을 한번에 조회
3. **IN 절**을 사용해 여러 프로세스를 한 번에 조회

해당 로직으로 변경한 결과, 데이터의 수와 관계없이 `GET /v1/processes` 요청 시 인가 쿼리 2개와 API 로직으로 발생하는 쿼리 4개로 처리할 수 있습니다.

```sql
-- 회원 정보 조회
select *
from 
    member m1_0 
where 
    m1_0.email = ?;

-- 대시보드 ID로 대시보드 조회
select 
    d1_0.dashboard_id, 
    d1_0.club_id 
from 
    dashboard d1_0 
where 
    d1_0.dashboard_id = ?;

-- 클럽 ID로 클럽 정보 조회
select *
from 
    club c1_0 
where 
    c1_0.club_id = ?;

-- 대시보드 ID로 신청서 조회
select 
    af1_0.apply_form_id, 
    af1_0.created_date, 
    d1_0.dashboard_id, 
    d1_0.club_id, 
    af1_0.description, 
    af1_0.end_date, 
    af1_0.start_date, 
    af1_0.title, 
    af1_0.updated_date 
from 
    apply_form af1_0 
    join dashboard d1_0 on d1_0.dashboard_id = af1_0.dashboard_id 
where 
    d1_0.dashboard_id = ?;

-- 대시보드 ID로 프로세스 정보 조회
select *
from 
    process p1_0 
    left join dashboard d1_0 on d1_0.dashboard_id = p1_0.dashboard_id 
where 
    d1_0.dashboard_id = ?;

-- 여러 프로세스 ID에 해당하는 지원자 정보 조회
select 
    a1_0.applicant_id, 
    a1_0.name, 
    a1_0.created_date, 
    a1_0.is_rejected, 
    count(e1_0.evaluation_id), 
    coalesce(avg(e1_0.score), 0.00), 
    a1_0.process_id 
from 
    applicant a1_0 
    left join evaluation e1_0 on e1_0.applicant_id = a1_0.applicant_id 
where 
    a1_0.process_id in (?,?,?,?) 
group by 
    a1_0.applicant_id, 
    a1_0.name, 
    a1_0.created_date, 
    a1_0.is_rejected, 
    a1_0.process_id; 
```

API 로직을 개선한 후에, 인덱스 없이 API의 실행 시간을 살펴보면 약 700ms로 감소한 것을 확인할 수 있습니다.


![image](https://github.com/user-attachments/assets/ffe4f056-8511-4717-82d8-b5f10913c7ae)


![image](https://github.com/user-attachments/assets/1b7b5641-fa0e-4428-b8ae-77d6f65a185c)

그러나 API의 실행 시간은 여전히  700~800ms로 만족스럽지 않습니다. API 로직으로 발생하는 4개의 쿼리 중 병목이 발생할 수 있는 GROUP BY 절과 JOIN 절이 있는 쿼리를 살펴봤습니다.

```sql
-- 여러 프로세스 ID에 해당하는 지원자 정보 조회
select 
    a1_0.applicant_id, 
    a1_0.name, 
    a1_0.created_date, 
    a1_0.is_rejected, 
    count(e1_0.evaluation_id), 
    coalesce(avg(e1_0.score), 0.00), 
    a1_0.process_id 
from 
    applicant a1_0 
    left join evaluation e1_0 on e1_0.applicant_id = a1_0.applicant_id 
where 
    a1_0.process_id in (?,?,?,?) 
group by 
    a1_0.applicant_id, 
    a1_0.name, 
    a1_0.created_date, 
    a1_0.is_rejected, 
    a1_0.process_id; 
```

해당 쿼리의 실행 계획은 아래와 같습니다.

![image](https://github.com/user-attachments/assets/b6885c5c-ba59-4bc9-928c-2e96fb6c0f84)


type이 ALL이기 때문에 Full Table Scan으로 실행되고, 어떠한 인덱스도 적용되지 않았습니다. 또한 rows 칼럼에서 994,125행을 스캔해야 한다고 예측하고 있습니다.

이어서 `EXPLAIN ANALYZE`를  실행한 결과를 살펴보겠습니다.

![image](https://github.com/user-attachments/assets/35139b2f-9634-4754-ad94-0715fc55d5e8)


1. Table scan on `e1_0`:  Evaluation 테이블에 대한 Full Table Scan을 진행합니다. 총 `994125`개의 행을 처리하고, 실제 실행 시간은 `0.0489ms`~ `331ms`입니다.
2. Index range scan on `a1_0` using `fk_applicant_to_process`:`fk_applicant_to_process` 인덱스를 사용한 범위 스캔입니다. 이는 효율적으로 인덱스를 사용하여 `process_id`가 `1, 2, 3, 4`인 항목을 스캔한 것입니다. 실제 실행 시간은 `0.0653ms`~ `0.382ms` 사이이며, `150`개의 행이 처리합니다.
3. Hash: 해시 작업이 수행된 것으로, 조인을 효율적으로 수행하기 위해 데이터를 해싱한 단계입니다. 
4. Left hash join: 해시 조인 방식으로 Applicant 테이블과 Evaluation 테이블을 조인합니다. 실제 시간이 `503ms`에서 `685ms` 사이이며, `742`개의 행을 처리했습니다.
5. Aggregate using temporary table: 임시 테이블을 사용한 집계 작업을 수행합니다. SELECT에서 사용된 `count()`   , `avg()` 를 처리합니다. 
6. Table scan on `<temporary>`: 임시 테이블의 결과를 읽어서 반환합니다. 실제 처리된 시간은 `686ms`이며, `150`개의 행을 처리했습니다.

Evaluation 테이블에서 Full Table Scan이 발생하고 있어서 약 10만개의 row를 조회하고 있습니다.

Evaluation에 인덱스를 유도하면, 성능을 높일 수 있습니다. 또한 Evaluation 테이블의 칼럼이 모두 필요한 것이 아니기 때문에 커버링 인덱스를 유도하는 것이 성능에 좋을 것이라 판단했습니다.


<aside>

💡 커버링 인덱스(Covering Index)는 쿼리를 처리할 때, **인덱스만으로** 필요한 모든 데이터를 조회할 수 있는 인덱스를 말합니다. 테이블의 데이터를 따로 읽지 않기 때문에 랜덤 I/O가 발생하지 않고 인덱스만으로 쿼리의 모든 요청을 처리할 수 있어 성능을 크게 향상시킬 수 있습니다.

</aside>


## 커버링 인덱스 유도하기

Evaluation 테이블에 일부 정보(applicant_id, evaluation_id, score)만을 인덱싱하여 인덱스를 생성했습니다.


![image](https://github.com/user-attachments/assets/9852c89c-c990-4600-adb4-8b04a31c9dcd)


실행 계획을 조금 더 자세히 분석 해봅시다.

![image](https://github.com/user-attachments/assets/618e35d7-6f09-4161-9c9a-4126435d6be3)


실행 계획이 이전과 달라진 것을 알 수 있습니다. 집중해서 볼 부분은 type, key, rows, Extra 칼럼입니다.

1. type(ALL → ref): 특정 인덱스를 사용하여 데이터가 조회됩니다.
2. rows(994125 → 4): 994,125개 행에서 4개의 row만 조회할 것으로 예상됩니다. 인덱스를 사용하여 필요한 행만 조회함으로써 성능이 크게 개선되었습니다.
3. Extra(Using where; Using join buffer → Using index): 추가적인 조건 없이 인덱스 자체로 데이터를 찾아냅니다.


![image](https://github.com/user-attachments/assets/07122181-572c-42b9-8e0f-8ad65e09c663)

`EXPLAIN ANALYZE`의 결과에서 실제로 커버링 인덱스가 유도됨을 알 수 있습니다.

---

# 쿼리 성능 개선 결과

최종적으로 평균 API 실행 시간이 900ms에서 101ms로 최적화되었습니다!


![image](https://github.com/user-attachments/assets/7e98c668-39f1-4a47-9c3e-ef31d922a2a7)
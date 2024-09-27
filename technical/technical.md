# API 속도 최적화하기

현재 팀 크루루는 서비스 성장을 가정하고, 데이터베이스에 대량의 데이터를 삽입하여 성능을 개선하는 작업을 진행하고 있습니다.

# 선행 작업

성능 개선을 위해 다음과 같은 대량의 더미 데이터를 테이블에 삽입하였습니다.

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

---

# API 실행 시간 관측

API의 실행 시간을 Grafana 모니터링 툴로 관측해보겠습니다.

![image.png](https://blog.cruru.kr/image_9336158642033857773.png)

![image.png](https://blog.cruru.kr/image_11954446710404310860.png)

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

![스크린샷 2024-09-24 오후 3.14.52.png](https://blog.cruru.kr/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2024-09-24_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_3.14.52_16442101277174382411.png)

### 인덱스 1. (process_id, applicant_id)

`(process_id, applicant_id)` 인덱스를 적용한 후 실행 계획을 살펴봅시다.

![image.png](https://blog.cruru.kr/image_4014018625485784643.png)

주목해야할 칼럼은 possible_keys입니다. possible_keys는 옵티마이저가 최적의 실행 계획을 수립하기 위해 고려한 여러 접근 방법 중에서 사용할 수 있는 인덱스들의 목록입니다 possible_keys가 NULL이기 때문에 옵티마이저가 어떠한 인덱스도 고려하지 않았다는 것을 알 수 있습니다.

따라서 해당 쿼리는 테이블을 처음부터 끝까지 읽는 Full Table Scan으로 실행됩니다.

### 인덱스 2. (applicant_id, process_id)

이번에는 `(applicant_id, process_id)` 인덱스를 적용하여 실행 계획을 살펴보겠습니다.

![image.png](https://blog.cruru.kr/image_3424114008181725518.png)

이전 실행 계획과는 다르게 type, key가 변경되었습니다. type이 ref로 변경되어 인덱스를 사용했으며, 여러 인덱스 중에서 저희가 생성한 `(applicant_id, process_id)` 인덱스를 사용하였음을 알 수 있습니다. 

![image.png](https://blog.cruru.kr/image_8642672252369110754.png)

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

![image.png](https://blog.cruru.kr/image_1154319178054865090.png)

![스크린샷 2024-09-25 오후 4.37.34.png](https://blog.cruru.kr/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2024-09-25_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_4.37.34_1251149608918002654.png)

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

![image.png](https://blog.cruru.kr/image_15585335535790558913.png)

type이 ALL이기 때문에 Full Table Scan으로 실행되고, 어떠한 인덱스도 적용되지 않았습니다. 또한 rows 칼럼에서 994,125행을 스캔해야 한다고 예측하고 있습니다.

이어서 `EXPLAIN ANALYZE`를  실행한 결과를 살펴보겠습니다.

![image.png](https://blog.cruru.kr/image_17050981368580204372.png)

1. **Table scan on `e1_0`**:  Evaluation 테이블에 대한 Full Table Scan을 진행합니다. 총 `994125`개의 행을 처리하고, 실제 실행 시간은 `0.0489ms`~ `331ms`입니다.
2. **Index range scan on `a1_0` using `fk_applicant_to_process`**:`fk_applicant_to_process` 인덱스를 사용한 범위 스캔입니다. 이는 효율적으로 인덱스를 사용하여 `process_id`가 `1, 2, 3, 4`인 항목을 스캔한 것입니다. 실제 실행 시간은 `0.0653ms`~ `0.382ms` 사이이며, `150`개의 행이 처리합니다.
3. **Hash**: 해시 작업이 수행된 것으로, 조인을 효율적으로 수행하기 위해 데이터를 해싱한 단계입니다. 
4. **Left hash join:** 해시 조인을 통해 Applicant 테이블과 Evaluation 테이블을 조인합니다. 실제 시간이 `503ms`에서 `685ms` 사이이며, `742`개의 행을 처리했습니다.
5. **Aggregate using temporary table**: 임시 테이블을 사용한 집계 작업을 수행합니다. SELECT에서 사용된 `count()`   , `avg()` 를 처리합니다. 
6. **Table scan on `<temporary>`**: 임시 테이블의 결과를 읽어서 반환합니다. 실제 처리된 시간은 `686ms`이며, `150`개의 행을 처리했습니다.

Evaluation 테이블에서 Full Table Scan이 발생하고 있어서 약 10만개의 row를 조회하고 있습니다.

Evaluation에 인덱스를 유도하면, 성능을 높일 수 있습니다. 또한 Evaluation 테이블의 칼럼이 모두 필요한 것이 아니기 때문에 커버링 인덱스를 유도하는 것이 성능에 좋을 것이라 판단했습니다.


<aside>

💡 커버링 인덱스(Covering Index)는 쿼리를 처리할 때, **인덱스만으로** 필요한 모든 데이터를 조회할 수 있는 인덱스를 말합니다. 테이블의 데이터를 따로 읽지 않기 때문에 랜덤 I/O가 발생하지 않고 인덱스만으로 쿼리의 모든 요청을 처리할 수 있어 성능을 크게 향상시킬 수 있습니다.

</aside>


## 커버링 인덱스 유도하기

Evaluation 테이블에 일부 정보(applicant_id, evaluation_id, score)만을 인덱싱하여 인덱스를 생성했습니다.

![image.png](https://blog.cruru.kr/6122d278-4a9c-41f1-8a99-4b8f69b2d05d_16613944332768663439.png)

실행 계획을 통해 조금 더 분석을 해봅시다.

![image.png](https://blog.cruru.kr/image_3014179940655528394.png)

실행 계획이 이전과 달라진 것을 알 수 있습니다. 집중해서 볼 부분은 type, key, rows, Extra 칼럼입니다.

1. type(ALL → ref): 특정 인덱스를 통해서만 데이터를 조회됩니다.
2. rows(994125 → 4): 994,125개 행에서 4개의 row만 조회할 것으로 예상됩니다. 인덱스를 사용하여 필요한 행만 조회함으로써 성능이 크게 개선되었습니다.
3. Extra(Using where; Using join buffer → Using index): 추가적인 조건 없이 인덱스 자체로 데이터를 찾아냅니다.

![image.png](https://blog.cruru.kr/image_4769791571049791832.png)

**`EXPLAIN ANALYZE`**의 결과에서 실제로 커버링 인덱스가 유도됨을 알 수 있습니다.

---

# 쿼리 성능 개선 결과

최종적으로 평균 API 실행 시간이 900ms에서 101ms로 최적화되었습니다!

![image.png](https://blog.cruru.kr/image_17189423053705570230.png)
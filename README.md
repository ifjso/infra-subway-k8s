<p align="center">
    <img width="200px;" src="https://raw.githubusercontent.com/woowacourse/atdd-subway-admin-frontend/master/images/main_logo.png"/>
</p>
<p align="center">
  <img alt="npm" src="https://img.shields.io/badge/npm-%3E%3D%205.5.0-blue">
  <img alt="node" src="https://img.shields.io/badge/node-%3E%3D%209.3.0-blue">
  <a href="https://edu.nextstep.camp/c/R89PYi5H" alt="nextstep atdd">
    <img alt="Website" src="https://img.shields.io/website?url=https%3A%2F%2Fedu.nextstep.camp%2Fc%2FR89PYi5H">
  </a>
  <img alt="GitHub" src="https://img.shields.io/github/license/next-step/atdd-subway-service">
</p>

<br>

# 인프라공방 샘플 서비스 - 지하철 노선도

<br>

## 🚀 Getting Started

### Install
#### npm 설치
```
cd frontend
npm install
```
> `frontend` 디렉토리에서 수행해야 합니다.

### Usage
#### webpack server 구동
```
npm run dev
```
#### application 구동
```
./gradlew clean build
```
<br>

## 미션

* 미션 진행 후에 아래 질문의 답을 작성하여 PR을 보내주세요.

### 1단계 - 쿼리 최적화

1. 인덱스 설정을 추가하지 않고 아래 요구사항에 대해 200ms 이하(M1의 경우 2s)로 반환하도록 쿼리를 작성하세요.

- 활동중인(Active) 부서의 현재 부서관리자 중 연봉 상위 5위안에 드는 사람들이 최근에 각 지역별로 언제 퇴실했는지 조회해보세요. (사원번호, 이름, 연봉, 직급명, 지역, 입출입구분, 입출입시간)

```sql
select
    a.employee_id as '사원번호',
    e.last_name as '이름',
    a.annual_income as '연봉',
    p.position_name as '직급명',
    max(r.time) as '입출입시간',
    r.region as '지역',
    r.record_symbol as '입출입분'
from (
    select m.employee_id, m.department_id, s.annual_income
    from manager m
      join department d on d.id = m.department_id
      join salary s on s.id = m.employee_id
    where
      m.end_date >= now()
      and s.end_date >= now()
      and d.note = 'active'
    order by s.annual_income desc
    limit 5
) a
  join employee e on e.id = a.employee_id
  join position p on p.id = a.employee_id
  join record r on r.employee_id = a.employee_id
where
  p.end_date >= now()
  and r.record_symbol = 'O'
group by a.employee_id, e.last_name, a.annual_income, p.position_name, r.region, r.record_symbol
```

---

### 2단계 - 인덱스 설계

1. 인덱스 적용해보기 실습을 진행해본 과정을 공유해주세요
   1. Coding as a Hobby 와 같은 결과를 반환하세요.
      1. 첫 실행 시 2.3s 정도 나왔으며 full table scan 으로 실행됨
      2. programmer.hobby 를 인덱스로 추가 후 full index scan 으로 실행되고 0.232s 걸림
      ```sql
      select
        concat(round(count(case when hobby = 'Yes' then 1 end) / count(*) * 100, 1), '%') as Yes,
        concat(round(count(case when hobby = 'No' then 1 end) / count(*) * 100, 1), '%') as No
      from programmer;
      ```
   2. 프로그래머별로 해당하는 병원 이름을 반환하세요.
      1. 첫 실행 시 4.8s 정도 나오고 전부 풀 스캔을 확인.
      2. 조인 시 사용하는 컬럼을 기준으로 programmer.id, hospital.id를 각각 pk로 만들고 covid.programmer_id를 유니크 인덱스로 설정 후 다시 실행 계획을 보니 설정한 키를 잘 타고 있음.
      3. 다만 programmer 테이블이 full index scan 이긴 하지만 별도의 조건이 없고 걸린시간도 0.029s 라서 준수하다고 판단함.
   ```sql
    select p.id, c.id, h.name
    from programmer p
    join covid c on c.programmer_id = p.id
    join hospital h on h.id = c.hospital_id
   ```
   3. 프로그래밍이 취미인 학생 혹은 주니어(0-2년)들이 다닌 병원 이름을 반환하고 user.id 기준으로 정렬하세요.
      1. 첫 실행 시 0.027s 가 나와서 준수한 결과라고 판단함.
      2. where 조건에 있는 programmer.hobby와 programmer.years_coding 를 인덱스로 고려해 보았지만 카디널리티가 너무 낮다고 생각했고, 실제 적용했을 때도 효과가 없었음
   ```sql
    select c.id, h.name, p.hobby, p.dev_type, p.years_coding
    from programmer p
    join covid c on c.programmer_id = p.id
    join hospital h on h.id = c.hospital_id
    where
        p.hobby = 'Yes'
        or p.years_coding = '0-2 years'
    order by p.id
   ```
   4. 서울대병원에 다닌 20대 India 환자들을 병원에 머문 기간별로 집계하세요.
      1. 첫 실행 시 member, programmer, covid 테이블 모두 full table scan 으로 나옴.
      2. member.id 를 pk로 만들고 covid.member_id, programmer.member_id 를 유니크 인덱스로 만들어서 다시 실행 계획을 보니 member 외에 모두 range scan으로 바뀜.
      3. member.age 를 index로 설정하였으나 효과가 없어서 삭제.
      4. 기존 programmer.member_id 유니크 인덱스를 삭제하고 programmer.member_id 와 programmer.country를 결합하여 유니크 인덱스로 설정하니 인덱스를 잘 탔지만 programmer 에서 index full scan 발생.
      5. 조건 중에 병원이름이 있다는 것을 확인하고 hospital.name을 유니크 인덱스로 설정하니 programmer 가 index range scan으로 바뀌고 full scan이 없어짐.
      6. 조인 시 사용하는 covid.hospital_id를 인덱스로 추가함.
      7. group by로 인한 filesort를 없애기 위해 order by null 적용 후 실행해보니 0.218s 나옴
   ```sql
    select c.stay, count(c.stay)
    from member m
    join programmer p on m.id = p.member_id
    join covid c on m.id = c.member_id
    join hospital h on c.hospital_id = h.id
    where
        m.age between 20 and 29
        and p.country = 'India'
        and h.name = '서울대병원'
    group by c.stay
    order by null
   ```
   5. 서울대병원에 다닌 30대 환자들을 운동 횟수별로 집계하세요.
      1. 첫 실행 시 전부 range, constant scan 이었고 실행해 보니 0.259s 걸림
   ```sql
    select p.exercise, count(p.exercise)
    from member m
    join programmer p on m.id = p.member_id
    join covid c on m.id = c.member_id
    join hospital h on c.hospital_id = h.id
    where
        m.age between 30 and 39
        and h.name = '서울대병원'
    group by p.exercise
    order by null
   ```
---



### 3단계 - 쿠버네티스로 구성하기
1. 클러스터를 어떻게 구성했는지 알려주세요~ (마스터 노드 : n 대, 워커 노드 n대)
- 마스터 1대 , 워커 3대
2. 스트레스 테스트 결과를 공유해주세요 (기존에 container 한대 운영시 한계점도 같이 공유해주세요)
![stress.png](perf%2Fstress.png)
- container를 한대 운영하면 container 장애 시 다시 새로운 container가 구성될 때까지
  아무것도 할 수 없습니다. 여러 container를 운영하면 한 container에서 장애가 발생하여
  다시 새로운 container를 구성하는 동안에도 다른 container를 통해 정상 운영할 수 있습니다.
3. 현재 워커노드에서 몇대의 컨테이너를 운영중인지 공유해주세요
- 3대
---

### [추가] DB 미션

1. 페이징 쿼리를 적용한 API endpoint를 알려주세요

---


### [추가] 클러스터 운영하기
1. kibana 링크를 알려주세요

2. grafana 링크를 알려주세요

3. 지하철 노선도는 어느정도로 requests를 설정하는게 적절한가요?

4. t3.large로 구성할 경우 Node의 LimitRange, ResourceQuota는 어느정도로 설정하는게 적절한가요?

5. 부하테스트를 고려해볼 때 Pod은 몇대정도로 구성해두는게 좋다고 생각하나요?

6. Spinaker 링크를 알려주세요.

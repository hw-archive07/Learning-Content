# 미니 DBMS 학습 노트

Week11-12 AI 활용 · Python으로 구현한 SQL 엔진과 퍼즐 기록 저장

**DB**는 저장된 데이터, **DBMS**는 저장·조회·수정·복구를 관리하는 프로그램, **SQL**은 관계형 DBMS에 요청을 표현하는 언어다. 여러 프로그램이 같은 데이터를 다를 때 DBMS가 검색과 동시 수정, 저장 실패를 공통으로 처리한다.

## 1. SQL을 해석하고 파일에 저장하기

SQL 문자열을 토큰으로 나눈 뒤 AST로 바꾼다. 실행 엔진은 테이블, 컬럼, 값을 검증하고 단일 테이블의 INSERT, SELECT, UPDATE, DELETE를 처리한다.

```sql
INSERT INTO products (name, price) VALUES ('키보드', 30000);
SELECT name, price FROM products WHERE id = 1;
UPDATE products SET price = 35000 WHERE id = 1;
DELETE FROM products WHERE id = 1;
```

빈 DB에서 첫 INSERT의 ID는 1이다. DELETE는 행을 제거하고 변경 로그를 남긴다. 쓰기 요청은 체크섬이 있는 로그를 `data/transactions.jsonl`에 추가하고 `fsync`가 성공한 뒤 메모리에 반영한다. 부분 쓰기나 동기화가 실패하면 파일을 이전 크기로 되돌리고, 복구에 실패하면 추가 쓰기를 중단한다.

## 2. 메모리 B+ 트리로 ID 조회하기

ID는 직접 지정하거나 자동 발급받는다. B+ 트리의 내부 노드는 검색 방향을 정하고, 서로 연결된 리프는 ID와 로그 위치를 보관한다.

- **ID 조회**는 트리에서 파일 위치를 찾은 뒤 저장 파일에서 해당 행을 읽는다.
- **닉네임처럼 ID가 아닌 조건**은 리프를 순회한다. 삽입·삭제 시 노드 분할, 재분배, 병합으로 트리 균형을 유지한다.
- **서버 재시작** 시 로그를 순서대로 읽어 현재 행, 다음 ID, 테이블 버전과 트리를 복원한다.

이전 Windows 로컬 측정에서 HTTP로 1,000행씩 1,000회 저장했다. 100만 행 INSERT는 **21.69초**, 임의 ID 100회 조회 평균은 **10.72ms**, 재시작 복구는 **3.67초**였다. `npm run benchmark:dbms`로 재현할 수 있다. 이 수치는 1,000행씩 묶어 저장한 기존 측정값이다.

## 3. 외부 API와 동시성 제어

`npm run dev`로 앱 서버를 실행한다. 외부 클라이언트는 `POST /api/sql`에 SQL과 `params`를 보낸다. `GET /api/db/stats`에서 테이블 버전, 트리 높이, 조회 횟수 등을 확인한다.

- **ThreadPoolExecutor**가 기본 4개 작업 스레드를 재사용한다. 대기 요청은 최대 128개다.
- **파일 소유권**은 `owner.lock`으로 관리한다. 같은 데이터 폴더를 두 DBMS 프로세스가 동시에 열 수 없다.
- **커밋 잠금**으로 파일 쓰기를 직렬화한다. 변경 계획에 사용한 테이블 버전이 달라지면 최신 상태에서 다시 계획한다. 읽은 값을 바탕으로 수정하는 클라이언트는 `expectedVersion`을 보내고, 충돌로 409가 오면 다시 읽고 계산한다.
- 지원 범위는 단일 테이블 CRUD, 동등 조건 하나, 정렬, LIMIT이다. SELECT는 최대 1,000행, INSERT는 요청당 최대 5,000행이며 HTTP 본문 크기 제한도 적용된다.

## 4. 퍼즐 성공 기록과 이름별 랭킹

```text
공통 도전 발급 → 보드 첫 클릭으로 측정 시작
  → 성공 확인 시 서버가 풀이를 검증하고 시간 확정
  → 닉네임으로 기록 등록 → 이름별 최고 기록 TOP 10 조회
```

처음 등록한 이름은 INSERT한다. 같은 이름은 **이동 횟수가 적을 때**, 또는 **횟수가 같고 시간이 짧을 때** 기존 행을 UPDATE한다. 예를 들어 7회·1초보다 5회·10초가 앞서며, 다음에 5회·8초를 달성하면 갱신한다. 이름 앞뒤 공백을 제거하고 대소문자는 구분한다. 기존 중복 행은 보존하며 조회에서 이름별 최고 기록을 선택한 뒤 TOP 10을 계산한다. 기록 원본은 DBMS에 저장하고 조회 결과는 [미니 Redis](redis-study-notes.md)에 잠시 보관한다.

`npm run test:dbms`로 SQL·B+ 트리·파일 복구·동시 등록을 검증한다. 이름별 기록 갱신, Redis 캐시 무효화, 화면 표시까지 포함한 최근 전체 `npm test`는 74개가 통과했다.

## 참고 자료

- [IBM IMS 역사](https://www.ibm.com/history/information-management-system) · [에드거 코드의 관계형 모델](https://research.ibm.com/publications/a-relational-model-of-data-for-large-shared-data-banks)
- [Python ThreadPoolExecutor](https://docs.python.org/3/library/concurrent.futures.html) · [Python os.fsync](https://docs.python.org/3/library/os.html#os.fsync)
- [PostgreSQL 쿼리 처리](https://www.postgresql.org/docs/17/query-path.html)

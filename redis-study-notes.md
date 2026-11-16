# Redis 학습 노트

Week11-12 AI 활용 · 개념 정리 2026-11-13 · 구현 내용 추가

Redis가 생긴 이유와 캐시 동작을 이해하고, 미니 DBMS를 만든 뒤 Redis를 연결하는 흐름을 정리한다.

## 1. Redis가 생긴 이유와 하는 일

2009년 Salvatore Sanfilippo는 실시간 방문 분석 서비스 LLOOGG를 개발하고 있었다. 페이지를 열 때마다 MySQL에 방문 기록을 쓰는 구조라 방문자가 늘면 쓰기 부하가 커졌다. 최근 방문 기록을 빠르게 가져오는 기능도 필요했다. Redis는 이 작업을 처리하기 위해 시작됐다.

**Redis는 데이터를 주로 RAM에 저장하고 키로 읽거나 변경하는 서버 프로그램**이다. 백엔드가 Redis에 명령을 보내는 연결에서는 백엔드가 클라이언트이고, Redis가 서버다. RAM에서 값을 바로 읽는 이점과 캐시로 원본 DB 조회를 줄이는 이점은 각각 저장 위치와 요청 경로에서 생긴다.

| 용도 | Redis가 하는 일 |
| --- | --- |
| 상품 캐시 | 자주 요청하는 상품 조회 결과를 일정 시간 보관한다. |
| 로그인 세션 | 세션 ID에 연결된 로그인 정보를 보관하고 만료시킨다. |
| 조회수·요청 횟수 | `INCR`로 값을 올리고, 필요하면 TTL로 집계 기간을 정한다. |
| 최근 방문 기록 | `List`의 앞에 기록을 넣고 최신 N개를 가져온다. |
| 게임 랭킹 | `Sorted Set`에 사용자별 점수를 넣고 점수 순으로 읽는다. |

## 2. 키와 값, 명령이 오가는 과정

`SET post:42:views 10`의 키는 `post:42:views`, 저장되는 값은 문자열 `"10"`이다. `GET`은 키의 값을 읽고 `DEL`은 키를 지운다. `INCR`는 정수로 해석할 수 있는 값을 1 올린다.

```redis
SET post:42:views 10
INCR post:42:views
INCR post:42:views
GET post:42:views
```

두 번의 `INCR`는 각각 정수 `11`과 `12`를 돌려준다. 마지막 `GET`의 결과는 문자열 `"12"`다.

**TCP**는 클라이언트와 Redis 사이에서 바이트가 오가는 연결이다. **RESP**는 그 바이트가 명령, 인자, 응답 중 무엇을 뜻하는지 표현하는 형식이다.

```text
백엔드 또는 redis-cli → TCP로 RESP 명령 전송
Redis → 명령과 인자 해석 → RAM의 값 읽기 또는 변경 → RESP로 응답
```

## 3. 캐시 히트·미스와 TTL

상품 페이지를 조회할 때 **백엔드가** Redis와 DBMS에 각각 요청을 보낸다. Redis에 요청한 키의 유효한 값이 있으면 **히트**, 없거나 만료됐으면 **미스**다. 백엔드가 캐시를 확인한 뒤 미스일 때 원본을 읽고 채우는 패턴을 **캐시 어사이드(cache-aside)**라고 한다.

```text
사용자 → 백엔드 → Redis에 GET product:42
  히트 → Redis의 상품 정보로 응답
  미스 → 백엔드가 DBMS에 요청 → DBMS가 원본 상품 조회
       → 백엔드가 Redis에 SET → 사용자에게 응답
```

`SET product:42 "상품 정보" EX 60`은 이 키에 60초의 유효기간을 준다. **TTL**은 만료까지 남은 시간이다. 추가 변경이 없다면 30초 뒤에는 값이 나오고, 70초 뒤에는 값이 없다. 중간에 `GET`해도 만료 시각은 그대로다.

- **만료 후 같은 키에 저장**하면 그 키에 새 값이 들어간다. 만료된 키의 `GET`은 값 없음으로 처리하므로, 백엔드가 먼저 `DEL`할 필요는 없다. 원본 DB의 ID `42`와 Redis 키 `product:42`는 각 저장소에서 따로 관리한다.
- **DB 가격 변경 직후**에는 TTL이 남은 이전 가격이 보일 수 있다. 기본 흐름은 DB 수정을 성공시킨 뒤 Redis에서 `DEL product:42`로 기존 캐시를 비우는 것이다. 다음 조회가 원본에서 새 값을 가져온다.

## 4. 동시 요청과 재시작

- 두 백엔드가 모두 `GET`으로 `10`을 읽은 다음 각각 `SET`으로 `11`을 쓰면 최종 값은 `11`이다. 두 요청 중 하나의 증가가 덮어써진다.
- 두 백엔드가 `INCR`를 한 번씩 보내면 최종 값은 `12`다. Redis가 각 증가를 **한 명령으로 원자적으로 처리**해 `10 → 11 → 12`가 된다.

| 영속성 방식 | 재시작 후 동작 |
| --- | --- |
| 저장 끔 | RAM에만 있던 값은 사라진다. 상품 캐시는 다음 미스에 DB에서 다시 채울 수 있다. |
| RDB | 특정 시점의 데이터 스냅샷을 저장하고 시작할 때 읽는다. |
| AOF | 데이터에 영향을 주는 쓰기 명령을 기록하고 시작할 때 적용해 상태를 복원한다. |

RDB와 AOF를 함께 쓸 수 있다. 저장 대상의 크기는 메모리 용량을 고려하고, 장애 시 어느 시점까지 복원되는지는 디스크 저장 설정에 따라 달라진다.

## 5. 미니 Redis 구현과 퍼즐 랭킹

JavaScript와 Node.js도 검토했지만 최종 구현은 **Python 표준 라이브러리**를 사용했다. 키의 해시값으로 버킷을 고른다. 같은 버킷에 들어온 키는 목록에 함께 보관하고 실제 키를 비교한다(체이닝). 같은 키를 다시 저장하면 값을 덮어쓴다.

- 잠금으로 저장·조회·삭제를 보호한다. 만료된 값은 조회 시 미스로 처리하고 주기적으로 정리한다.
- 외부 프로그램은 `PUT /api/kv/{key}`, `GET /api/kv/{key}`, `DELETE /api/kv/{key}`를 호출한다. PUT은 JSON 값과 선택적인 `ttlSeconds`를 받는다. 이 구현은 HTTP JSON 방식이며, RESP, `redis-cli`, RDB/AOF는 지원하지 않는다.
- 퍼즐 기록 원본은 [미니 DBMS 학습 노트](dbms-study-notes.md)의 파일 저장소에 남는다. 미니 Redis는 이름별 최고 기록 TOP 10 응답을 RAM에 보관한다.

```text
랭킹 조회 → 앱 서버가 Redis 확인
  히트 → 저장된 TOP 10 반환
  미스 → 미니 DBMS 조회 → Redis에 30초 동안 저장 → 반환
```

새 이름이 등록되거나 같은 이름의 최고 기록이 갱신되면 DBMS 저장이 끝난 뒤 기존 랭킹 캐시를 지운다. 캐시 키에 DB 테이블 버전이 들어가므로 삭제에 실패해도 다음 조회는 새 버전의 키를 사용한다. 기존 최고 기록을 유지하는 경우 캐시도 유지한다. Redis가 중단되면 앱은 DBMS에서 랭킹을 조회한다.

## 6. 동작 확인과 성능

1. 터미널 두 개에서 `npm run redis`와 `npm run dev`를 각각 실행한다.
2. [퍼즐 페이지](http://127.0.0.1:4173)에서 **랭킹 새로고침**을 눌러 캐시를 채운다. 30초 안에 다시 누르면 [캐시 통계](http://127.0.0.1:4175/api/cache/stats)의 `hits`가 늘어난다.
3. 캐시를 채운 시점부터 30초가 지난 뒤 다시 조회하면 `misses`가 늘고 DBMS에서 결과를 가져온다.

`npm run test:redis`는 해시 충돌, TTL, 동시 접근, 최고 기록 갱신 시 캐시 무효화와 장애 시 DBMS 대체 조회를 확인한다.

이전 로컬 벤치마크에서는 1,000건을 넣고 비교했다. 랭킹 100회 조회에서 **DB 조회는 100회 → 0회**로 줄었고, 평균 응답 시간은 **8.611ms → 26.217ms**로 늘었다. 이 환경에서는 별도 HTTP 호출과 JSON 처리 비용이 있었으며, 수치는 이번에 다시 측정하지 않았다. 재현 명령은 `npm run benchmark:redis`다.

## 참고 자료

- [Redis 창시자 인터뷰](https://www.eu-startups.com/2011/01/an-interview-with-salvatore-sanfilippo-creator-of-redis-working-out-of-sicily/) · [Redis 소개](https://redis.io/tutorials/what-is-redis/)
- [캐시 어사이드](https://redis.io/docs/latest/develop/use-cases/cache-aside/) · [INCR](https://redis.io/docs/latest/commands/incr/) · [RESP](https://redis.io/docs/latest/develop/reference/protocol-spec/) · [RDB와 AOF](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)

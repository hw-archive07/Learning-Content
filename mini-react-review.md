# 미니 React 동작 복습

Week11-12 · AI 활용 학습 퀴즈 정리

## 1. 실제 영역과 테스트 영역

테스트 영역에서 제목을 `이번 주 학습`에서 `React 실험`으로 고치면 미리보기는 즉시 바뀐다. 실제 영역은 **Patch 적용**을 누를 때 바뀐다.

## 2. Diff와 Patch

**Diff**는 이전 Virtual DOM과 새 Virtual DOM을 비교해 변경의 위치와 종류를 찾는다. **Patch**는 그 변경을 실제 DOM에 적용한다. `<h3>`의 글자만 달라지면 요소를 재사용하고 그 안의 텍스트만 바꾼다.

## 3. State History

`A → B → C`에서 뒤로가기를 누르면 기록의 현재 위치만 `B`로 이동한다. 실제 영역과 미리보기, HTML 편집기가 함께 복원된다. 여기서 `D`를 적용하면 기록은 `A → B → D`가 되고, 앞으로가기로 `C`에 갈 수 없다.

## 4. 루트 상태와 Props

댓글 초안은 루트 컴포넌트 **`Board`**가 `useState`로 기억한다. 작성창을 닫으면 기존 `textarea`는 DOM에서 제거된다. 다시 열면 `CommentComposer`가 `draft`를 props로 받아 새 `textarea`의 `value`에 넣는다. 초안이 빈 문자열이면 `댓글을 입력하세요.`라는 안내 문구가 보인다.

## 5. useMemo와 useEffect

- **`useMemo`**: 의존성인 `comments`가 그대로면 댓글 수와 글자 수 계산 결과를 재사용한다. 좋아요만 누르면 댓글 요약을 다시 계산하지 않는다.
- **`useEffect`**: DOM에 변경을 반영한 뒤 실행된다. 현재 구현에서는 좋아요 수를 탭 제목에 반영하고, 좋아요와 댓글 상태를 브라우저에 저장한다.

## 6. 새로고침과 서버 재시작

`localStorage`에 **서버 실행 ID와 값**을 함께 저장한다. 같은 브라우저에서 새로고침하거나 탭을 다시 열면, 같은 ID로 저장한 좋아요·댓글·초안·Patch 기록이 복원된다. 서버를 재시작하면 새 ID를 받아 좋아요 0, 빈 댓글, 처음 샘플로 시작한다.

## 종합 · 댓글이 표시되는 두 경로

- **등록:** `Board`가 상태에 댓글을 추가 → `CommentList`가 props로 받아 표시 → `useEffect`가 저장.
- **새로고침:** `Board`가 `storage.read("comments")`로 같은 서버 ID의 값을 초기 상태에 복원 → `CommentList`가 props로 받아 표시.

관련 노트 · `react-study-notes.md`

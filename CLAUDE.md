# 국성티처

강사별 학생/시간표/출석 관리 웹앱. 단일 파일 [index.html](index.html)로 되어 있고, 데이터는 전부 Supabase에 저장됨(로컬 파일이나 별도 백엔드 없음).

## Supabase 접속 정보

- URL: `https://haaiwbjdlqwrfrvsurxi.supabase.co`
- anon key: index.html의 `SUPABASE_ANON_KEY` (line 503 부근)에 있음 — 이걸로 REST API 직접 조회/수정 가능
- 테이블: `app_data` (컬럼: `id`, `data`, `updated_at`). row 하나에 큰 JSON 덩어리 하나씩 통째로 들어있음:
  - `id = 'teacherDataStore'` → 강사별 학생/시간표
  - `id = 'attendanceRecords'` → 주차별 출석 기록
  - `id = 'weeklyOverrides'` → 주차별 임시 시간표 이동
  - `id = 'slotFreeMemos'`, `'globalReservations'`, `'smsTemplates'`, `'classroomAssignments'` 등도 같은 방식

**버전 이력 없음**: 이 REST API로는 과거 스냅샷을 조회할 수 없다. 뭔가 잘못 덮어쓰면 사용자에게 원래 값(전화번호 등)을 직접 물어봐야 함.

## 데이터 조회/수정 절차 (curl)

```bash
APIKEY="<SUPABASE_ANON_KEY>"

# 조회
curl -s "https://haaiwbjdlqwrfrvsurxi.supabase.co/rest/v1/app_data?select=id,data&id=eq.teacherDataStore" \
  -H "apikey: $APIKEY" -H "Authorization: Bearer $APIKEY" -o fresh.json

# (python 등으로 fresh.json 수정 → updated.json)

# 반영 (PATCH, upsert 아님 — 이미 row가 있으므로 PATCH 사용)
curl -s -X PATCH "https://haaiwbjdlqwrfrvsurxi.supabase.co/rest/v1/app_data?id=eq.teacherDataStore" \
  -H "apikey: $APIKEY" -H "Authorization: Bearer $APIKEY" \
  -H "Content-Type: application/json" -H "Prefer: return=minimal" \
  --data-binary @payload.json   # payload.json = {"id": "teacherDataStore", "data": {...}}
```

**반드시 지킬 것**: 수정 직전에 항상 최신본을 다시 fetch해서 그 위에 수정을 적용할 것 (다른 사람이 앱에서 그 사이 저장했을 수 있음). 수정 후에는 다시 조회해서 반영 결과를 검증할 것.

## teacherDataStore 구조

`data[강사이름] = { slots: {...}, students: [...] }`

- `slots.{friday|saturday|sunday}` = `[{ id: "slot-<요일>-<타임스탬프>", title: "학교 (시간)" }]`. title에 "청람중3 / 가현중3"처럼 여러 학교가 한 슬롯에 같이 있는 경우가 있음 — 그 슬롯을 공유하는 서로 다른 학교 학생들이 존재한다는 뜻.
- `students[]` 각 원소:
  - `name`, `phone`, `school`, `parentPhone`
  - `slotId`(단일, 옛날 형식) 또는 `slotIds`(배열, 최신 형식) — 학생이 배정된 시간표 슬롯
  - `firstWeekKey`, `registeredAt` — **이번 주에 신규 등록된 학생에만 있음**. 원래 재원생이던 학생인데 이 필드가 잘못 생기면 "신규 등록"으로 잘못 표시되니 주의. 재원생을 수정할 때 이 필드를 새로 넣거나 건드리면 안 됨.
  - `status: 'deleted'`, `deletedAt`, `deletedWeekKey`, `excludedWeeks` — 퇴원/특정 주차 제외 처리된 학생

## 동명이인(같은 강사 내 이름 중복) 규칙 — 중요

같은 강사 밑에 이름이 완전히 같은 학생이 둘 이상 있으면 문제가 생길 수 있는 지점들:

1. **출석 체크**: 기록 키가 `이름__슬롯ID` (`getRecordKey`, index.html:1862 부근). **같은 슬롯**을 쓰는 동명이인은 이름이 같으면 키도 같아져서 한 명 체크하면 같이 체크됨. → 이 경우엔 **반드시 이름 자체를 다르게** 해야 함(알파벳 A/B/C 등을 이름 뒤에 붙이는 방식 사용 중).
2. **퇴원 처리 / 완전삭제** (`softDeleteStudent`, `removeStudentPermanently`, `purgeStudentEverywhere`): `overrideKey`(이름+학교, index.html:1221 부근)로 대상을 찾도록 이미 수정해놨음(2026-09-19). 그래서 **학교가 다른 동명이인은 이름이 같아도 이제 안전**함 — 굳이 알파벳 안 붙여도 서로 안 건드림.
3. 그래도 화면에서 헷갈리니, 사용자는 학교가 다른 동명이인도 알파벳(A=먼저 발견된 쪽/원래 있던 쪽, B, C...)을 붙여서 표기하는 걸 선호함. 새로 동명이인을 발견하면 정리 방식을 물어보고 붙여줄 것.

**동명이인 스캔 방법**: `teacherDataStore`의 강사별 `students`에서 `status !== 'deleted'`인 것만 이름으로 그룹핑 → 2개 이상이면 동명이인. 슬롯이 겹치는지(같은 slotId/slotIds) 확인해서 "충돌 위험(같은 슬롯)" vs "안전(다른 슬롯)"으로 구분해서 보고할 것.

## 학생 데이터 잘못 수정됐을 때 대응 순서

1. teacherDataStore를 fetch해서 해당 강사의 students 배열에서 무슨 일이 있었는지 진단 (중복 생성됐는지, 필드가 사라졌는지, 다른 학생 걸 덮어썼는지).
2. 애매하면 추측해서 고치지 말고 사용자에게 구체적으로 물어볼 것 (특히 전화번호처럼 서버에 남아있지 않은 정보는 사용자가 직접 알려줘야만 복구 가능).
3. 수정은 항상 fresh fetch → 수정 → PATCH → 재조회 검증 순서로.
4. `index.html` 코드 자체를 고친 경우엔 git commit + push까지 사용자가 요청하면 진행.

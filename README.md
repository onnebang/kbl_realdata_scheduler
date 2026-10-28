# KBL realdata scheduler

`onnebang/kbl_realdata`(비공개)의 운영 수집 작업만 실행하는 공개 저장소다. 예약 workflow의 관리
경계를 애플리케이션 소스와 분리하기 위해 존재하며, 애플리케이션 코드는 이 저장소에 복사하지
않는다. 이 저장소가 공개라 Actions 실행 시간이 무료이고, 비공개 소스는 읽기 전용 deploy key로만
읽는다. Supabase 쓰기 자격증명 값은 커밋하지 않고 암호화된 GitHub Actions Secrets에만 둔다.

## Workflows

| workflow | schedule (KST) | command |
|---|---:|---|
| `KBL Live Collector` | 매일 13:00 | `live_collector.py --max-runtime 20700 --stall-timeout 5400` |
| `KBL Daily Batch` | 매일 06:00 | `daily_batch.py` |
| `Courtside Monitor` | 30분마다(매시 7분·37분) | 운영 사이트 `/api/health`(DB 조회·키-프로젝트 일치)와 주요 페이지 9곳 점검 |

앞의 두 수집 workflow는 비공개 `kbl_realdata` 저장소의 `SOURCE_REF` 커밋을 읽기 전용 deploy key
(`SOURCE_DEPLOY_KEY`)로 고정 체크아웃한다. 수집기 코드를 바꾼 뒤에는 변경을 검토하고 두 YAML의
`SOURCE_REF`를 같은 새 커밋으로 올린다. `main` 같은 이동 브랜치를 직접 실행하지 않는다.

`Courtside Monitor`는 소스를 체크아웃하지 않고 운영 주소(`https://courtside.ai.kr`)만 `curl`로 본다.
비공개가 된 `kbl_realdata`의 GitHub Actions(`courtside-ci`·`courtside-smoke`)는 계정 한도 문제로 돌지 않아
2026-10-28 비활성화했고, 배포 후 스모크가 하던 점검을 이 예약 모니터가 대신한다(배포 커밋과의 일치 검사만
빠졌다). 하나라도 어긋나면 workflow가 실패하고 GitHub 알림 메일이 간다. 이 저장소는 공개라 로그를 누구나
볼 수 있으므로 토큰·응답 본문은 출력하지 않는다.

## Required Actions secrets

- `DATABASE_URL`: Supabase 직결 쓰기 URL. RLS를 우회하므로 로그에 출력하거나 공개 저장소에
  넣지 않는다.
- `REVALIDATE_TOKEN`: 적재 후 Courtside 웹 캐시 무효화 토큰.
- `SOURCE_DEPLOY_KEY`: 비공개 `kbl_realdata`를 읽는 읽기 전용 deploy key의 개인키. 공개키는
  `kbl_realdata` Settings → Deploy keys에 쓰기 허용 없이 등록돼 있다. 그 한 저장소 전용이라 이
  키로 다른 저장소는 읽을 수 없다.
- `SMOKE_BYPASS_TOKEN`: `Courtside Monitor`가 Vercel Bot Protection 챌린지를 건너뛰려고 보내는
  `x-courtside-smoke` 헤더의 값. Vercel 방화벽의 "헤더 `x-courtside-smoke`가 이 값과 같으면 Bypass"
  규칙과 같은 값이어야 한다(바꿀 때는 두 곳을 함께).

DB·재검증 secrets는 각 Python 실행 step에만 전달한다. checkout step에는 읽기 전용 deploy key만
전달하고(`persist-credentials: false`로 체크아웃 뒤 제거), Python 설치·의존성 설치 step에는
아무것도 전달하지 않는다. 모니터 점검 step에는 우회 토큰만 전달한다. GitHub Actions 자체도 검증한
커밋 SHA로 고정한다.

**주의 - 60일 규칙:** 공개 저장소는 60일 동안 저장소 활동(커밋 등)이 없으면 GitHub가 예약 workflow를
자동으로 끈다(수집·모니터가 조용히 멈춘다). 시즌 중에는 `SOURCE_REF` 갱신 같은 커밋이 생기지만 비시즌에는
Actions 탭에 비활성화 안내가 없는지 확인하고, 필요하면 커밋을 하나 남기고 workflow를 다시 켠다.

## Cutover / recovery

1. 이 저장소가 `Public`이고 `kbl_realdata`가 `Private`인지 확인한다.
2. 네 Actions secrets(`DATABASE_URL`, `REVALIDATE_TOKEN`, `SOURCE_DEPLOY_KEY`, `SMOKE_BYPASS_TOKEN`)를
   등록하고, `kbl_realdata`에 읽기 전용 deploy key(공개키)가 있는지, Vercel 방화벽에 `x-courtside-smoke`
   Bypass 규칙이 있는지 확인한다.
3. `KBL Daily Batch`를 수동 실행해 성공을 확인한다.
4. 경기 없는 시간에 `KBL Live Collector`를 `once=true`로 수동 실행해 한 주기만 확인한다.
5. `kbl_realdata` 저장소에 예약 workflow가 없는지 확인한다(CI·스모크 workflow는 2026-10-28
   비활성화했다).
6. `Courtside Monitor`를 수동 실행해 성공을 확인한다.
7. 같은 시간대에 로컬 Windows `KBL_LiveCollector`/`KBL_DailyBatch`와 클라우드 잡을 함께
   켜지 않는다. 장애 시에는 클라우드 workflow를 끄고 로컬 태스크를 복구 경로로 쓴다.

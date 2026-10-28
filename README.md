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

두 workflow는 비공개 `kbl_realdata` 저장소의 `SOURCE_REF` 커밋을 읽기 전용 deploy key
(`SOURCE_DEPLOY_KEY`)로 고정 체크아웃한다. 수집기 코드를 바꾼 뒤에는 변경을 검토하고 두 YAML의
`SOURCE_REF`를 같은 새 커밋으로 올린다. `main` 같은 이동 브랜치를 직접 실행하지 않는다.

## Required Actions secrets

- `DATABASE_URL`: Supabase 직결 쓰기 URL. RLS를 우회하므로 로그에 출력하거나 공개 저장소에
  넣지 않는다.
- `REVALIDATE_TOKEN`: 적재 후 Courtside 웹 캐시 무효화 토큰.
- `SOURCE_DEPLOY_KEY`: 비공개 `kbl_realdata`를 읽는 읽기 전용 deploy key의 개인키. 공개키는
  `kbl_realdata` Settings → Deploy keys에 쓰기 허용 없이 등록돼 있다. 그 한 저장소 전용이라 이
  키로 다른 저장소는 읽을 수 없다.

DB·재검증 secrets는 각 Python 실행 step에만 전달한다. checkout step에는 읽기 전용 deploy key만
전달하고(`persist-credentials: false`로 체크아웃 뒤 제거), Python 설치·의존성 설치 step에는
아무것도 전달하지 않는다. GitHub Actions 자체도 검증한 커밋 SHA로 고정한다.

## Cutover / recovery

1. 이 저장소가 `Public`이고 `kbl_realdata`가 `Private`인지 확인한다.
2. 세 Actions secrets(`DATABASE_URL`, `REVALIDATE_TOKEN`, `SOURCE_DEPLOY_KEY`)를 등록하고,
   `kbl_realdata`에 읽기 전용 deploy key(공개키)가 있는지 확인한다.
3. `KBL Daily Batch`를 수동 실행해 성공을 확인한다.
4. 경기 없는 시간에 `KBL Live Collector`를 `once=true`로 수동 실행해 한 주기만 확인한다.
5. `kbl_realdata` 저장소에 예약 workflow가 없는지 확인한다(CI·스모크 workflow만 있다).
6. 같은 시간대에 로컬 Windows `KBL_LiveCollector`/`KBL_DailyBatch`와 클라우드 잡을 함께
   켜지 않는다. 장애 시에는 클라우드 workflow를 끄고 로컬 태스크를 복구 경로로 쓴다.

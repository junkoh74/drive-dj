---
title: DRIVE-DJ 핸드오프
date: 2026-07-15
type: handoff
tags: [drive-dj]
---

# DRIVE-DJ 핸드오프 문서 (2026-06-16 ~ 2026-07-07 세션 종합)

> 다음 세션은 이 문서 + [[TODO]] + [[context-notes]] + 메모리(`drive-dj-phase-state`)로 시작.
> 전체 대화 원본: `~/Documents/drive-dj-transcripts/` (raw JSONL=완전판·시크릿 포함·공개금지 / readable MD=읽기용)
> 시크릿 값: `.env.local`(git 제외) + Supabase Secrets에만. 이 문서엔 없음.

---

## 1. 제품 정의 (왜 만드나)
- **문제**: 내 플레이리스트는 반복돼서 지겹고, Spotify/YouTube 추천은 (a)취향과 안 맞거나 (b)취향은 맞아도 **지금 상황(무드)과 안 맞음**.
- **해법**: 상황(활동·위치·날씨·시간·속도)을 읽어 **맥락에 맞는 "새로운" 곡**을 자동 재생.
- **차별점 = 맥락 정합 + 새로움 통제.** (취향 추천 자체는 Spotify가 이미 잘함)
- **비영리·무료 확정.** 광고 없음(Spotify/YouTube 약관 위반 위험). 비용은 캐싱으로 0 수렴 + 필요시 앱 밖 후원.

## 2. 현재 아키텍처 (동작 중)
```
[웹 v31 (GitHub Pages, 단일 index.html)]
  Spotify 로그인(PKCE, Client ID 내장: com.junkoh 계열) — 유저는 로그인만
  GPS(watchPosition)+Haversine 속도 → 상황
  ↓ 백엔드 호출(anon key)
[Supabase dj-bot (ref: rkaygkqucioliwhridbj, 서울)]
  Edge Functions (verify_jwt, CORS=junkoh74.github.io, rate limit):
    yt-search  : YouTube 무드 플레이리스트 채굴→곡제목 (4/min·IP)
    enrich     : ReccoBeats(energy/valence/tempo→bpm)+Deezer(bpm폴백)+Last.fm(태그→genres) → track_facts 캐싱 (6/min, backfill 1/min)
    weather    : 국가 라우팅 — KR=기상청(격자변환+실황PTY/T1H+예보SKY), 그외/실패=Open-Meteo (4/min)
    log        : users/serve/feedback/user_location 집계 + 새국가 텔레그램 알림 (20/min·user우선)
  DB (전부 RLS on, service_role만):
    users(마스터: id=spotify, country=로케일용, loc_country=실위치, settings jsonb)
    track_facts(곡 속성 캐시: bpm·energy·valence·danceability·acousticness·genres(Last.fm)·popularity_at·genres_at)
    situation_track_stats(상황×곡: serve/pos/neg — 전유저 공통, DB추천 후보원)
    user_situation_prefs(유저×상황 집계: 평균bpm/energy/valence·genre_weights·artist_weights)
    countries_seen(새 국가 감지→텔레그램), rate_limits(분창 카운터)
  Secrets: YOUTUBE_API_KEY(jun.koh8874 계정), LASTFM_API_KEY, KMA_WEATHER_KEY, TELEGRAM_BOT_TOKEN/CHAT_ID
[재생] 폰 Spotify 앱(유저 본인 Premium)이 백그라운드 재생 — 우리는 Web API로 원격 제어(리모컨 모델)
[iOS] ~/Documents/Projects/DriveDJ — Xcode 26.6, SwiftUI, Bundle ID com.junkoh.DriveDJ, 시뮬레이터 Hello world 확인
```

### 추천 파이프라인 (웹 v31 기준)
1. 상황 시그니처 = `activity|장소유형|날씨|시간대|속도구간` (activity=driving 고정, 속도: 정체0-20/주행21-60/빠른61-90/고속91+)
2. 시그니처 변화(또는 ↻) 시에만 재검색, 45s 디바운스. 새 추천은 **현재 곡 끝난 뒤** 반영(pendingUris)
3. 한국어+영어 무드 쿼리 각각 → yt-search(플레이리스트 6개→곡제목) → cleanTitle → Spotify 검색 → **isGoodMatch 50% 토큰일치만 채택(유사곡 대체 금지 — 사용자 요구)** → 선호점수+셔플 정렬 40곡 재생
4. enrich fire-and-forget(속성 캐싱) + logServe/logFeedback(집계)
5. 취향: 완청(끝12초내)=+1.0, 스킵 10초내=-1.0+곡제외 / 60초내=-0.5 / 이후=-0.3, 👍=+1.0+liked목록. 키="아티스트 / 장르" **결합 카테고리**(분리 아님 — 사용자 명시)
6. UI 핵심 3개만: 👍 좋아요 / ↻ 지금 상황 반영 / 취향 목록(편집: 선호[삭제]·좋아요[삭제·큐추가]·제외[해제·큐추가])

## 3. ❌ 폐기·중단된 경로들 (같은 실수 방지 — 중요)
| 폐기한 것 | 이유 |
|---|---|
| **YouTube IFrame 재생** (v1~v8) | 모바일 임베드는 **백그라운드 재생 불가**(프리미엄 무관, YouTube 정책). 주행 중 네비가 포그라운드여야 하므로 치명적 → Spotify 리모컨 모델로 전환. WakeLock으로 화면 유지도 시도했으나 "네비와 동시 사용 불가"로 기각 |
| **Spotify 추천 API(recommendations)·audio-features** | 2024.11부터 **신규/개발모드 앱 차단(403)**. "Spotify 추천으로 핸드오프" 구상 자체가 불가 → 자체 DB 추천으로 방향 확정 |
| **Spotify 플레이리스트 채굴** | 검색은 되지만 `/playlists/{id}/tracks`가 **에디토리얼·유저 플레이리스트 모두 403**(개발모드 제한). owner!=spotify 필터로도 해결 안 됨 → **YouTube에서 발굴, Spotify에서 재생** 하이브리드로 확정 |
| **Spotify 아티스트 장르** | 243곡 중 1곡만 값 존재(사실상 빈 데이터, 특히 한국 아티스트) → **Last.fm 태그로 전환**(track/artist.getTopTags) |
| **Spotify API limit 파라미터** | 값과 무관하게 400 "Invalid limit" (원인불명 특이동작). **모든 Spotify 호출에서 limit 제거**로 해결 — 재도입 금지 |
| **웹 장르 필터 UI** (포함/제외 검색드롭다운+칩) | Spotify 장르가 비어 무력했고 UX 단순화 방침 → v30에서 제거. 데이터(Last.fm genres)는 백엔드에 계속 축적 중이므로 나중에 재도입 가능 |
| **한국곡/해외곡 비율 슬라이더** | 실체가 "곡 국적"이 아니라 "검색어 언어 비중"이라 부정확 + UI 단순화 → 제거, 한/영 반반 자동 |
| **👎 싫어요·⏭ 다음곡 버튼** | 스킵 타이밍으로 비선호가 판별되므로 중복. Spotify 앱에서 넘기면 poll이 진행도로 자동 판정 → 버튼 제거 |
| **곡당 이벤트 로그 테이블(serves/user_feedback per-event)** | 곡×유저×시간으로 행 폭증 → **집계 테이블**(situation_track_stats, user_situation_prefs)로 교체. 생성 직후 드롭했음 |
| **Open-Meteo 단독 날씨** | 실측검증(서울 노원·도쿄): **"방금 그친/약한 비"를 "지금 비"로 과대판정**. 기상청이 정답 → KR=기상청 라우팅. 단 기상청도 정시관측+40분발표라 ~1h 시차(장마 분단위 갭은 못 잡음 — 한계 수용) |
| **Open-Meteo의 KMA 모델(models=kma_*)** | current/hourly가 전부 null 반환 → 사용 불가. 기상청 직접 연동(data.go.kr)으로 |
| **videoDuration=short·제목 믹스필터 등 YouTube 개별곡 검색** | 무드 키워드는 1~3시간 믹스 영상만 잡힘. 필터로 싸우다가 → **사용자 아이디어: 플레이리스트를 발굴원으로 역이용**이 정답이었음 |
| **BYOK(유저가 API키 입력)** | 초기 웹 임시책. 키보호 백엔드 필수 확정으로 폐기 (일반 유저에게 키 발급 요구는 UX 불가) |
| **ntfy.sh 알림** | 사용자가 텔레그램 사용 중이라 텔레그램 봇으로 결정 |
| **AI 오디오 분석(Cyanite·Sonoteller·Music.AI) / 악기 분석** | 오디오 파일 필요한데 Spotify가 오디오를 안 줌(미리듣기도 폐기) → 사실상 불가. energy/valence는 ReccoBeats(ID기반 무료)로 대체 |
| **수익화(광고 포함)** | 비영리 확정. 광고는 특히 Spotify 개발자 약관 위반 위험 큼 |

## 4. 실데이터 현황 (2026-07-07)
- users 2명(친구: 조은비, ChoKoh — 둘 다 premium/KR). **ChoKoh의 테스트 스킵 데이터는 삭제됨**(본인 요청, prefs 3행+stats 7행)
- track_facts **369곡** (genres 84%·bpm 85%·energy 84%). 조은비 실취향 형성 확인: `driving|city|cloudy|evening|idle`에서 avg BPM 132, rnb+2/soul+2
- **확인된 데이터 이슈 2개**: ① Last.fm 태그 노이즈(팬덤명·"my top songs"·아티스트명 태그 등)가 genre_weights 오염 ② serve와 feedback 사이 상황이 바뀌면 다른 signature에 기록(정합 이슈 — 알고리즘 설계 시 서빙 시그니처를 곡에 붙여다닐지 결정 필요)

## 5. 운영 규칙 (변하지 않는 것)
- **배포는 "업데이트 해줘" 명령 시에만.** 버전=배포 횟수(작업량 무관, 1회 배포=+1). 현재 배포 **v31**, 로컬 미배포분 있음 → 다음 배포 **v32**
- v32에 포함될 로컬 변경: 웹 fetchWeather→weather 함수 호출(WEATHER_LABELS 매핑), 위치국가 보고(reportLocCountry)
- 배포 시: index.html VERSION 올리고 CHANGELOG.md 기록 → commit/push → Pages 반영 확인(`?cb=N`)
- 시크릿은 Supabase Secrets(실사용) + .env.local(참고, git제외·600). 절대 커밋 금지
- 문서 체계: [[TODO]](할일) / [[CHANGELOG]](배포이력) / [[context-notes]](결정+이유) / [[GUIDE]](친구용 안내)

## 6. 보안 상태 (2026-07-07 완료분)
- 2FA: Supabase·GitHub·Google 전부(계정 단위). GitHub 리커버리코드=`~/Documents/Github Recovery Code`(레포 밖·700)
- YouTube 키 로테이션: rhrudwns88→jun.koh8874 계정 통일(무중단, KEY2로 사전검증 후 교체)
- CORS=junkoh74.github.io 제한 + rate limit(rate_limits 테이블, 429 검증) — 4개 함수 전부
- **남은 보안**: Spotify 토큰 서버검증(가짜 user_id 오염 방지 — 친구 확대 전 필수), iOS Keychain+App Attest(Phase3), iOS 전환 시 CORS 재검토(네이티브는 Origin 없어 무영향)
- 잔여 임시 함수: tg-test, yt-key-test (무해, 대시보드에서 삭제 가능)

## 7. ★ 다음 세션 우선순위 (사용자 지정)
> ✅ 1·2순위는 2026-07-15 세션에서 완료 — 검토 수치·구현 상세는 [[context-notes]] 2026-07-15 항목, 남은 후속은 [[TODO]] 참조.
1. ~~완료~~ **DJ세션/라이브세션 채굴 (1순위)** — 플레이리스트뿐 아니라 DJ set/live session 영상(예: https://youtu.be/f7jZoHs3QfY)도 발굴원으로. 단일 롱폼이라 playlistItems 불가 → videos.list 설명란(+챕터)의 타임스탬프 트랙리스트("12:34 Artist - Title") 파싱 → 기존 Spotify 매칭 재사용.
   ⚠️ **착수 전 "효과성 검토" 먼저 하기로 함**(사용자 마지막 요청, 미수행): 트랙리스트가 설명란에 실제로 얼마나 존재하는지(댓글에만 있는 경우 많음 — 댓글은 별도 API), 파싱 성공률, 무드검색시 세션 영상이 얼마나 잡히는지 샘플 검증 후 진행.
2. ~~완료~~ **태그 노이즈 정제 (2순위)** — enrich isGenreLike 강화 또는 **장르 사전 화이트리스트**(블랙리스트는 두더지잡기), 기존 track_facts.genres 재정제, user_situation_prefs.genre_weights 청소.
3. 이후: iOS 실기기 연결(iPhone 14 Pro USB+개발자모드+AppleID) → Spotify iOS SDK(대시보드에 Bundle ID 등록) → 확정 온보딩 UX(수동 ▶ 기본, 설정 옵트인 자동: authorizeAndPlayURI) 구현.

## 8. 환경 함정 (시간 아끼기)
- **~/Documents 간헐 EPERM 락**(TCC): index.html/.gitignore 등이 갑자기 읽기·쓰기 거부 → 보통 Claude(터미널) 재시작으로 해결. 근본책: 시스템설정>개인정보>전체 디스크 접근에 터미널 추가. cwd가 Documents면 python import도 실패 가능 → **/tmp에서 실행**
- data.go.kr 키는 발급 직후 ~1h 미활성(401). 신형 키는 hex라 Encoding=Decoding 동일
- Supabase MCP가 간헐 502 → REST API(service_role, /tmp에서 curl)로 우회 가능
- `!` 채팅 명령은 sudo 암호 입력 처리 못함 → sudo는 Terminal.app에서
- Xcode 26: Interface/Language 선택 없음(SwiftUI+Swift 고정). 시뮬레이터 목록=가상기기(실폰은 USB 연결해야 뜸)
- 친구 추가 시: Spotify 대시보드 User Management에 이메일 등록 필수(개발모드 25명 상한)

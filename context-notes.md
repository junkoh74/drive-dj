---
title: Drive DJ 컨텍스트 노트
date: 2026-07-15
type: notes
tags: [drive-dj]
---

# Drive DJ 컨텍스트 노트

> 결정과 이유의 시간순 기록. 최신 항목이 로드맵 섹션 바로 아래에 옴. 할 일은 [[TODO]], 배포 이력은 [[CHANGELOG]], 종합 현황은 [[HANDOFF]].

## 제품 로드맵 (2026-06-23 브레인스토밍 정리)

### 문제 정의 / 차별점
- 문제: 같은 플레이리스트가 지겨워 새 곡을 원함. 그런데 Spotify/YouTube 추천이 (a)취향과 안 맞거나 (b)취향은 맞아도 **현재 무드와 안 맞음**.
- 차별점: 더 나은 "취향 추천"이 아니라 **맥락(무드) 정합 + 새로움 통제**. (Spotify는 이미 취향은 앎.) BPM·악기는 수단의 하나일 뿐, 목적 아님.

### 추천 코어 스택
- 취향 부트스트랩: **Spotify 내 이력**(Get User's Top / Recently Played / Saved) — 추천기는 막혔어도 이력은 읽힘.
- 새 곡 발굴(취향+새로움): **Last.fm**(track/artist.getSimilar, tag.getTopTracks) / ListenBrainz. (Spotify Recommendations는 신규앱 차단.)
- 맥락→시드: **LLM** (상황·로케일 받아 그 언어로 무드 쿼리/시드 생성). 핵심 레버.
- 개인화: 내 피드백(완청/스킵/좋아요) 상황 태깅 학습.
- 새 곡 우선 **토글**(explore/exploit).
- 보조(선택): BPM=Deezer(무료, JSONP), 오디오/무드/악기=Cyanite·Sonoteller·Music.AI(유료, 백엔드, ISRC매칭). 악기는 오버킬.
- 재생: Spotify.

### 활동 컨텍스트 + 센서
- 타깃 상황: 헬스(혼자운동) / 러닝 / 자전거 / 차·오토바이 / 대중교통 출근 / 대중교통 주말.
- **폰만으로**: iOS Core Motion(stationary·walking·running·cycling·automotive) + GPS → 러닝·자전거·차 감지. 출근vs주말=캘린더. 차vs대중교통=정차패턴 or 유저 확인.
- **워치 필요**: 심박(헬스·강도). 실시간 심박은 watchOS 앱 필요.
- 외부기기: 개별 SDK 대신 **HealthKit(iOS)/Health Connect(Android)** 허브 경유(Apple Watch 최적, Galaxy/Mi는 동기화 시).

### 보안 / 멀티유저
- **비밀키는 앱에 절대 X → 백엔드 프록시(BFF).**
- Spotify: PKCE 유저 OAuth(Client ID 공개 OK, 토큰 iOS Keychain), >25명은 Extended Quota Mode 신청.
- YouTube Data / LLM / Last.fm / AI분석: **백엔드에서만** 호출.
- 백엔드 보호: Sign in with Apple, **App Attest/DeviceCheck**, rate limit, 캐싱.

### 캐싱 (비용 0 수렴)
- **★ 핵심: `track_facts`(곡별 속성 캐시) — 2026-06-23 생성 완료(dj-bot DB).** 컬럼: spotify_id(PK), isrc, name, artists[], artist_ids[], album, release_year, duration_ms, popularity, explicit, genres[], bpm(Deezer), energy/valence/danceability/acousticness(ReccoBeats 0~1, ID기반 무료급), analyzed_at, created_at. RLS on(백엔드 service_role만). **영구**(속성=사실). 같은 곡 재등장 시 외부 API 건너뜀.
  - 결정: AI 악기/무드 분석은 오디오 파일 필요 → Spotify 오디오 못 받아 보류. energy/valence는 ReccoBeats(ID기반 무료급)로 채움.
  - **정적 vs 변동 구분(2026-06-23)**: isrc·name·artists·album·release_year·duration·explicit·bpm·energy·valence·danceability·acousticness = 정적(1회). **popularity·genres = 변동** → `popularity_at`/`genres_at` 추가. popularity는 곡 재등장 시 매번 덮어씀(Spotify 트랙객체에 무료), genres는 TTL~30일 지나면 /artists 재조회.
  - ⚠️ 장르 분류 개편/세분화 시 genre 문자열이 바뀜 → 학습 키 "아티스트/장르"가 드리프트할 수 있음(user_feedback 설계 때 고려: 키 매핑/마이그레이션 or 상위 장르로 정규화).
  - 미완: 이 테이블을 채우는 백엔드 로직(Deezer BPM + ReccoBeats + Spotify 장르) = Phase 2.
  - **진행(2026-06-23): `enrich` Edge Function 배포·검증 완료.** ReccoBeats `/v1/audio-features?ids={spotifyIds}`(배치, href에서 id추출)로 energy/valence/danceability/acousticness/tempo→bpm, tempo 없으면 Deezer `track/isrc:`로 bpm 폴백. 캐시 조회 후 부족분만 호출, track_facts upsert(service_role 자동주입). 입력 {tracks:[{id,isrc,name,popularity,genres,...}]}. 테스트: Blinding Lights bpm171/energy0.73, Mr.Brightside bpm148/energy0.92 → DB 저장 확인.
  - **진행(2026-06-23~25): 웹 v29가 매칭 후 enrich를 fire-and-forget 호출** → 실주행으로 243곡 축적.
  - **장르 = Last.fm 태그로 전환(2026-06-30).** Spotify 아티스트 장르가 거의 전부 빈 값(243곡 중 1곡)으로 확인 → 폐기. enrich가 Last.fm track/artist.getTopTags(count≥10, junk필터, 상위6)로 `genres` 채움. enrich에 backfill 모드({backfill:N}) 추가해 기존 곡 일괄 채움.
  - **현황: genres 84% / bpm 85% / energy 84%** (243곡). 못 채운 16%=Last.fm에도 없는 무명곡. 노이즈 태그(팬덤·국가명) 소량.
  - LASTFM_API_KEY는 Supabase Secret 등록됨. enrich 함수 버전 v4(백필 포함).
  - ★ 미완(다음 작업): (1) 웹의 장르필터·취향카테고리가 아직 (빈)Spotify 장르를 봄 → **track_facts의 Last.fm genres를 읽게 전환**(enrich를 루프에서 await, 반환 genres 사용, Spotify /artists fetchGenres 제거). (2) BPM/energy 상황 매칭.
- 보조: `yt_pool`(무드쿼리→후보 제목), 짧은 TTL, 멀티유저 쿼터용. `context_cache`(맥락→무드어휘/시드, 상황+언어 키).
- 캐싱 X(매번 신선): **최종 선곡** → 셔플+이미들은곡제외+취향재정렬로 매번 다르게(새로움 유지).
- 비용이 유저 수가 아니라 "새 곡·새 상황 수"에만 비례.

### 국제화
- 기기 로케일 감지 → 그 언어로 검색어(LLM) → Spotify market=유저 국가, Nominatim accept-language=기기언어.

### 수익/지속가능성
- **유료화 목적 없음. 비영리·무료.**
- 광고는 Spotify/YouTube 개발자 약관 위반 위험 → 비추.
- 비용 충당: **캐싱(0 수렴)이 1순위** + 앱 밖 후원. (BYOK는 키 보호 백엔드 원칙과 충돌 → 영구 수단으로 안 씀. 현재 웹 프로토타입의 키 입력은 임시일 뿐.)
- 참고: YouTube Data API 키 검색은 개인 YouTube 추천 알고리즘에 영향 없음(서버호출, 시청기록 미반영).

### 단계 (번호 = 실행 순서)
- **Phase 0 (현재)**: 웹 프로토타입 — 아이디어/로직 검증. (YouTube 채굴→Spotify 정확매칭→Spotify 재생, 취향·장르 필터.) ※ 현재는 키 직접 입력(임시 BYOK).
- **★ Phase 1 (다음): 키 보호 백엔드 (얇게) — Supabase로 확정 (2026-06-23)**. 비밀키(YouTube/LLM/Last.fm/AI) 프록시 + 캐싱 + 클라이언트 인증. 이후 모든 지능을 이 위에서 안전하게 호출(BYOK 폐기).
  - **Edge Functions**(Deno/TS) = 키 보호 프록시(Secrets에 키 보관, 클라는 함수만 호출). **Postgres** = 캐시(track_facts, context_cache) + 추후 유저데이터. **Auth + RLS** = 유저 격리.
  - 조직 `junkoh74's Org`(nxotelzztwgymfrqdefs), 새 프로젝트 비용 $0/월. 기존 MetaCloset 프로젝트와 별개.
  - 1단계 함수: `yt-search`(YouTube 프록시) → 웹을 BYOK 직접호출에서 이걸로 교체. CORS는 Pages 도메인 허용.
  - **진행(2026-06-23)**: dj-bot 프로젝트 생성(ref rkaygkqucioliwhridbj, 서울). `yt-search` Edge Function 배포(verify_jwt=true). URL: https://rkaygkqucioliwhridbj.supabase.co/functions/v1/yt-search . body {queries:[...]} → {results:{q:[titles]}}. 시크릿 YOUTUBE_API_KEY는 대시보드에서 등록(MCP로 불가). 캐시 테이블은 Phase 2(BPM 등 실제 캐싱 시) 생성.
  - 미완: 웹의 ytTitlesFor를 이 함수 호출로 교체(Authorization: Bearer <anon JWT>), 그 후 클라에서 YouTube 키 입력 제거.
- **Phase 2: 추천 지능 고도화** (백엔드 위에서). ← **핵심 가치 검증.**
  - Last.fm 유사곡(취향+새로움), LLM 맥락 매핑, BPM(Deezer), 필요시 AI 분석(무드/악기)
  - 상황 태깅 취향 학습 + 새 곡 우선 토글
  - 개인화 핸드오프(소프트 블렌딩: 개인화비율 = min(0.8, 긍정수/30), 항상 채굴 ≥20%)
- **Phase 3 (폰 기준)**: iOS 앱. 폰 휴대 시나리오만(차·오토바이·자전거·폰 들고 러닝·대중교통).
  - Core Motion 활동감지 + GPS + 날씨 → 맥락
  - Spotify iOS SDK 백그라운드 재생
  - Phase 1 백엔드 재사용 + App Attest + 로케일 대응
  - **로그 UX**: 현재 웹 하단 로그 패널은 개발용 → 앱에선 유저에게 안 보임. 대신 **중요 이벤트(에러·갱신 실패·기기 없음 등)만 사용자 알림(토스트/알림창)** 으로. 상세 로그는 내부(디버그/텔레메트리)로만. (웹은 지금 그대로 둠)
- **Phase 4**: watchOS 앱 — 폰 없는 헬스·맨몸 러닝(워치 센서·심박, 셀룰러 워치, Spotify Web API 제어 제약).
- (메모) 분리 순서 근거: 백엔드는 필수+얇아서 먼저 세우면 지능을 처음부터 안전하게 + 재배선 헛수고 없음. 반대안(지능 먼저)은 검증은 빠르나 BYOK 임시·재작업 발생.

## DJ세션/라이브세션 채굴 — 효과성 검토 (2026-07-15, 구현 전 게이트)
- **방법**: YouTube Data API 실측 샘플링(검색 8쿼리×10영상=80개 + 예시영상 + 댓글 API 10개). 스크립트/원데이터는 세션 스크래치(휘발), 수치는 여기 기록.
- **결과 요약**
  - 무드 쿼리 그대로 type=video 검색: 롱폼(15분+) 98%, 그중 **설명란 트랙리스트(5곡+ 파싱) 18%** (7/39).
  - 영어 세션 쿼리("... dj set/mix/live session"): **34%** (13/38). 쿼리별 3~5/10로 안정적. 히트 시 영상당 12~61곡.
  - **한국어 "드라이브 dj mix" 계열: 0/9 — 한국 DJ믹스는 설명란 트랙리스트 문화가 없음** → 한국어 세션 전용 쿼리는 만들지 않기로. 단 한국어 일반 무드 쿼리의 감성 플레이리스트 영상엔 2~3/10 존재(혼합검색으로 공짜 커버 가능).
  - **파싱 성공률: 타임스탬프+구분자(-–—~|) 있으면 ~95%+** (22/22, 24/24, 61/63 등). "구분자 필수" 규칙이 **아티스트 없는 AI 양산 믹스(제목만 나열)를 자연 필터링**해줌 — 실패 케이스 대부분이 이 부류였고, 어차피 Spotify 매칭 불가한 곡들이라 오히려 이득.
  - 예시영상(f7jZoHs3QfY)은 설명란·상위20댓글 모두 트랙리스트 없음. **댓글(commentThreads) 구제율 +2/10** — 수확 대비 복잡도·지연 커서 v1에선 제외(옵션으로 TODO 유지).
- **결론: 조건부 GO** — ① 플레이리스트 검색(type=playlist)은 그대로 유지 ② 쿼리당 영상 검색 1회 추가(+100유닛): 영어="쿼리 + dj set"(수확 최고 경로), 한국어=무드 쿼리 그대로(감성 플리 영상 채굴) ③ 한국어 "dj mix" 세션 쿼리 제외 ④ 댓글 API 보류.
- **⚠️ 혼합검색(type=playlist,video) 시도 → 폐기** #deprecated — 이유: 쿼터 동일해서 매력적이었으나 실측 결과 영상이 랭킹을 100% 독식, 플레이리스트가 0개 잡혀 기존 채굴이 죽음(272곡→68곡 회귀). 검색 1회 추가 방식으로 확정. **재시도 금지.**
- **쿼터 계산**: 갱신당 현행 ~212유닛 → ~415유닛(일 10,000 한도에서 ~24회/일). 유저 2명 규모 문제없음, 확장 시 yt_pool 캐싱(기존 TODO)으로 해결.
- **구현·배포 완료(2026-07-15)**: yt-search v14 (v13=혼합검색 버전은 회귀로 즉시 폐기). curl 검증 — 플레이리스트 수확 유지(272/218곡) + 세션 트랙 병합(예: Folamour DJ set 곡들, 8090 발라드 트랙리스트). 웹 변경 불필요(응답 형식 동일, cleanTitle+isGoodMatch가 그대로 흡수).
- 참고: Supabase 무료티어 프로젝트가 1주 미사용으로 INACTIVE 됨(7/7→7/15) → restore로 복원(~1분). 장기 방치 시 재발 가능.
- **주문 유의**: 트랙리스트가 "제목 — 아티스트" 역순인 경우 있음(한국 영상 다수). 웹의 cleanTitle+isGoodMatch(50% 토큰)가 순서 무관 검색이라 그대로 흡수됨 — 다운스트림 변경 불필요.

## 태그 노이즈 정제 — 장르 사전 화이트리스트 (2026-07-15)
- **방식 확정: 화이트리스트**(203개 정본 장르 + 36개 정규화 매핑). 블랙리스트(기존 isGenreLike)는 두더지잡기라 폐기 #decision — [[HANDOFF]] 방침대로.
- 정규화 예: kpop→k-pop, hip hop→hip-hop, r&b→rnb, korean indie→k-indie, synth-pop→synthpop. 정규화 후 중복은 병합(genre_weights는 가중치 합산).
- **enrich v14 배포**: lastfmTags가 canonGenre(정규화→사전검사) 통과분만 저장. 사전은 enrich 소스 안에 인라인(단일 소스). 항목 추가 시 enrich의 GENRES/GENRE_NORM 수정.
- **기존 데이터 일괄 정제 완료**: track_facts.genres 316행 중 266행 정리, 태그 1261→766개(**39%가 노이즈였음** — korean·팬덤명·아티스트명·"my top songs" 등). 16행은 전량 노이즈라 NULL로 비움(backfill이 재시도). user_situation_prefs.genre_weights도 청소(조은비 1행: loud/groovy/british/melodic/moderate/my top songs 제거, 유효 10키 유지).
- **검증**: ① 정제 후 distinct 태그 113종 전부 사전 내 ② backfill 30곡 filled=0은 정상(무명곡이라 Last.fm 태그 자체가 없음, 원본 확인) ③ 긍정 케이스 — Blinding Lights 강제 재수집 → [synthwave, synthpop, pop, electropop]만 저장(연도·잡태그 필터됨).
- 관찰: "BIG Naughty (서동현)"처럼 아티스트명에 괄호 병기가 있으면 Last.fm 조회 실패 가능 — 필요해지면 괄호 제거 폴백 검토(기존 동작, 회귀 아님).

## iOS 개발 시작 (2026-07-07, Phase 3 착수)
- Xcode 26.6 설치 완료(iOS 26.5 플랫폼+시뮬레이터+Predictive Completion). xcode-select 전환 완료.
- **Bundle ID 확정: `com.junkoh.DriveDJ`** (Organization Identifier=com.junkoh). TestFlight 첫 업로드 전까지 변경 가능, 이후 영구 — 사용자 인지함.
- **✅ 프로젝트 생성·첫 실행 성공(2026-07-07 새벽)**: ~/Documents/Projects/DriveDJ. Xcode 26은 Interface/Language 선택 없음(SwiftUI+Swift 고정). Testing System=None, Storage=None, Team 미설정(시뮬레이터는 불필요). git author=junkoh74/jun.koh8874@gmail.com. iPhone 17 Pro 시뮬레이터에서 Hello world 확인.
- 참고: 시뮬레이터 목록=가상기기(iPhone 17세대만). 실기기(iPhone 14 Pro)는 USB 연결+개발자모드+Xcode에 Apple ID 추가 시 목록에 뜸 — 다음 세션.
- 이름 전략: DriveDJ=가제. 런칭 브랜드명은 Display Name(언제든 변경)+스토어명으로 처리, 프로젝트 재생성 불필요. Bundle ID만 TestFlight 첫 업로드 전 확정하면 됨.
- 확정 UX: 온보딩=수동 기본(▶ 버튼→authorizeAndPlayURI→자동복귀), 설정 옵트인 시 자동. 로그는 유저에게 숨기고 중요 이벤트만 알림.

## 운영 보안 (2026-07-07)
- **2FA 완료**: Supabase(관리자 계정, MS Authenticator+iCloud백업), GitHub(junkoh74, 리커버리코드는 ~/Documents/Github Recovery Code — 레포 밖·700). 2FA는 계정 전체 적용(프로젝트별 아님). Google은 API키 발급 계정 기준.
- **YouTube 키 로테이션**: rhrudwns88@gmail.com → jun.koh8874@gmail.com 계정으로 통일. YOUTUBE_API_KEY2로 사전 테스트(yt-key-test 임시함수) 후 YOUTUBE_API_KEY 교체, yt-search 검증 완료(133/181곡). 옛 키 폐기. 유저 영향 0(키는 서버에만).
- 임시 진단 함수 존재: tg-test(텔레그램), yt-key-test(키 테스트) — 대시보드에서 삭제 가능, verify_jwt라 해롭진 않음.
- **✅ CORS+rate limit 완료(2026-07-07)**: 4개 함수(yt-search/weather/enrich/log) CORS를 https://junkoh74.github.io 로 제한. rate_limits 테이블(키당 1행, 분창 덮어씀) + 한도: yt-search 4/min(IP)·weather 4/min(IP)·enrich 6/min(IP, backfill 1/min)·log 20/min(user_id 우선, 없으면 IP — CGNAT 대비). 입력 크기 캡(queries≤4, tracks≤60, serve ids≤60). 검증: CORS 헤더 확인 + 연타 5회째 429 ✓. 한도 근거=실사용의 2~3배(웹 자체 디바운스 45s/90s).
- 남은 보안 TODO: Spotify 토큰 서버검증(친구 확대 전), iOS Keychain+App Attest(Phase3). iOS 전환 시 CORS 허용 오리진 재검토 필요(네이티브는 Origin 없음→CORS 미적용이라 무영향).

## 날씨 정확도 & 국가별 소스 전략 (2026-07-06)
- **실측 검증**: 서울 노원(실제 비 그침) — 기상청 "지금 비 안 옴"=정답, Open-Meteo "비 1.5mm"=오답. 5개 도시 비교(서울/도쿄/뉴욕/베를린/시드니): OM은 **"약한 비/방금 그친 비" 경계에서 '지금 비'로 과대 판정** 패턴(서울·도쿄), 명확한 날씨(베를린·뉴욕 비여부)는 정확. 시드니는 비교지표 결함(BOM rain_since_9am=누적)으로 판정 불가.
- **결정**: 날씨는 국가 라우팅 구조 — **좌표의 국가**(유저 국적 아님, Nominatim country_code) 기준. KR=기상청(정식 data.go.kr API, 미착수—키 대기), 그 외=Open-Meteo 기본. 어댑터는 자동 추가 불가(수작업) → 소켓(라우팅)만 미리, 검증된 나라부터 꽂기. Spotify country=로케일용으로만.
- 참고: 날씨누리 비공식 엔드포인트 `weather.go.kr/w/wnuri-fct2021/main/current-weather.do?lat=&lon=` (키 없이 실황, 스크래핑이라 불안정—정식은 data.go.kr). JMA 도쿄 실황: bosai/amedas point json. NWS/BrightSky/BOM도 무키 확인.
- **새 국가 유저 텔레그램 알림 구축(검증완료)**: users.loc_country + countries_seen(kr 시드). 웹이 위치국가 변경 시 log{type:'user_location'} → 처음 보는 국가면 텔레그램 발송(TELEGRAM_BOT_TOKEN/CHAT_ID Secrets, 봇 @drive_dj_alert_bot). tg-test 진단함수 있음(임시). 웹 변경분은 로컬(다음 배포 포함).
- **✅ 완료(2026-07-06)**: weather Edge Function v2 배포·검증 — POST {lat,lon,country} → country=kr+`KMA_WEATHER_KEY`(Secrets, 대문자 이름 확정)면 기상청(DFS 격자변환 + 초단기실황 PTY/T1H + 초단기예보 SKY), 실패/그외=OM 폴백(kma_error 표기). 반환 {key,temp,isDay,source}. 실측: 노원 source:kma ✓, 뉴욕 source:openmeteo ✓. 웹 fetchWeather도 함수 호출로 교체(WEATHER_LABELS 매핑, 로컬 미배포—다음 배포 v32 포함).
- 한계 인지: 기상청 실황도 정시관측+40분발표라 최대 ~1h 시차 — 오락가락 비(장마)의 분단위 갭은 어떤 무료 API도 못 잡음. 무드 용도로는 허용.

## DB 추천 파운데이션 (2026-07-01)
곡당 이벤트 행은 폭증 → **집계 테이블**로 저장(유저·상황 기준). 아래 테이블 + track_facts가 향후 "DB에서도 신규곡 추천" 알고리즘의 재료.

- **users** (마스터, PK=Spotify user id): display_name, product, country, locale, settings jsonb(향후 앱설정), created_at, last_seen_at. 로그인 시 log{type:'user'}로 upsert. 다른 테이블 user_id가 논리 참조(현재 소프트, FK 미설정).

- **활동(activity) 차원 (2026-07-01)**: signature 맨 앞에 activity 포함(`driving|place|weather|tod|speed`). situation_track_stats·user_situation_prefs에 activity 컬럼 추가. **지금은 driving 고정**(웹 자동감지 불가), 러닝·사이클링·커뮤팅은 Phase2/3(디바이스 감지) 이후. 파운데이션만 준비 — 활동 추가 시 상황이 자동 분리됨.
- **situation_track_stats** (전 유저 공통, 상황×곡): signature, track_id, dims(activity 포함), serve_count, pos_count(완청/좋아요), neg_count(스킵). "어떤 상황에 어떤 곡이 서빙/호평됐나" → DB추천 후보원 + 분석. PK(signature,track_id).
- **user_situation_prefs** (유저×상황 집계): user_id, signature, pos/neg_count, bpm_sum/n·energy_sum/n·valence_sum/n(→평균, 긍정 신호만), genre_weights jsonb, artist_weights jsonb. **곡당 아님 → 크기 유한**(유저×상황 수). "이 상황에서 이 유저의 평균 BPM·선호 장르/아티스트". PK(user_id,signature).
- **log Edge Function**(v2, verify_jwt): {type:'serve',user_id,situation,track_ids[]} → situation_track_stats serve_count 증가. {type:'feedback',user_id,situation,track_id,action,weight} → stats pos/neg + user_situation_prefs 집계(track_facts 속성으로 평균·가중치). read-modify-write(단일유저 스케일 ok; 멀티유저 규모면 원자적 RPC로 전환).
- **웹 배선**(로컬 미배포): showProfile서 spUserId 캡처. logServe(final) 서빙 로그. logFeedback: 완청 full_play+1.0 / 👍 like+1.0 / 스킵 skip -pen. situationObj()=현재 상황.
- **향후 알고리즘(파운데이션만 준비, 미구현)**: 상황 S에서 user_situation_prefs의 평균BPM·장르·아티스트로 후보 필터/정렬 + situation_track_stats에서 그 상황 호평곡(아직 안 들은 것=새로움) → YouTube뿐 아니라 **우리 DB에서도** 추천. 곡 속성 매칭(BPM/energy)까지.
- 참고: 실사용 데이터는 웹 배포(v30) 후부터 쌓임. serves per-event/user_feedback per-event 테이블은 폐기(집계로 대체).

## 취향 시스템 v28 (2026-06-21, 로컬 v28-prefcat / 미푸시)
- prefs = { blocked:{uri:name}, fav:{"아티스트 / 장르":score}, liked:{uri:{name,artist}} }.
- **선호 단위 = "아티스트 + 장르" 결합 카테고리**(따로 아님). 트랙의 (아티스트 × 각 장르) 조합마다 점수. 장르 없으면 "아티스트 / (unknown)".
- 완청(자연종료) → 결합카테고리 +1.0. 👍 → liked 추가 + +1.0. 👎/스킵 → 곡 제외 + 이미 점수 있는 카테고리 감점(<10s -1.0/<60s -0.5/≥60s -0.3).
- 완청/스킵/강제교체 구분: skipFlag·forceFlag·trackStartTs (폴링 3s).
- 추천 정렬: scoreTrack(결합카테고리 점수합) 내림차순 + 랜덤 지터.
- 취향 패널: 선호 카테고리[삭제] / 좋아요 곡(클릭→다음 큐, [삭제]) / 제외 곡(클릭→다음 큐, [해제]). 큐추가=POST /me/player/queue.
- 한국/해외 비율 슬라이더 제거(한·영 쿼리 절반씩 자동 혼합).

## 결정 사항과 이유
- **음악 = YouTube + 프리미엄 대상.** 백그라운드 재생·광고 제약을 프리미엄이 해결. 웹은 IFrame Player API로 제어.
- **키 없는 의존만 사용:** 날씨 Open-Meteo, 역지오코딩 Nominatim. 외부 키는 YouTube Data API 1개뿐.
- **API 키는 레포에 넣지 않음.** GitHub Pages는 공개이므로 키 노출 방지 → 앱 입력창 + localStorage 저장.
- **호스팅 = GitHub Pages.** 주행 중 폰은 셀룰러·집 밖 → 노트북 터널은 위험. Pages는 독립 HTTPS.
- **속도:** coords.speed(m/s) 우선, null이면 Haversine 직접계산. 데스크톱은 보통 null이라 폴백 필수. 5샘플 이동평균.
- **재검색 트리거:** 매 GPS틱이 아니라 "시그니처(속도구간|날씨|시간대|장소유형)" 변화 시에만. 곡 끊김 방지 + 쿼터 절약. 추가로 45초 디바운스.
- **장소유형은 rough.** 메타데이터 기반이라 해안/산 판별 정확도 낮음. 속도구간(특히 고속도로)을 우선 신호로 사용. v2에서 스트리트뷰 비전으로 보강.

## 알려진 한계
- Nominatim 브라우저 호출은 저빈도여야 함(90초 throttle 적용). 차단 시 장소는 무시되고 앱은 계속 동작.
- 시그니처가 바뀌면 현재 곡이 끊기고 새 플레이리스트 시작 → v2에서 "곡 끝나고 반영" 옵션 검토.
- YouTube Data API 무료 쿼터 1만 units/일, search=100 units → 하루 약 100회 검색. 디바운스로 충분.

## 주행 테스트 체크
- HTTPS Pages URL을 폰 브라우저로 열기 → 위치 권한 "허용"
- "시작" 탭(자동재생 잠금 해제 제스처) → 첫 곡 재생 확인
- 가능하면 조수석에서 조작. 운전 중 화면 조작 금지.

## v2 배포 설계 (2026-06-16 합의, 오늘은 보류)
- **키는 서버가 보관:** 앱 → 백엔드 프록시(키 1개) → YouTube API. 사용자 키 발급 부담 제거.
- **쿼터 함정:** 무료 10,000 units/일, search=100 → 하루 100검색(전체 합산)뿐. 사용자 몇 명에 즉시 소진.
- **해결 A(추천): 상황 큐레이션 + 캐싱.** 시그니처(장소5×날씨6×시간대5×속도4≈수백)는 유한 → 조합별 결과 캐싱 → 런타임 호출 0 수렴.
- 해결 B: Google 쿼터 증액 신청. 해결 C: 구글 OAuth(사용자 맥락 재생, 키 발급 불요 / 검색 쿼터는 앱단위라 A와 병행).
- 무드 매핑 로직은 그대로 재사용, 앞에 캐싱 서버만 추가.

## 추천 단위 = 개별 곡 (2026-06-16)
- 믹스/플레이리스트 영상이 아니라 개별 곡 추천으로 변경.
- 검색어 접미사 `플레이리스트` → `노래`.
- videos.list contentDetails로 재생시간 필터(90s~9min) 적용해 긴 믹스 제외. 남은 곡<5면 폴백(미적용).

## 추천 파이프라인 전환 (v7, 2026-06-16)
- 문제: 무드 키워드 검색(type=video)은 결과 전부가 1~3시간 믹스 단일영상. videoDuration=short도 한계.
- 해법(사용자 제안): 플레이리스트를 곡 발굴 소스로 역이용.
  1) search?type=playlist 로 무드 플레이리스트 6개.
  2) playlistItems 로 각 멤버(개별 곡) 수집.
  3) 여러 모음에 중복 등장한 곡 빈도 집계 → '대표곡' 우선.
  4) 빈도 desc + 선호 아티스트 점수로 정렬.
  5) 상위 40개 videos.list 길이검증(삭제/긴영상 제거) 후 재생.
- 단일 긴영상은 type=playlist 검색에 안 잡혀 자연 배제됨.
- 폴백: 플레이리스트 0개면 searchVideosFallback(type=video,short).
- 쿼터: search(100)+playlistItems(6)+videos.list(1) ≈ 107 units/추천.

## 광고 (2026-06-16)
- 임베드 IFrame은 그 브라우저가 youtube.com 로그인(프리미엄) 상태여야 광고 제거. 비로그인이라 광고 발생.
- 완전 무광고는 v2에서 YouTube OAuth 필요.

## YouTube → Spotify 전환 (v9, 2026-06-16, 결정적 변경)
- **이유:** 주행 중 네비가 포그라운드여야 하는데 임베드 YouTube는 포그라운드에서만 재생됨(백그라운드 불가, Premium 무관). WakeLock으로 화면 켜둬도 네비와 동시 불가.
- **해법 = 리모컨 아키텍처:** 우리 앱은 두뇌(상황→곡선택), 실제 재생은 백그라운드 되는 음악 앱에 위임. 외부 재생제어 공개 API가 있는 건 Spotify뿐(YouTube는 없음).
- **구조:** 폰 Spotify 앱이 활성 기기로 백그라운드 재생 → 우리 웹앱이 Web API(PUT /me/player/play)로 곡 푸시. 네비 포그라운드 유지하며 음악 지속.
- **인증:** OAuth PKCE (백엔드 불필요, 클라이언트만). Client ID는 사용자가 입력(localStorage). Redirect URI = Pages URL을 Spotify 앱에 등록 필수.
- **추천:** search type=playlist(무드) → playlists/{id}/tracks 추출 → 중복빈도+선호아티스트 정렬 → play uris. 빈약하면 type=track 검색 보강.
- **제약:** (1) Spotify Premium 필수(재생제어), (2) 폰 Spotify 앱이 켜져 활성 기기여야 함(없으면 404), (3) /recommendations·audio-features는 신규앱 deprecated → search만 사용.
- **취향:** prefs.blocked(uri), prefs.artist(점수). localStorage 'djPrefsSp'.
- 상황변화는 폴링(3s)으로 현재 곡 끝나는 시점에 반영(pendingUris).

## 제품화 로드맵 — 앱으로 만들기 (2026-06-16 합의)
- 현재 "사용자가 Client ID 직접 입력"은 개발자용 임시 방식. 일반 사용자엔 부적합.
- **핵심 변화: Client ID를 우리가 보유.** PKCE라 Client ID 공개 안전 → 앱에 내장 → 사용자는 "Spotify로 로그인" 버튼만 탭.
- **관문 1: Extended Quota Mode 신청.** 개발모드는 본인+수동등록 25명 한정. 불특정 다수 로그인하려면 Spotify에 Extended Quota Mode 심사 신청 필요.
- **관문 2: 네이티브 앱.** iOS/Android + Spotify SDK → 매끄러운 백그라운드 제어, 안정적 GPS, 스토어 배포, 딥링크로 Spotify 앱 자동 실행.
- 로드맵: (지금) 웹+내 ClientID 입력/본인만 → (v2) PWA+우리 ClientID 내장+"Spotify 로그인" → (v3) 네이티브앱+Spotify SDK.

## ⚠️ Spotify limit 파라미터 거부 (v14, 2026-06-16)
- 증상: /search 및 /playlists/{id}/tracks 에 limit 파라미터를 넣으면 값(20·50 등 유효범위)과 무관하게 400 "Invalid limit".
- 자가진단(variant 탐색)으로 확인: limit 제거 시 정상. market=KR는 무관(정상).
- 원인 불명(Spotify 측 특이동작/개발모드앱 가능성). 규격상 limit 0~50은 유효한데 거부됨.
- 조치: 모든 Spotify 호출에서 limit 제거, 기본값 사용. (이 때문에 플레이리스트 곡 추출이 전부 실패해 곡이 3개만 나왔었음.)

## ⚠️ 에디토리얼 플레이리스트 곡 접근 403 (v22, 2026-06-16)
- 증상: /search?type=playlist는 결과를 주지만, 그 중 Spotify 공식(에디토리얼) 플레이리스트의 /playlists/{id}/tracks가 전부 403 Forbidden.
- 원인: Spotify 2024 제한 — 공식/알고리즘 플레이리스트 곡 접근 차단(특히 개발모드 앱).
- 무드 검색결과는 대부분 Spotify 공식이라 순수 플레이리스트 채굴이 거의 불가.
- 조치: 검색 결과에서 owner.id==='spotify' 제외하고 **유저 생성 플레이리스트만 채굴**(이건 200). 유저PL이 0이면 트랙검색 폴백(최후수단).
- 한계: 유저PL이 적게 잡히면 곡이 빈약. 근본적으론 Extended Quota Mode 승인 시 에디토리얼 접근 가능해질 수 있음(미확인).
- 확정(v24): 유저 플레이리스트 곡도 전부 403. 우리 개발모드 앱에선 Spotify 플레이리스트 곡 추출이 전면 불가.

## YouTube 검색 + Spotify 재생 하이브리드 (v24, 2026-06-16, 결정적)
- 사용자 결정: 곡 발굴은 YouTube(플레이리스트 채굴 가능), 재생은 Spotify(백그라운드).
- 흐름: YouTube 무드 플레이리스트 검색(type=playlist)→playlistItems로 곡 '제목' 수집 → 각 제목 cleanTitle 후 Spotify track 검색으로 같은 곡 URI 매칭 → Spotify 재생.
- 키 2개 필요: YouTube Data API 키(입력, localStorage 'ytKey') + Spotify OAuth.
- 한/영 비율: 한국어/영어 YouTube 쿼리로 각각 제목 수집 후 비율대로 혼합, 매칭실패 대비 2배수 수집.
- 동시 Spotify 검색은 mapLimit(6)로 제한. N=40.
- 한계: YouTube 제목 파싱(cleanTitle)이 완벽치 않아 일부 곡 매칭 실패/오매칭 가능. 쿼터: YT search 100×2 + playlistItems.

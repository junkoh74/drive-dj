# Drive DJ 컨텍스트 노트

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
- **★ 핵심: `track_facts`(곡별 속성 캐시).** id(spotify track id/ISRC) → bpm(Deezer)·genres(Spotify 아티스트)·mood/energy/instruments(AI). **영구**(속성은 사실 → 안 변함). 같은 곡 재등장 시 API 건너뛰고 즉시 읽기 → 처리 빠름 + AI/Deezer 비용 곡당 1회로 고정(전 유저 공유). Phase 2에서 생성.
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
- **Phase 4**: watchOS 앱 — 폰 없는 헬스·맨몸 러닝(워치 센서·심박, 셀룰러 워치, Spotify Web API 제어 제약).
- (메모) 분리 순서 근거: 백엔드는 필수+얇아서 먼저 세우면 지능을 처음부터 안전하게 + 재배선 헛수고 없음. 반대안(지능 먼저)은 검증은 빠르나 BYOK 임시·재작업 발생.

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

# Drive DJ 컨텍스트 노트

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

# Drive DJ — To-Do (이후 업무)

> 현재 위치: 웹 프로토타입 **v30** 배포됨. Phase 1(키보호 백엔드) 완료, Phase 2(추천 지능) 진행 중.
> 상세 맥락은 context-notes.md 참고. 배포는 "업데이트 해줘" 시 (버전=배포횟수, CHANGELOG 기록).

## ✅ 지금까지 완료
- 웹 프로토타입: 상황(위치·날씨·시간·속도)→YouTube 채굴→Spotify 정확매칭→백그라운드 재생
- 키보호 백엔드(Supabase): `yt-search`, `enrich`, `log` Edge Functions
- 곡 속성 캐시 `track_facts`: BPM·energy·valence(ReccoBeats)+bpm폴백(Deezer)+장르(Last.fm)
- DB 파운데이션: `users` / `situation_track_stats` / `user_situation_prefs` (집계·activity 차원)
- 취향: 좋아요 + 완청/스킵 자동판정(진행도) → 점수·DB 반영. UI 단순화.

## 🔜 다음 (Phase 2 — 추천 지능 완성)
- [ ] **DB 기반 추천 알고리즘**: 상황 S에서 `situation_track_stats`(호평곡)+`user_situation_prefs`(평균BPM·선호장르/아티스트)로 후보 생성·정렬, 이미 들은 곡 제외(새로움)
- [ ] **YouTube + DB 혼합 추천**: 콜드스타트=YouTube, 데이터 쌓이면 DB 비중↑ (소프트 블렌딩: 개인화비율 = min(0.8, 긍정수/30), 항상 채굴 ≥20%)
- [ ] **BPM/energy 상황 매칭**: 활동·속도 → 타깃 BPM·energy 범위로 후보 필터/정렬
- [ ] **Last.fm 유사곡** 발굴 (취향+새로움: track/artist.getSimilar)
- [ ] **LLM 맥락 매핑**: 상황→시드/쿼리 생성 (하드코딩 표 대체), 로케일별 언어
- [ ] **새 곡 우선 토글** (explore/exploit)
- [ ] 실주행으로 3개 DB 테이블 실데이터 축적 → 알고리즘 검증

## 🌐 국제화
- [ ] 기기 로케일 감지 → 그 언어로 검색/시드, Spotify market=유저 국가, Nominatim accept-language

## 🔐 멀티유저/배포 준비 (Phase 1 마무리)
- [ ] per-user 인증(Sign in with Apple 등) + rate limit (현재 anon 키만)
- [ ] App Attest / DeviceCheck (앱 검증)
- [ ] 캐싱(yt_pool 등) — 멀티유저 쿼터 보호
- [ ] Spotify Extended Quota Mode 신청 (>25명)
- [ ] 집계 read-modify-write → 원자적 RPC (동시성 규모 시)

## 📱 Phase 3 — iOS 네이티브 앱
- [ ] iOS 앱: Core Motion 활동감지 + GPS 백그라운드 + Spotify iOS SDK 백그라운드 재생
- [ ] **활동 자동 구분**(러닝/사이클링/커뮤팅) → signature의 activity 채움 (지금 driving 고정)
- [ ] **로그 UX**: 유저에겐 로그 숨김, 중요 이벤트(에러·갱신실패·기기없음)만 알림(토스트). 상세는 내부 텔레메트리
- [ ] Phase 1 백엔드 재사용

## ⌚ Phase 4 — watchOS
- [ ] 폰 없는 헬스·맨몸 러닝(워치 센서·심박, 셀룰러 워치, Spotify Web API 제어)

## 🧹 데이터 품질/개선
- [ ] Last.fm 태그 노이즈 정제 (팬덤·국가명 태그 필터 강화)
- [ ] 장르 드리프트 대응 (카테고리 정규화/매핑)
- [ ] popularity(재등장 갱신)·genres(TTL 30일) 갱신 실동작 확인
- [ ] 새로움(유저가 들어본 곡) 추적 설계 — 압축 방식(폭증 방지)

## 💰 지속가능성 (비영리·무료 전제)
- [ ] 비용은 캐싱으로 0 수렴. 광고 X(Spotify/YouTube 약관 위반 위험). 필요 시 앱 밖 후원.

---
title: Drive DJ To-Do
date: 2026-07-15
type: todo
tags: [drive-dj]
---

# Drive DJ — To-Do (이후 업무)

> 현재 위치: 웹 프로토타입 **v31** 배포됨(로컬 미배포분 있음, 다음 배포 v32). Phase 1(키보호 백엔드) 완료, Phase 2(추천 지능) 진행 중.
> 상세 맥락은 [[context-notes]] 참고. 배포는 "업데이트 해줘" 시 (버전=배포횟수, [[CHANGELOG]] 기록).

## ✅ 지금까지 완료
- 웹 프로토타입: 상황(위치·날씨·시간·속도)→YouTube 채굴→Spotify 정확매칭→백그라운드 재생
- 키보호 백엔드(Supabase): `yt-search`, `enrich`, `log` Edge Functions
- 곡 속성 캐시 `track_facts`: BPM·energy·valence(ReccoBeats)+bpm폴백(Deezer)+장르(Last.fm)
- DB 파운데이션: `users` / `situation_track_stats` / `user_situation_prefs` (집계·activity 차원)
- 취향: 좋아요 + 완청/스킵 자동판정(진행도) → 점수·DB 반영. UI 단순화.

## 🔜 다음 (Phase 2 — 추천 지능 완성)
- [x] **★1순위: DJ세션/라이브세션 채굴** (2026-07-15 완료) — 효과성 검토(샘플 실측) 후 yt-search v14 배포. 쿼리당 영상 검색 1회 추가(영어="+ dj set", 한국어=무드 그대로) → videos.list 설명란 타임스탬프 트랙리스트 파싱(구분자 필수=AI믹스 필터, 15분+·5곡+·상위 3영상). 웹 변경 불필요. 상세·수치는 [[context-notes]] 2026-07-15 참조.
  - [ ] (옵션) 댓글(commentThreads) 트랙리스트 구제 — 실측 +2/10, 지연·복잡도 대비 보류
  - [ ] (관찰) 세션 채굴분의 Spotify 매칭률·실주행 체감 확인 후 영상 수(현 3개)·쿼리 튜닝
- [x] **★2순위: 태그 노이즈 정제** (2026-07-15 완료) — 장르 사전 화이트리스트(203종+정규화 36종)로 enrich v14 배포, track_facts.genres 일괄 재정제(태그 39%가 노이즈였음), genre_weights 청소. 사전 추가/수정은 enrich 소스의 GENRES/GENRE_NORM. 상세는 [[context-notes]] 2026-07-15.
- [ ] **DB 기반 추천 알고리즘**: 상황 S에서 `situation_track_stats`(호평곡)+`user_situation_prefs`(평균BPM·선호장르/아티스트)로 후보 생성·정렬, 이미 들은 곡 제외(새로움)
- [ ] **YouTube + DB 혼합 추천**: 콜드스타트=YouTube, 데이터 쌓이면 DB 비중↑ (소프트 블렌딩: 개인화비율 = min(0.8, 긍정수/30), 항상 채굴 ≥20%)
- [ ] **BPM/energy 상황 매칭**: 활동·속도 → 타깃 BPM·energy 범위로 후보 필터/정렬
- [ ] **Last.fm 유사곡** 발굴 (취향+새로움: track/artist.getSimilar)
- [ ] **LLM 맥락 매핑**: 상황→시드/쿼리 생성 (하드코딩 표 대체), 로케일별 언어
- [ ] **새 곡 우선 토글** (explore/exploit)
- [ ] 실주행으로 3개 DB 테이블 실데이터 축적 → 알고리즘 검증

## 🌐 국제화 / 날씨
- [x] **날씨 국가 라우팅** (2026-07-06 완료): weather Edge Function — KR=기상청(KMA_WEATHER_KEY, 격자변환+PTY/SKY), 그 외/실패=OM 폴백. 웹 연결됨(배포 대기)
- [x] 새 국가 유저 텔레그램 알림 (countries_seen + log user_location, 2026-07-06 검증)
- [ ] 타국 유저 등장 시 그 나라 기상청 어댑터 추가 (JMA 엔드포인트 파악됨)
- [ ] 기기 로케일 감지 → 그 언어로 검색/시드, Spotify market=유저 국가, Nominatim accept-language

## 🔐 멀티유저/배포 준비 (Phase 1 마무리)
- [ ] per-user 인증(Sign in with Apple 등) + rate limit (현재 anon 키만)
- [ ] App Attest / DeviceCheck (앱 검증)
- [ ] 캐싱(yt_pool 등) — 멀티유저 쿼터 보호
- [ ] Spotify Extended Quota Mode 신청 (>25명)
- [ ] 집계 read-modify-write → 원자적 RPC (동시성 규모 시)

## 📱 Phase 3 — iOS 네이티브 앱
- [ ] iOS 앱: Core Motion 활동감지 + GPS 백그라운드 + Spotify iOS SDK 백그라운드 재생
- [ ] **온보딩/재생 시작 플로우** (확정 2026-07-07, 유저 컨트롤 원칙): 기본=수동 — 앱 실행 → 로딩(상황분석, 추천계산 겹침) → "▶ Spotify에서 재생 시작" 버튼 유저 탭 → `authorizeAndPlayURI(1번곡)` 전환 → 자동복귀·재생. **설정 옵트인 시 자동 모드** — 앱 켜면 묻지 않고 즉시 전환·재생(문구: '화면이 잠깐 전환됩니다'). 설정 저장=users.settings jsonb {auto_spotify_handoff}+로컬. Spotify 프로세스 살아있으면 전환 자체를 스킵. 전환 순간은 iOS상 가릴 수 없음.
- [ ] **활동 자동 구분**(러닝/사이클링/커뮤팅) → signature의 activity 채움 (지금 driving 고정)
- [ ] **로그 UX**: 유저에겐 로그 숨김, 중요 이벤트(에러·갱신실패·기기없음)만 알림(토스트). 상세는 내부 텔레메트리
- [ ] Phase 1 백엔드 재사용

## ⌚ Phase 4 — watchOS
- [ ] 폰 없는 헬스·맨몸 러닝(워치 센서·심박, 셀룰러 워치, Spotify Web API 제어)

## 🧹 데이터 품질/개선
- [x] Last.fm 태그 노이즈 정제 (2026-07-15, 장르 사전 화이트리스트 — 위 ★2순위 참조)
- [x] 장르 드리프트 대응 (2026-07-15, GENRE_NORM 정규화 매핑으로 kpop/k-pop·hip hop/hip-hop 등 병합)
- [ ] popularity(재등장 갱신)·genres(TTL 30일) 갱신 실동작 확인
- [ ] 새로움(유저가 들어본 곡) 추적 설계 — 압축 방식(폭증 방지)

## 💰 지속가능성 (비영리·무료 전제)
- [ ] 비용은 캐싱으로 0 수렴. 광고 X(Spotify/YouTube 약관 위반 위험). 필요 시 앱 밖 후원.

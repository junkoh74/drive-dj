---
title: Drive DJ 체크리스트 (초기 MVP)
date: 2026-06-16
type: todo
tags: [drive-dj]
---

# Drive DJ 체크리스트

> 2026-06-16 첫날 MVP 당시 기록(역사 자료 — YouTube IFrame 등 폐기 경로 포함). 이후 진행은 [[TODO]] 참조.

## MVP (오늘 주행 테스트)
- [x] 단일 index.html 작성
- [x] 위치/속도 (watchPosition + coords.speed, Haversine 폴백, 이동평균)
- [x] 날씨 (Open-Meteo, 키 불필요)
- [x] 장소유형 (Nominatim 역지오코딩, best-effort)
- [x] 무드 → YouTube 검색 쿼리 생성
- [x] YouTube IFrame 재생 + 시그니처 변화 시 재검색
- [ ] GitHub Pages 배포
- [ ] YouTube Data API 키 발급 (사용자)
- [ ] 폰에서 위치 권한 + 재생 확인
- [ ] 실제 주행 테스트

## v2 (이후)
- [ ] 곡 끊김 최소화 (현재 곡 끝나고 다음 큐 반영 옵션)
- [ ] 스트리트 뷰 비전 분석으로 무드 보강
- [ ] 무드 매핑 사용자 피드백(👍/👎) 학습
- [ ] 한국 서비스(멜론 등) 또는 Spotify 분기

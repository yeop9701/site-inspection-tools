# 건축물 상태평가 프로그램(SafetyMan) 웹앱 재현 — 설계 문서

> 원본: `건축물상태평가프로그램및메뉴얼` 저장소의 `상태평가 프로그램 메뉴얼.hwp`
> (한국시설안전기술공단, 2008) 를 분석해 정리한 설계안. 아직 구현 전 단계.

## 1. 원본 프로그램이 하는 일 (요약)

건물 1개에 대해 **표본층**(대표로 조사한 층, 기본 7개, 추가 가능)을 선정하고,
표본층마다 5가지 부재(기둥/내력벽/큰보/작은보/슬래브)에 대해

- **안전성평가**: 구조해석 소요강도 vs 부재설계강도(또는 측정강도) → 손상여부
- **상태평가**: 콘크리트강도, 균열, 콘크리트중성화, 염화물함유량, 철근부식,
  표면노후(박리/박락·층분리/누수·백태/철근노출) — 6개 세부항목

를 입력하면 부재별 A~E 등급이 매겨지고, 층별 대표등급 → 건물 전체
**안전성평가등급 / 상태평가등급 / 종합평가등급**이 산출된다.
여기에 건물 전체 단위의 **변위·변형**(기울기, 부동침하) 평가가 별도로 더해진다
— 이 부분은 이미 만든 `tools/tilt-survey`가 담당하는 영역과 겹친다.

## 2. 데이터 모델

```
Project (건물 1개 = 진단 1건)
├─ basicInfo
│   ├─ buildingName, address, builtYear, mainUse
│   ├─ area: { siteArea, buildingArea, totalFloorArea }
│   ├─ owner, manager
│   ├─ surveyType: '정밀점검' | '정밀안전진단'
│   └─ coverPhoto
│
├─ sampleFloors[]                     // 표본층, 기본 7개 + 추가 가능
│   ├─ floorName (예: "지상7층")
│   ├─ zone, structureType, structureMaterial
│   └─ members: {
│        column: MemberEval, wall: MemberEval,
│        girder: MemberEval, beam: MemberEval, slab: MemberEval
│      }
│
├─ tiltSettlement                     // 건물 단위, tilt-survey 데이터 재사용/연동 검토
│   ├─ tilt: { measured, grade }
│   └─ settlement: { measured, grade }
│
└─ result (계산됨, 저장은 해도 되고 매번 재계산해도 됨)
    ├─ perFloorGrade[]
    └─ overall: { safetyGrade, conditionGrade, totalGrade }

MemberEval (부재 1종류에 대한 평가 묶음)
├─ safety: { items: [{ requiredStrength, memberStrength, damaged }] }  // 최대 5개
└─ condition: {
     concreteStrength: { points:[{measured1,2,3}], designStrength, damaged },
     crack:            { counts: {a,b,c,d,e}, areaRatioOver20pct },
     carbonation:      { points:[{measured1,2,3}], coverThickness },
     chloride:         { measured },
     rebarCorrosion:   { grades: {a,b,c,d,e}, environment: '건조'|'습윤'|'부식성'|'고부식성' },
     surfaceAging: {                       // 4개 중 최저점수가 대표값
       spalling:      { grades:{a,b,c,d,e}, areaRatio },
       delamination:  { grades:{a,b,c,d,e}, areaRatio },
       leakage:       { grades:{a,b,c,d,e} },
       rebarExposure: { grades:{a,b,c,d,e} },
     }
   }
```

## 3. 화면(탭) 구조 — tilt-survey와 동일한 탭 UI 관례를 따름

1. **기본정보** — basicInfo 입력, 전경사진
2. **표본층 관리** — 표본층 추가/삭제, 층별 구조형식·구조재료
3. **안전성평가** — 표본층 선택 → 부재(5종) 탭 → 소요강도/부재강도 입력
4. **상태평가** — 표본층 선택 → 부재(5종) × 세부항목(6종) 그리드 입력
5. **변위·변형** — 기울기/부동침하 (tilt-survey 결과를 가져오거나 여기서 재입력)
6. **종합결과/보고서** — 등급 계산 결과 + 인쇄용 리포트
   (원본 매뉴얼의 "대상건축물 평가결과 출력" 서식을 그대로 참고해서 표로 구성)

표본층 수 × 부재 5종 × 세부항목 6종이라 입력 항목이 많으므로, 화면은
"표본층 선택 드롭다운 + 부재 탭 + 세부항목 아코디언" 형태로 한 번에 하나씩만
보여주는 구조를 권장 (tilt-survey처럼 building마다 통째로 펼치면 표본층 7개 ×
부재 5개만으로도 화면이 감당 안 됨).

## 4. 등급 산정 엔진 — ⚠️ 현재 가장 큰 미확정 요소

매뉴얼 텍스트에는 화면 사용법만 있고, **등급(A~E) 산정 기준표와 층별/건물
종합등급 계산식(최저값 기준인지, 가중평균인지 등)은 나와있지 않다.**
따라서 이 부분은 `grading-rules.js` 라는 별도 파일로 분리해서, 로직 전체를
갈아끼울 수 있게 만드는 것을 권장한다. (계산 엔진과 UI를 분리해두면
기준표를 나중에 알게 됐을 때 UI를 안 건드리고 이 파일만 교체 가능)

정확한 기준을 확보하는 방법 후보:
- "정밀안전진단 세부지침" 공식 문서(국토안전관리원 발간) 확보
- `SafetyMan_setup.exe`를 실제 설치해서 알려진 입력값 → 결과값으로 역산
- 지금은 매뉴얼에 나온 예제 하나(등급 없이 손상여부만 표기된 표, 1.2.10
  결과표)로는 등급 컷오프 값을 알 수 없음 — 등급 로직 없이 "입력만 되는
  뼈대"부터 만들고 등급 계산은 나중 단계로 미루는 것도 방법

## 5. 파일 구조 제안

```
site-inspection-tools/
├─ service-worker.js          # APP_SHELL 배열에 아래 파일들 전부 추가 필요
├─ shared/                    # 신규: tilt-survey와 공통으로 쓸 유틸 분리
│   ├─ idb-store.js           #   IndexedDB 저장/불러오기 공통 헬퍼
│   ├─ photo-widget.js        #   사진 첨부/라이트박스 (tilt-survey에서 이미 만든 것)
│   └─ print-report.css       #   인쇄 레이아웃 공통 스타일
└─ tools/
   ├─ tilt-survey/index.html
   └─ safety-assessment/
      ├─ index.html           # 탭 UI, 상태 관리, 렌더링
      ├─ grading-rules.js     # 등급 산정 로직만 분리 (위 4번 이유)
      └─ DESIGN.md            # 본 문서
```

tilt-survey는 이미 완성돼서 단일 파일로 잘 동작하니 손대지 않고, 새 도구부터
공통 부분을 `shared/`로 분리해 두면 앞으로 도구가 늘어날 때마다 반복 작성하는
걸 줄일 수 있다. (서비스워커는 파일 하나로 캐시 배열에 없어도 온라인 상태에서
한 번 열면 자동으로 캐시되지만, 오프라인 첫 설치를 위해 배열에 명시하는 게 안전함)

## 6. 진행 순서 제안

1. `basicInfo` + `sampleFloors` CRUD (표본층 추가/삭제) — 뼈대
2. 부재 5종 × 세부항목 6종 입력 폼 (등급 계산 없이 입력·저장만)
3. `grading-rules.js` 자리만 만들어두고 등급 기준 확정되면 채우기
4. 종합결과 리포트 화면 (Excel 서식 참고 인쇄 레이아웃)
5. `tiltSettlement`은 tilt-survey 데이터와 연동할지, 여기서 별도 입력할지 결정 필요

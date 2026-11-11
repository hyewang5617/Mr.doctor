# 구현 현황

> 상태: 코드 정적 검토 · 최종 수정: 2026-11-11 · 작성: Codex

파일 존재와 코드 확인 결과다. 컴파일·플레이·빌드는 수행하지 않았다. Inspector(에디터의 오브젝트 설정 창)의 실제 연결과 동작은 별도 검증해야 한다.

## 환경

- 프로젝트: `PLumproject1/`
- 기록된 Unity 버전: `6000.0.43f1`
- 주요 패키지: URP `17.0.4`, Cinemachine `3.1.4`, Input System `1.13.1`
- Unity CLI 설치 확인. 최신 버전 조회는 네트워크 호스트 조회 실패로 완료하지 못함.
- manifest에 `com.unity.pipeline` 없음. CLI status에서 연결된 인스턴스를 찾지 못했으며 실행 중 에디터 조회 제한 경고도 나옴. 에디터가 꺼져 있다고 단정할 수 없음.

## 기능별 확인

| 기능 | 근거 | 현재 판단 |
| --- | --- | --- |
| 이동·플레이어 유지 | `Player.cs` | 싱글톤으로 씬 사이 플레이어 유지. 물리 컴포넌트가 있으나 이동은 transform 직접 변경 |
| 씬 입구 | `Door.cs`, `SceneEntrance.cs`, `SpawnPoint.cs` | 프롤로그↔1층 복도, 1-1·1-2, 2층 복도 연결 정의 |
| 대화·독백 | `MonolgueManager.cs`, `Talkdata.cs` | 대사 출력·조작 잠금·종료 이벤트 구현. 실행 검증 필요 |
| 아이템 | `Inventory.cs`, `LanternPickup.cs`, `KeyPickup.cs` 등 | 인벤토리·픽업 코드와 프리팹 존재 |
| 저장·불러오기 | `SaveSystem.cs`, `SaveGameLoader.cs` 등 | 위치·인벤토리·플래그 복원 코드 존재. 전체 조합 검증 필요 |
| 일시정지 | `PauseController.cs` | 코드 존재. 퍼즐·몬스터의 시간 처리와 함께 확인 필요 |
| 자물쇠 | `RhythmPuzzleManager.cs` | 기본 noteCount=4, 시작할 때 nextIndex=0, 실패·ESC 시 닫기 |
| 귀신 | `EnemiesController.cs` | 왼쪽 접근과 원위치 리셋. unscaledDeltaTime을 사용해 timeScale=0에서도 이동 |
| 사망 | `Player.cs` | 애니메이션·alive=false. 숫자 0 키로 부활하는 개발용 경로 |
| 타이틀 이어하기 | `Titlecontroller.cs` | OnClickContinue는 미구현 로그. 버튼의 실제 연결은 별도 확인 |
| 엔딩 | 씬 목록·Scripts 검색 | 엔딩 전용 씬·명시적 엔딩 처리 코드를 확인하지 못함 |

## 우선 위험

1. **대사 식별 충돌:** MonologueManager의 `HashSet<int>`는 progress만 기록한다. 다른 TalkData의 같은 번호를 구분하지 않는다. 기존 changelog에도 설명이 혼재하므로 코드 기준으로 판단했다.
2. **기획과 퍼즐 차이:** 중단 유지·한 단계 후퇴·천천히 후퇴하는 몬스터 규칙이 현재 코드와 다르다. noteCount의 실제 씬·프리팹 설정은 기본값과 다를 수 있다.
3. **저장 복원 순서:** SaveGameLoader는 여러 프레임 대기, 랜턴 개별 보정, 전체 MonoBehaviour에 OnLoadGame 메시지를 보내는 방식이다. 씬 이동·같은 씬 로드 모두 검사해야 한다.
4. **이벤트 수명:** SaveGameLoader의 sceneLoaded 람다 구독은 해당 코드에서 해제를 확인하지 못했다. 반복 로드 동작을 점검한다.
5. **상태 초기화:** 사망·대화·퍼즐·새 게임 사이에 정적 조작 잠금과 생존 상태가 정상 초기화되는지 확인한다.
6. **상호작용 중복 가능성:** Player의 공용 Item 픽업 경로는 LanternPickup만 제외한다. KeyPickup 등 자체 획득 처리와 중복 실행되는지는 태그·씬 연결을 확인해야 한다.

## 씬과 빌드

빌드 목록 순서: `intro`, `ingame`, `TITLE`, `1-hall`, `Prologue`, `1-1`, `1-2`, `2-hall`.

`2-1`, `2-2` 파일은 있지만 빌드 목록에 없다. 시작 씬이 `intro`인 것이 의도인지 확인해야 한다. 3층·지하·엔딩 씬은 현재 목록에서 확인하지 못했다. 이름만으로 기존 씬을 삭제하지 않는다.

## 다음 실행 검증

- 프로젝트에 맞는 에디터와 CLI 연결·컴파일 오류 확인
- 실제 빌드 시작 씬에서 타이틀·새 게임 진입
- 프롤로그→1층 방 왕복→열쇠→퍼즐→2층 이동
- 대화 중 이동·다른 상호작용 차단과 종료 후 복구
- 사망·일시정지·퍼즐 종료·새 게임 상태 확인
- 아이템 획득 전후 저장, 다른 씬에서 로드, 과거 슬롯 로드, 반복 로드

관련 원본 코드: `PLumproject1/Assets/Scripts/`, 씬: `PLumproject1/Assets/Scenes/`, 빌드 설정: `PLumproject1/ProjectSettings/EditorBuildSettings.asset`.

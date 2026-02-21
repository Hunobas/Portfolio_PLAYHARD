# 박태훈 | 유니티 게임 클라이언트 프로그래머

> 1998년생, 군필 (의경 만기제대)
>
> 📧 hunobas.dev@gmail.com

> Unity와 C#을 주력으로, 기획자 친화적인 데이터 기반 설계와 아키텍처에 관심이 많습니다.
> 독학으로 3주 만에 뱀서라이크 프로토타입을 빈 템플릿에서 완성한 경험이 있습니다.

---

## 플레이하드 지원동기

플레이하드의 게임들이 이용자의 90%가 해외에서 유입된다는 사실이 인상적이었습니다.
국내 시장의 반응을 검증한 뒤 글로벌로 확장하는 것이 아니라, 처음부터 글로벌 한 우물을 판다는 방향성이 명확했습니다.

저는 3주 만에 뱀서라이크 프로토타입을 혼자 설계하고 완성했습니다.
완성도보다 속도를 택하고, 동작하는 결과물로 가설을 검증하는 방식이 몸에 익어 있습니다.
"완벽을 추구하기보다 빠르게 시도하고 그 과정에서 배움을 얻어 결국 성공을 만들어낸다"는 플레이하드의 방식과 같은 방향을 보고 있다고 느꼈습니다.

5인 팀 프로젝트에서 기획자가 코드 수정 없이 직접 밸런싱할 수 있는 데이터 기반 파이프라인을 구축하고 문서화한 경험이 있습니다.
소규모 팀에서 각자의 권한과 책임이 명확할수록 속도가 나온다는 것을 직접 느꼈고, 그 구조를 만드는 데 기여하는 것에 보람을 느낍니다.

글로벌 시장에서 TOP 10을 목표로 하는 도전에, 그 여정을 끝까지 함께할 사람으로 합류하고 싶습니다.

---

## 핵심 역량

### 1. Unity C# 아키텍처 설계 — FSM 기반 플레이 모드

![GameState 버그 영상](https://github.com/user-attachments/assets/fa973d2f-df58-483d-ae3b-05d5104e9bc6)

**1인창 퍼즐 게임**인 목성의 노래 프로젝트에서 서로 다른 4가지 플레이 모드(일반/패널/시네마/일시정지)가 중첩되어 조작 불가 버그가 반복 발생했습니다.

상태가 여러 파일에 분산되어 디버깅에 매번 평균 1시간이 소요되었고, 중앙 집중식 FSM으로 재설계했습니다.
```csharp
public void ChangePlayMode(IPlayMode next)
{
    if (next == null || ReferenceEquals(_activeMode, next)) return;

    // 시네마 모드는 일시정지 이외의 전환 요청 무시
    if (IsPlayingCinema && !ReferenceEquals(next, PauseMode)) return;

    var prev = _activeMode;
    prev?.OnExit(next);
    _activeMode = next;
    _activeMode.OnEnter(prev);
    InputManager.Instance?.UpdateCursorLock();
}
```

| 개선 항목 | Before | After |
|---------|--------|-------|
| 상태 충돌 버그 | 주 2~3건 | **0건** |
| 디버깅 소요 시간 | 평균 60분 | 평균 30분 |
| 신규 모드 추가 | - | `IPlayMode` 구현만으로 20분 이내 |

📂 [전체 코드 보기 — GameState.cs](https://github.com/Hunobas/Song-Of-Jupitor/blob/7386ab978fc3115a13a700758c7a618567bc168a/Scripts/System/GameState.cs#L15)

---

### 2. 기획자 친화적 사운드 시스템 설계

Unity 기본 AudioSource로는 페이드/크로스페이드/Duck 처리가 번거롭고, 기획자가 Inspector에서 제어할 수 없었습니다.

![preview](https://github.com/user-attachments/assets/43a81247-8927-4e6c-a465-2a5db6d5be0f)

영상 편집 프로그램의 트랙/클립 개념을 적용하여 `SoundSource`(트랙) + `SoundEntry`(클립) 구조로 재설계했습니다.
```csharp
// 프로그래머: 한 줄 체이닝으로 복잡한 사운드 연출 완성
_soundSource.PlayByNameOrNull("GeneratorStartUp")
    .WithFadeIn(0.5f)
    .WithFadeOut(1.0f)
    .WithPriority(SoundPriority.High)
    .WithOptions(PlayOptions.DuckOthers)
    .OnFinish(() => _soundSource.PlayByNameOrNull("GeneratorLoop")
        .WithLoop()
        .WithOptions(PlayOptions.DuckOthers));
```

- 기획자: Inspector에서 기본값(FadeIn 시간, OverlapPolicy 등) 직접 설정
- 프로그래머: Fluent API로 런타임 오버라이드
- Duck 시스템으로 중요 사운드 재생 시 배경음 자동 감쇠

📂 [SoundSource 코드](https://github.com/Hunobas/Song-Of-Jupitor/blob/main/Scripts/Sound/SoundSource.cs) | [SoundEntry 코드](https://github.com/Hunobas/Song-Of-Jupitor/blob/main/Scripts/Sound/SoundEntry.cs) | [SoundManager 코드](https://github.com/Hunobas/Song-Of-Jupitor/blob/main/Scripts/Sound/SoundManager.cs)

---

### 3. 성능 최적화 — 배칭 & GC

#### 3-1. Unity 렌더링 배칭 최적화

<img width="1370" height="814" alt="image" src="https://github.com/user-attachments/assets/b1cb833c-c6ae-4603-bdc4-5901bd7b340f" />

400만 버텍스 + 300개 머터리얼 씬에서 30~60 FPS로 불안정하던 문제를 해결했습니다.

MeshBaker로 방 단위 텍스처 아틀라스 + 콤바인 메쉬, 오클루전 컬링을 적용했습니다.

<img width="1548" height="591" alt="image" src="https://github.com/user-attachments/assets/6b35a453-6a45-4258-9635-3bcff6062e97" />

| 지표 | Before | After |
|------|--------|-------|
| Batches | 2,650 | 601 |
| FPS | 30~60 | **120+** |

📂 [MeshBaker 에디터 확장 코드](https://github.com/Hunobas/Song-Of-Jupitor/blob/main/Scripts/Editor/MB3_ApplyCombinedMaterialToSourceObjects.cs) | [최적화 개발일지](https://velog.io/@po127992/목성의-노래-MeshBaker-최적화-삽질기-텍스처-아틀라스만-vs-콤바인-메쉬까지)

<details>
<summary><b>3-2. ASCII 렌더러 최적화 (CPU 27.6ms → 2.15ms)</b></summary>

<br />

![image (2)](https://github.com/user-attachments/assets/389ec02c-9fdf-4cdd-aa57-0c9e79bbfa4b)

*Unity 에디터에서 실시간 미리보기 가능한 아스키 렌더러*

160×90 그리드 × 4×4 슈퍼샘플 = 230,400회 픽셀 접근으로 CPU 점유 70.4%를 차지하던 문제를 해결했습니다.

**해결:** 비동기 Readback + 색상 변경 구간에만 Rich Text 태그 삽입 + 색상 해상도 다운샘플링

| 지표 | Before | After |
|------|--------|-------|
| CPU 시간 | 27.6ms | 2.15ms |
| 프레임 비중 | 70.4% | 3.5% |

📝 [UPM 플러그인 | GitHub](https://github.com/Hunobas/AsciiImageUGUI-UPM) | [개발일지](https://velog.io/@po127992/목성의-노래-Unity-ASCII-렌더러-공유-및-개발일지)

</details>

---

### 4. 디버깅 & 문제 해결 — My Little Puppy (드림모션 인턴)

NPC 루트모션 및 FSM 관련 버그 3건을 해결했습니다. 공통 패턴은 외부 모듈이 캐릭터 트랜스폼을 오염시키는 구조였고, 이를 바탕으로 트랜스폼 수정 메서드에 대한 코드 컨벤션을 팀에 제안했습니다.

🐶 [드림모션 경력기술서 | Notion](https://ethereal-judo-1f1.notion.site/My-Little-Puppy-1c6486e2cdb980fcbc33f487a01bd7fc)

<details>
<summary><b>사례 1: 루트모션 회전이 FPS에 따라 달라지는 문제</b></summary>

| 문제 상태 | 정상 상태 |
|------|---------|
| ![문제](https://github.com/user-attachments/assets/c038982c-4e66-4c04-a3c1-a4878d3d48c5) | ![정상](https://github.com/user-attachments/assets/f0d4974a-a54d-4b7d-870c-e7b6ad794d5c) |

**원인:** 루트모션 각도 업데이트와 `ActorPosDir` 선형 보간이 중복 적용  
**해결:** 루트모션 전용 위치/각도 업데이트 메서드 분리

</details>

<details>
<summary><b>사례 2: 루트모션 종료 시 캐릭터가 앞으로 튀는 문제</b></summary>

| 문제 상태 | 정상 상태 |
|------|---------|
| ![문제](https://github.com/user-attachments/assets/4e19abda-d7a0-41ea-a84e-c6ffce35ca67) | ![정상](https://github.com/user-attachments/assets/53eb6482-83c2-4f4d-bbe5-2e215c2b7f15) |

**원인:** Walk → RootMotion → Idle 전환 시 속도값이 초기화되지 않고 잔류  
**해결:** 루트모션 종료 시점에 Idle 블렌딩 속도 0 초기화

</details>

<details>
<summary><b>사례 3: 컷씬 일시정지 시 NPC 위치가 튀는 문제</b></summary>

| 문제 상태 | 정상 상태 |
|------|---------|
| ![문제](https://github.com/user-attachments/assets/436f5f08-5091-4b73-bb91-3871885ca447) | ![정상](https://github.com/user-attachments/assets/9bb6594b-afca-43f8-b29a-31fe61e320e3) |

**원인:** 컷씬 일시정지 모드에서 `LateUpdate` 트랜스폼 재조정 처리 누락  
**해결:** 일시정지 진입 전 모드가 컷씬이면 `HandleGameActors()` 호출 추가

</details>

---

### 5. CS 기초 & 알고리즘

크래프톤 정글 6개월 합숙 과정에서 백준 티어 브론즈 3 → 골드 5로 성장했습니다.

[![Solved.ac 뱃지](http://mazassumnida.wtf/api/v2/generate_badge?boj=po1279)](https://solved.ac/po1279)

- [RB트리 직접 구현](https://github.com/Hunobas/rbtree-lab) — 시각화 디버깅 프레임 직접 제작
- [malloc 직접 구현](https://github.com/Hunobas/malloc-lab)
- [PintOS 테스트 137/141 통과](https://github.com/sjlim32/4-5-team5-pintos-project3/tree/Week10/Hunobas) — 멀티스레드 우선순위 기부 로직 구현
- 네트워크: Photon + Firebase 기반 [체스 게임 토이 프로젝트](https://github.com/Hunobas/Chess_App_Unity), 풀스택 웹게임 [THE RATTUS](https://github.com/younggun339/jungleTwo)

---

## 우대사항 관련 경험

| 항목 | 경험 |
|------|------|
| **풀사이클 경험** | 드림모션 인턴 3개월 — My Little Puppy 스팀 데모 출시까지 참여 |
| **라이브러리 구축** | [ASCII Image UGUI UPM 플러그인](https://github.com/Hunobas/AsciiImageUGUI-UPM) 제작 및 배포, 사운드 시스템 모듈 설계 |
| **알고리즘** | BOJ 골드 5, 크래프톤 정글 6개월 집중 수련 |

---

## 프로젝트 요약

| 프로젝트 | 엔진 | 기간 | 규모 | 역할 |
|----------|------|------|------|------|
| [목성의 노래](https://github.com/Hunobas/Song-Of-Jupitor) | Unity | 2025.06~2026.01 | 5명 | FSM 아키텍처, 사운드 시스템, 렌더링 최적화, 퍼즐 로직 |
| [My Little Puppy](https://ethereal-judo-1f1.notion.site/My-Little-Puppy-1c6486e2cdb980fcbc33f487a01bd7fc) | Unity | 2025.01~03 | 38명 | 다국어 폰트 시스템, 슈퍼점프 콘텐츠, 에디터 확장, 버그 수정 |
| [TOGU: Planet Survivors](https://github.com/Hunobas/Planet) | Unreal 5.4 | 2025.04~06 | 1명 (개인) | 전체 아키텍처 설계 및 구현 |

---

# 📞 Contact

- 휴대폰 : 010-3702-1279
- 이메일 : hunobas.dev@gmail.com
- 블로그 : [Velog](https://velog.io/@po127992/posts)
- 깃허브 : [GitHub](https://github.com/hunobas)

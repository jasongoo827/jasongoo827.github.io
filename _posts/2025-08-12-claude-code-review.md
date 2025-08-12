---
title: AI 코딩의 새로운 패러다임, Claude Code 사용 후기
tags: [AI, Claude, Development, Automation, Unity]
style: fill
color: success
description: GPT에서 Claude Code로 갈아탄 개발자의 솔직한 사용 후기. MCP, Vibe Coding, Agent 기능까지 실전 활용 경험을 공유합니다.
---

## 도입: AI 코딩 도구의 진화

최근 AI를 활용한 Vibe Coding, MCP(Model Context Protocol), 자동화 등 새로운 기능들이 매일같이 쏟아지고 있다. 시대의 흐름에 맞춰 AI를 잘 활용하는 개발자가 되어야겠다는 생각이 들었다.

그동안 나는 ChatGPT를 이렇게 사용해왔다:
1. 궁금한 것, 모르는 것 질문
2. 답변을 바탕으로 작업 시작
3. 문답을 이어가며 프로젝트 진행

하지만 GPT가 제시한 코드를 복사해서 IDE에 붙여넣고 테스트하는 과정은 생각보다 번거로웠다. 특히 Unity의 NGO(Netcode for GameObjects)를 공부하면서 GPT의 부정확한 답변과 오류가 많은 코드를 보며 실망했다. "내가 한 달에 3만원씩 내는 도구가 이 정도라니?" 이참에 Claude Code를 사용해보기로 결심했다.

## Claude Code CLI 첫 만남

평소 터미널로 작업하는 것을 좋아하는 나에게 Claude Code의 CLI는 매력적이었다. 큰맘 먹고 월 $20 요금제를 결제했다.

### 첫 번째 실습: 사진 슬라이드쇼 만들기

Mac 사용자로서 컴퓨터에 저장된 사진들로 슬라이드쇼를 만들어달라고 요청했다. Claude Code의 반응은 놀라웠다:

- 필요한 환경 세팅부터 자동으로 구성
- 코드 작성 및 테스트까지 일괄 처리
- **가상환경 세팅을 자동으로 수행**
- 중간에 발생한 오류를 스스로 감지하고 수정

5컷짜리 간단한 슬라이드쇼가 순식간에 완성되었다. 복잡한 작업에도 충분히 활용할 수 있겠다는 확신이 들었다.

<details>
<summary>🎬 Claude Code가 생성한 슬라이드쇼 프로젝트 구조 보기</summary>

```python
# Claude Code가 자동 생성한 슬라이드쇼 코드 일부
class PhotoSlideshowMaker:
    def __init__(self, photos_dir="photos", output_dir="output"):
        self.video_size = (1920, 1080)  # Full HD
        self.fps = 30
        self.photo_duration = 3  # 각 사진 표시 시간 (초)
        
    def get_photos_sorted(self):
        """사진들을 날짜순으로 정렬하여 반환"""
        photo_files = []
        for ext in self.image_extensions:
            photo_files.extend(glob.glob(os.path.join(self.photos_dir, ext)))
        # EXIF 데이터로 날짜순 정렬
        photo_files.sort(key=self.get_photo_date)
        return photo_files
```

**자동 생성된 프로젝트 구조:**
```
ClaudeCLI/
├── photo_slideshow_env/      # 자동 생성된 가상환경
├── photo_slideshow/
│   ├── slideshow_maker.py   # 125줄의 완성된 코드
│   ├── photos/              # 5장의 사진 자동 처리
│   ├── output/
│   │   └── test_slideshow.mp4  # 최종 결과물
│   └── README.md            # 사용법 문서 자동 생성
```

**Claude Code가 작성한 README의 개발 체크리스트:**
```markdown
## 개발 단계
- [x] 기본 환경 설정
- [x] 사진 날짜순 정렬
- [x] 기본 슬라이드쇼 생성
- [ ] 배경음악 추가
- [ ] 전환 효과 추가
- [ ] 최종 테스트
```

</details>

놀라운 점은 단순히 코드만 생성한 것이 아니라:
- **가상환경을 자동으로 설정**하고 필요한 패키지 설치
- **EXIF 메타데이터**를 읽어 사진을 날짜순으로 정렬
- **에러 핸들링**과 사용자 친화적인 로그 출력
- **README 문서**까지 자동으로 작성

모든 것이 한 번의 요청으로 완성되었다.

## MCP(Model Context Protocol) 활용

### Notion MCP 연동

처음에는 커스텀 Notion MCP를 만들려고 `settings.json`을 조작하고 스크립트까지 작성했지만, 생각대로 동작하지 않았다. 결국 공식 Notion MCP를 사용하니 API 키만 등록하면 **5분 만에** 내 Notion 페이지를 Claude가 읽어와 작업을 수행할 수 있었다.

![Notion MCP 설정](/assets/notionmcp.png)
<sub>***settings.json에 Notion MCP 설정 추가 - API 키만 있으면 바로 연동 가능***</sub>

![Notion 페이지 읽기](/assets/notion.png)
<sub>***Claude가 내 Notion 페이지를 읽고 작업 수행 - 개발 일정과 문서를 자동으로 파악***</sub>

**활용 아이디어**: Notion으로 개발 일정을 관리한다면 이런 자동화 파이프라인을 구축할 수 있을 것 같다:
```
Notion Todo 작성 → 개발환경 세팅 → 작업 수행 → Notion에 결과 반영
```

실제로 Claude는 내 Notion 페이지의 개발 일정을 읽고, 해당 작업을 자동으로 수행한 뒤, 완료 상태를 다시 Notion에 업데이트할 수 있었다. 프로젝트 관리와 개발을 통합하는 강력한 워크플로우다.

## Unity Vibe Coding 실전

### Subagent 기능 발견

Claude 공식 문서에서 Subagent 기능을 발견하고 Unity 프로젝트에 적용해보기로 했다.

#### 첫 시도: Project Setting Agent

`/agents` 명령으로 프로젝트 세팅 Agent를 만들었다. 짧은 프롬프트로 2D 플랫포머 게임 환경을 구축하려 했지만, 예상과 달리 Agent가 **전체 플랫포머 게임 코드를 생성**해버렸다.

**교훈**: Agent에게는 제한적이고 명확한 역할을 부여해야 한다.

#### 두 번째 시도: Game Development Agent

이번에는 다른 접근을 시도했다:

1. Claude에게 게임 흐름을 **아주 자세하게** 설명
2. 각 기능의 동작 방식을 구체적으로 명시
3. "프로그래밍 도움만" 주도록 역할 제한
4. 단계별 기능 구현 요청

결과는 훨씬 나아졌다. 원하는 기능을 정확히 구현했고, 테스트 방법까지 단계별로 안내받아 **3일 만에** 게임의 핵심 기능을 완성할 수 있었다.

<details>
<summary>📝 실제 사용한 Game Development Agent 프롬프트 보기</summary>

```yaml
name: game-development-agent
description: Unity Mobile Game - My Sweet Bakery
model: sonnet
---

# Unity My Sweet Bakery Game Development Agent

## Project Overview
Develop a "My Sweet Bakery" business simulation game using Unity 2022.3.19f. 
The player operates a bakery, producing and selling bread while expanding their 
store in a 3D isometric style mobile game.

## Core Game Mechanics

### 1. Player Movement System
- Virtual joystick/pad in center of screen (mobile)
- Automatic task execution when player stands in specific areas
- Movement, idle, and work state animations

### 2. Production System
- Bread Machine: Produces 1 bread per second, maximum 8 stored
- Player Inventory: Can hold maximum 8 bread items
- Display Case: Can display maximum 8 bread items

### 3. Customer System
- Customer Spawning: Wait in front of display case
- Order Display: Speech bubbles showing desired bread quantity
- Customer Types: Takeout (7 coins) vs Dine-in (10 coins)

### 4. Cashier System
- Queue Processing: Handle customers in order
- Money Display: Money stacks in 3×3 grid
- Collection: Player stands on money to collect automatically

## Technical Architecture
GameManager (Singleton)
├── PlayerController
├── ProductionManager
├── CustomerManager
├── EconomyManager
├── UIManager
└── InteriorManager

## Development Priority
Phase 1 (MVP): Basic movement, production, display, customer AI
Phase 2: Money collection, interior purchase, table service
Phase 3: Animations, sound, balancing, optimization

## Key Considerations
- 3D Isometric View with proper camera settings
- Mobile Optimization for performance
- Intuitive Touch UI
- Immediate feedback for all interactions
```

</details>

이 프롬프트를 통해 Claude Code는 정확히 내가 원하는 범위 내에서 코드를 생성했고, 게임의 핵심 시스템을 체계적으로 구현할 수 있었다.

### 실제 개발 성과

<details>
<summary>📊 3일간의 Unity 게임 개발 통계</summary>

**프로젝트 구조:**
```
BakeryGame/
├── Scripts/             # 30개의 C# 스크립트
│   ├── Animation/       # 애니메이션 헬퍼
│   ├── Config/          # 게임 설정 관리
│   ├── Controllers/     # 플레이어, 인벤토리, 생산 시스템
│   ├── Customers/       # AI 고객 시스템
│   ├── Economy/         # 경제 시스템
│   ├── Interactions/    # 상호작용 시스템
│   ├── Interior/        # 확장 시스템
│   ├── Managers/        # 게임 매니저
│   ├── UI/             # UI 시스템 (조이스틱, HUD, 말풍선)
│   └── Utilities/       # 유틸리티 (패스파인딩 등)
```

**개발 성과 메트릭:**
```yaml
총 코드 라인수: 16,314줄
생성된 스크립트: 30개
구현된 핵심 시스템: 9개

시스템별 구현 현황:
✅ Player Movement (Virtual Joystick)
✅ Production System (자동 생산)
✅ Inventory Management (8개 제한)
✅ Customer AI (주문, 이동, 결제)
✅ Economy System (코인 수집)
✅ Display System (빵 진열)
✅ Cashier System (결제 처리)
✅ Interior Expansion (가구 구매)
✅ UI/UX (HUD, 말풍선, 튜토리얼)

자동화된 작업:
- NavMesh 설정 및 경로 찾기
- 애니메이션 컨트롤러 연결
- 이벤트 시스템 구축
- 오브젝트 풀링 최적화
```

**시스템 아키텍처 플로우:**
```
[GameManager]
    ├─→ [PlayerController] ←→ [VirtualJoystick]
    │       ↓
    │   [PlayerInventory]
    │       ↓
    ├─→ [ProductionSystem] → [BreadMachine]
    │       ↓
    │   [DisplayCase]
    │       ↓
    ├─→ [CustomerManager] → [Customer AI]
    │       ↓
    │   [CashierSystem] → [Payment]
    │       ↓
    ├─→ [EconomyManager] → [MoneyDisplay]
    │       ↓
    └─→ [UIManager] → [SpeechBubble, HUD, Tutorial]
```

</details>

Claude Code의 가장 인상적인 점은 **단순히 코드를 생성하는 것이 아니라**, 전체 시스템 간의 연결과 상호작용까지 고려해서 구현했다는 것이다. 특히 Customer AI가 DisplayCase의 상태를 확인하고, CashierSystem과 연동되며, MoneyManager를 통해 경제 시스템과 통합되는 과정이 자연스럽게 구현되었다.

## 사용하면서 발견한 한계와 팁

### 발견한 문제점들

1. **코드 복잡도와 정확도의 반비례**
   - 코드가 복잡해질수록 적중률이 떨어짐
   - 초기에는 정확했던 코드가 점점 오류 증가

2. **지시사항의 명확성이 핵심**
   - ❌ "이 부분에 문제가 있어. 고쳐줄래?"
   - ✅ "PlayerController.cs와 CharacterController.cs의 기능이 중복됨. PlayerController만 남기고 삭제해줘."

3. **토큰 소비량**
   - 예상보다 많은 토큰 필요
   - $100 요금제로 업그레이드 (Sonnet 모델은 거의 무제한)

4. **일반적인 오류 패턴**
   - 삭제된 변수에 접근
   - 기능 중복 구현 (이름만 살짝 다르게)
   - private 변수 접근 시도
   - 이벤트 사용 방식의 일관성 부족

### 생산성 향상을 위한 제안

코드가 길어지면서 시행착오가 늘어나 오히려 생산성이 떨어질 수 있다. 현재 경험을 바탕으로 한 개선 방안:

1. **Agent 역할을 명확하게 제한**
   ```yaml
   game-setup-agent:
     역할: "Unity 프로젝트 초기 설정만"
     제한: "코드 생성 금지, 설정 파일만 수정"
   
   feature-development-agent:
     역할: "단일 기능 구현"
     제한: "한 번에 하나의 시스템만, 최대 5개 스크립트"
   
   debugging-agent:
     역할: "오류 수정 전담"
     제한: "새 기능 추가 금지, 버그 수정만"
   ```

2. **단계별 작업 분할**
   - 복잡한 요청을 작은 단위로 나누어 요청
   - 각 단계마다 컴파일 확인 후 다음 단계 진행
   - Agent에게 "이전 단계 완료 확인 후 진행" 명시

3. **명확한 지시사항 작성법**
   ```
   ❌ "이 부분에 문제가 있어. 고쳐줄래?"
   ✅ "PlayerController.cs 47번째 줄의 NullReferenceException을 
       null 체크 추가로 해결해줘"
   
   ❌ "게임 시스템을 만들어줘"
   ✅ "Player가 아이템을 주울 수 있는 상호작용 시스템만 구현해줘.
       범위: PlayerController에 OnTriggerEnter 추가"
   ```

4. **프로젝트 컨텍스트 관리**
   - CLAUDE.md 파일에 프로젝트 규칙과 구조 명시
   - 코딩 컨벤션과 네이밍 규칙 사전 정의
   - 각 Agent 사용 전 컨텍스트 파일 참조 요청

**현실적인 워크플로우:**
```
1. 기능 요구사항 정리 → CLAUDE.md 업데이트
2. 단일 기능 Agent로 코어 로직 구현
3. 컴파일 및 테스트
4. 디버깅 Agent로 오류 수정
5. 다음 기능으로 반복
```

이렇게 하면 현재 Claude Code 기능 범위 내에서도 생산성을 크게 향상시킬 수 있다.

## 결론: AI 코딩 도구의 미래

Claude Code는 확실히 이전보다 빠른 개발을 가능하게 했다. 특히:

- **자동 환경 세팅**과 **오류 자동 수정**이 인상적
- MCP를 통한 **외부 도구 연동**이 강력함
- Agent 기능으로 **복잡한 작업 자동화** 가능

하지만 완벽한 도구는 아니다. AI를 잘 활용하려면:
- 명확하고 구체적인 지시사항 작성 능력
- Agent와 파이프라인 설계 능력
- AI의 한계를 이해하고 보완하는 능력

이 필요하다.

최근 GPT-5도 출시되었다고 하니, 다양한 AI 도구들을 계속 테스트하며 최적의 워크플로우를 찾아갈 예정이다. AI는 도구일 뿐, 결국 이를 얼마나 잘 활용하느냐가 개발자의 경쟁력이 될 것 같다.

## 마무리

AI 코딩 도구는 계속 진화하고 있다. Claude Code, GitHub Copilot, Cursor 등 다양한 도구들이 경쟁하며 발전하고 있다. 중요한 것은 **맹목적으로 의존하지 않고, 도구의 장단점을 파악해 적재적소에 활용**하는 것이다.

앞으로도 새로운 AI 기능들을 실험하며 생산성을 높이고, 더 창의적인 작업에 집중할 수 있는 환경을 만들어갈 계획이다. AI와 함께 성장하는 개발자가 되어야겠다.
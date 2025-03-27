---
title: Physics Engine
tags: [Game Engine, Physics]
style: border
color: success
description: ALEngine의 Physics Engine에 대한 소개글.
---

## Physics Engine
게임을 개발하다 보면, 화면 속의 캐릭터가 땅 위에 서 있고, 총알이 적에게 명중하며, 자동차가 벽에 부딪히는 등의 장면을 자주 보게 된다. 이런 장면들은 마치 현실처럼 자연스럽게 보이지만, 사실 그 뒤에는 수많은 수학적 연산과 논리가 숨어 있다. 바로 **물리 엔진(Physics Engine)**이 그 역할을 맡고 있으며, 게임 속 물리 법칙을 시뮬레이션하는 핵심적인 요소다.

물리 엔진은 크게 두 가지 중요한 역할을 수행한다.

충돌 감지 (Collision Detection): 객체들이 서로 부딪히는지를 판별한다.

물리 시뮬레이션 (Physics Simulation): 충돌이 발생했을 때 반응을 계산한다.

이 중에서도 충돌 감지는 특히 계산량이 많아 성능 최적화가 필수적인 영역이다. 예를 들어, 100개의 오브젝트가 있는 게임에서는 이론적으로 모든 오브젝트가 서로 충돌할 가능성이 있기 때문에, 순진한 방식으로 충돌을 검사하면 **O(n²)**의 연산량이 필요하다. 하지만 이런 방식으로는 현대적인 대규모 게임에서 실시간 처리가 불가능하다.

---

## Flow
물리 엔진의 흐름은 다음과 같다.

#### 1. Integrate
외부에서 가해진 힘-토크 를 계산해 물체의 가속도-각가속도, 속도-각속도, 위치-회전 을 계산한다.

#### 2. Broadphase
물체의 Bounding Volume을 계산해 Volume끼리 서로 충돌하는지 검사한다.

#### 3. Narrowphase
Broadphase에서 충돌한 물체의 쌍들이 실제로 충돌하는지 검사한다.

#### 4. Solve
Narrowphase에서 생성된 충돌 정보를 바탕으로 충돌을 해소한다.


내가 물리 엔진에서 구현한 부분은 Interate ~ Broadphase 단계이다. Integrate 단계는 뉴턴의 제2법칙을 바탕으로 식을 계산하는 단계이기 때문에 따로 설명은 하지 않겠다. Broadphase 단계부터 차근차근 설명해보겠다. 

---

## Broadphase의 개념

Broadphase 단계는 실제로 물체가 충돌하는지 확인하는 단계가 아니라, 빠르게 충돌 가능성이 있는 물체들을 선별하는 단계이다. 처음부터 물체가 충돌하는지 확인하는 게 단계를 더 줄일 수 있으니 빠르지 않냐라고 생각할 수 있지만, 충돌 확인 연산은 비용이 매우매우 비싸기 때문에 미리 충돌 쌍을 선별하는 게 더 좋다. 충돌 가능성이 있는 물체를 선별하기 위해, 먼저 물체를 대략적으로 판단할 수 있는 Bounding Volume을 씌운다.

### Bounding Volume
Bounding Volume은 오브젝트를 감싸는 단순한 형태의 가짜 외곽선이라고 생각하면 이해하기 쉽다. 복잡한 메시(Mesh)나 다각형을 정밀하게 검사하는 대신, 그걸 감싸는 단순한 기하 도형을 사용해서 "겹칠 가능성이 있는지"를 빠르게 판단하는 것이다. Bounding Volume을 만드는 방식은 여러가지가 있다.

![Bounding Volume](https://docs.unity3d.com/Packages/com.unity.physics@1.2/manual/images/bounding-volumes.png)

#### AABB(Axis-Aligned Bounding Box)
- 각 축(x,y,z)에 대해 정렬된(회전하지 않은) 사각형/박스
- 구현이 간단하고 비교 연산이 빠름
- 회전하는 물체를 감싸면 볼륨이 불필요하게 커질 수 있음

#### OBB(Oriented Bounding Box)
- 객체의 방향에 맞춰 회전된 경계 박스
- AABB보다 정확하지만 계산이 복잡하고 비용이 높음

#### Bounding Sphere
- 중심점과 반지름만 있으면 되기 때문에 계산이 매우 단순
- 대칭적인 물체에는 효과적, 하지만 길쭉한 물체에는 비효율적

---

## BVH(Bounding Volume Hierarchy)
Bounding Volume을 정했다면, 이를 가지고 계층 구조 트리를 만들게 된다. 그 전에 왜 트리를 만들어야 하는지 이해하고 가야 한다. Bounding Volume을 만들었다고 해도, 그 정보를 선형적으로 저장하면 Volume끼리 충돌하는지 알아내는 데 걸리는 시간복잡도가 **O(n²)** 이다. 

시간복잡도를 줄이기 위해 고안된 것이 이진 트리의 활용이다. 이진 트리를 사용하면 트리의 절반씩 후보를 제거할 수 있다. 이렇게 설명하면 너무 추상적이니 좀 더 와닿게 예시로 설명을 해보겠다. 

---

### 🧠 BVH를 봉식이와 원훈이의 대결로 이해해보자

BVH(Bounding Volume Hierarchy)를 처음 보면 "왜 굳이 계층적으로 묶지?"라는 의문이 들 수 있다.  
그걸 좀 더 쉽게 이해하기 위해, 우리 친구 **봉식이**와 **원훈이**의 **대결 스토리**를 들어보자.  

---

#### 🎮 설정: 전국에 퍼진 파이터들

- 봉식이와 원훈이는 전국 최고의 싸움꾼이다.  
- 그런데 두 사람은 전국을 돌아다니며, **자신과 대결할 상대를 찾고** 있다.  
- 하지만 모든 사람과 일일이 싸워볼 수는 없다. 효율적으로 **"나랑 싸울 가능성이 있는 사람만 고르고 싶다!"**

봉식이와 원훈이는 이렇게 생각한다:

> “서울, 부산, 대전, 대구, 광주, 제주… 먼저 **동네 단위로 사람들을 묶자!**  
> 나랑 같은 동네 사람만 싸울 가능성이 있으니까!”

그래서 전국의 파이터들을 **지역 단위**로 나눈다(이진 트리 구조인 것을 잠시 놔두고 생각하자).

```text
[대한민국]
├── [서울] ─ 파이터 A, 파이터 B
├── [부산] ─ 파이터 C
├── [대전] ─ 봉식이
├── [광주] ─ 원훈이
```

이 구조는 바로 **BVH의 상위 노드** 역할을 한다.  
여기서 봉식이는 "나는 대전에 있으니까, 광주 사람(원훈이)이랑은 아예 비교 안 해도 돼"라고 판단할 수 있다. 다른 동네를 아예 배제해버릴 수 있는 것이다! 


하지만 만약 봉식이와 원훈이가 **같은 동네**에 살고 있다면?

```text
[대전]
├── 봉식이
├── 원훈이
```

그렇다면 서로 싸울 가능성이 있는 것이다!!

---

다른 Volume들을 배제하며, 충돌쌍을 빠른 시간 안에 찾을 수 있기 때문에 BVH를 쓰는 것이 좋은 것이다. BVH를 표현한다면 아래의 이미지와 같다. 


![BVH](https://www.researchgate.net/publication/362859119/figure/fig1/AS:11431281080279198@1661224948923/Bounding-volume-hierarchies-structure.ppm)


---

## 구현
BVH에 필요한 기능은 노드 삽입, 삭제, 트리 회전이다. 회전은 왜 필요한지 의문이 들 수 있다. BVH의 이점을 극대화하려면 트리가 늘 균형을 유지할 수 있게 관리해야 한다. 트리가 불균형하다면 절반씩 탐색 범위를 줄일 수 없어지기 때문이다. 따라서 균형을 맞추기 위해 회전을 구현한다. 다음은 트리의 구현에 관한 코드이다.


#### TreeNode

```
struct TreeNode
{
	/**
	 * @brief 노드가 리프(Leaf)인지 확인합니다.
	 * @return 리프 노드이면 true, 그렇지 않으면 false.
	 */
	bool isLeaf() const
	{
		return child1 == nullNode;
	}

	AABB aabb;
	void *userData;

	union {
		int32_t parent;
		int32_t next;
	};
	int32_t child1;
	int32_t child2;
	int32_t height;
};

```

#### DynamicTree

다음은 트리이다. proxy라는 말이 생소한데, 그냥 노드라고 생각하면 된다.
노드를 vector로 관리하는데, 메모리 풀로 관리한다면 메모리 할당에 쓰는 cpu를 최적화할 수 있다. 

```

/**
 * @class DynamicTree
 * @brief 동적 트리 기반 충돌 감지를 수행하는 클래스.
 * @details DynamicTree는 AABB 트리를 관리하여 빠른 충돌 탐색을 지원합니다.
 */
class DynamicTree
{
  public:
	/**
	 * @brief DynamicTree 기본 생성자.
	 */
	DynamicTree();

	/**
	 * @brief DynamicTree 소멸자.
	 */
	~DynamicTree();

	/**
	 * @brief AABB와 사용자 데이터를 기반으로 새로운 Proxy(노드)를 생성합니다.
	 * @param aabb 삽입할 AABB.
	 * @param userData 사용자 데이터 포인터.
	 * @return 생성된 Proxy ID.
	 */
	int32_t createProxy(const AABB &aabb, void *userData);

	/**
	 * @brief Proxy ID에 해당하는 노드를 제거합니다.
	 * @param proxyId 제거할 Proxy ID.
	 */
	void destroyProxy(int32_t proxyId);

	/**
	 * @brief Proxy를 새로운 위치로 이동하고 트리를 재구성합니다.
	 * @param proxyId 이동할 Proxy ID.
	 * @param aabb 새로운 AABB.
	 * @param displacement 이동 벡터.
	 * @return 이동 성공 여부 (true: 성공, false: 실패).
	 */
	bool moveProxy(int32_t proxyId, const AABB &aabb, const alglm::vec3 &displacement);

	/**
	 * @brief Proxy ID에 해당하는 사용자 데이터를 반환합니다.
	 * @param proxyId 조회할 Proxy ID.
	 * @return 사용자 데이터 포인터.
	 */
	void *getUserData(int32_t proxyId) const;

	/**
	 * @brief Proxy ID에 해당하는 AABB를 반환합니다.
	 * @param proxyId 조회할 Proxy ID.
	 * @return 해당 Proxy의 AABB.
	 */
	const AABB &getFatAABB(int32_t proxyId) const;

	/**
	 * @brief AABB와 겹치는 노드를 탐색합니다.
	 * @tparam T 콜백 함수 타입.
	 * @param callback 검색된 노드를 처리할 콜백 함수.
	 * @param aabb 검색할 AABB 영역.
	 */
	template <typename T> void query(T *callback, const AABB &aabb) const;

  private:
	/**
	 * @brief 새로운 노드를 할당합니다.
	 * @return 할당된 노드 ID.
	 */
	int32_t allocateNode();

	/**
	 * @brief 특정 노드를 해제합니다.
	 * @param nodeId 해제할 노드 ID.
	 */
	void freeNode(int32_t nodeId);

	/**
	 * @brief 노드를 트리에 삽입하고 균형을 조정합니다.
	 * @param leaf 삽입할 리프 노드 ID.
	 */
	void insertLeaf(int32_t leaf);

	/**
	 * @brief 노드를 트리에서 제거합니다.
	 * @param leaf 제거할 리프 노드 ID.
	 */
	void removeLeaf(int32_t leaf);

	/**
	 * @brief 트리 균형을 조정합니다.
	 * @param index 균형을 조정할 노드 인덱스.
	 * @return 균형이 맞춰진 노드 인덱스.
	 */
	int32_t balance(int32_t index);

	/**
	 * @brief 특정 리프 노드의 삽입 비용을 계산합니다.
	 * @param leafAABB 삽입할 AABB.
	 * @param child 대상 노드.
	 * @param inheritedCost 상속된 비용.
	 * @return 삽입 비용.
	 */
	float getInsertionCostForLeaf(const AABB &leafAABB, int32_t child, float inheritedCost);

	/**
	 * @brief 특정 노드의 삽입 비용을 계산합니다.
	 * @param leafAABB 삽입할 AABB.
	 * @param child 대상 노드.
	 * @param inheritedCost 상속된 비용.
	 * @return 삽입 비용.
	 */
	float getInsertionCost(const AABB &leafAABB, int32_t child, float inheritedCost);

	/**
	 * @brief 동적 트리의 구조를 출력합니다.
	 * @param node 출력할 노드의 인덱스.
	 */
	void printDynamicTree(int32_t node);

	// // tree의 height 계산
	// int32_t ComputeHeight() const;
	// // sub-tree의 height 계산
	// int32_t ComputeHeight(int32_t nodeId) const;

	int32_t m_root;
	int32_t m_freeNode;
	int32_t m_nodeCount;
	int32_t m_nodeCapacity;
	std::vector<TreeNode> m_nodes;
};

```

#### Bounding Volume Collision Detection
인자로 검사하고자 하는 Volume을 넣어주고, 트리의 루트 노드부터 순회하며 단말 노드까지 검사한다. stack을 사용하는 것이 특징이다. testOverlap 함수는 두 AABB가 겹치는지 확인하는 함수이다.


```

template <typename T> inline void DynamicTree::query(T *callback, const AABB &aabb) const
{
	std::stack<int32_t> stack;
	stack.push(m_root);

	while (!stack.empty())
	{
		int32_t nodeId = stack.top();
		stack.pop();
		if (nodeId == nullNode)
		{
			continue;
		}

		const TreeNode node = m_nodes[nodeId];
		if (testOverlap(node.aabb, aabb))
		{
			if (node.isLeaf())
			{
				bool proceed = callback->queryCallback(nodeId);
				if (proceed == false)
				{
					return;
				}
			}
			else
			{
				stack.push(node.child1);
				stack.push(node.child2);
			}
		}
	}
}

bool BroadPhase::queryCallback(int32_t proxyId)
{
	if (proxyId == m_queryProxyId)
	{
		return true;
	}

	m_proxySet.insert({std::min(proxyId, m_queryProxyId), std::max(proxyId, m_queryProxyId)});
	return true;
}

```

글을 작성하면서 queryCallback 함수를 다시 살펴보니 그냥 void로 선언하는 게 적합했을 것이다. bool로 선언한 이유는 box2d의 영향이 클 것이다.

나머지 부분들에 대한 설명은 너무 방대하기 때문에 여기서 끝내겠다. 나중에 코드만 따로 설명할 다른 포스트를 작성할 수도 있을 것 같다. 밑의 참고 자료, 특히 ErinCatto의 pdf를 읽어보는 게 트리에서 노드의 삽입, 삭제, 회전 과정 이해에 큰 도움이 될 것이다.

---

## 시행착오
물리 엔진을 구현하기 위해 처음 공부한 것은 'Game Physics Engine Development' 라는 책이다. 지금은 팔지 않는 책인데, 강남구에 있는 도서관 중에 딱 한 권 있길래 운 좋게 빌려올 수 있었다. 책을 읽으며 물리 엔진의 큰 흐름에 대해 감을 잡을 수 있었다. 하지만 번역 issue, 너무 추상적인 설명 등으로 인해 책만으로 코드를 작성하기에는 무리였다. 그래서 구글을 열심히 찾아보다가 게임 엔진을 구현한 어떤 분의 블로그를 우연히 발견했다. 물리 엔진 뿐만 아니라 흥미로운 내용이 많았는데, 여기서 box2d라는 물리 엔진의 존재에 대해 처음 알게 됐다. 또 정말 다행히도 소스코드가 github에 있어서 학습을 하는데 큰 도움이 됐다. 

Broadphase까지 구현하는 데는 box2d의 도움이 컸지만, Narrowphase를 구현할 때 같이 구현한 동료가 box2d의 코드를 이해하는 데 어려움을 겪어 많이 해맸다. 그래서 다른 상용 물리엔진의 소스코드를 분석하며 학습을 했다. 어려움이 많았지만, 그럼에도 불구하고 끝까지 내용을 공부해 구현한 **seonjo** 에게 박수를 보낸다.

---

## 결과
물리 엔진의 성능을 확인할 수 있는 데모 영상이다.

#### ALEPhysics
[![ALEPhysics](https://i9.ytimg.com/vi/oJnp3A-QEsE/mqdefault.jpg?sqp=CLSxlb8G-oaymwEmCMACELQB8quKqQMa8AEB-AH-CYAC0AWKAgwIABABGF4gXiheMA8=&rs=AOn4CLCTFZ79uT29xdBXCAismSskUAKzCw)](https://youtu.be/oJnp3A-QEsE)


#### ALEPhysics Demo
[![ALEPhysics Demo](https://i9.ytimg.com/vi/_ZHyU_WvnIw/mqdefault.jpg?sqp=CLSxlb8G-oaymwEmCMACELQB8quKqQMa8AEB-AH-CYAC0AWKAgwIABABGGUgXChbMA8=&rs=AOn4CLDgmvOB6xlRh_DfpiWxcD_Z3H7oHQ)](https://youtu.be/_ZHyU_WvnIw)

---

## 개인적인 소감
개인적으로 아쉬웠던 점은 Narrowphase 파트도 내가 구현해 봤으면 좋았을 것 같다는 점, 단순한 모양이 아닌 복잡한 mesh를 가진 도형을 다뤄보지 못한 것, 물리 엔진 루프를 fixed duration으로 구현 못한 것과 렌더링 엔진과 다른 쓰레드로 나누지 못한 점 등등 너무나도 많다. 나중에 시간이 허락된다면, 아쉬웠던 점들을 처음부터 설계에 포함해 구현하면 좋을 것 같다. 


## 참고 자료

[Game Physics Engine Development](https://product.kyobobook.co.kr/detail/S000001535514)

[box2d](https://github.com/erincatto/box2d)

[box2d.js](https://github.com/kripken/box2d.js/)

[Dynamic Bounding Volume Hierarchies](https://box2d.org/files/ErinCatto_DynamicBVH_GDC2019.pdf)

[chanhaeng blog](https://chanhaeng.blogspot.com/)
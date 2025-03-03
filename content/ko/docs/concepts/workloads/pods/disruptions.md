---
# reviewers:
# - erictune
# - foxish
# - davidopp
title: 중단(disruption)
content_type: concept
weight: 60
---

<!-- overview -->
이 가이드는 고가용성 애플리케이션을 구성하려는 소유자와
파드에서 발생하는 장애 유형을 이해하기
원하는 애플리케이션 소유자를 위한 것이다.

또한 클러스터의 업그레이드와 오토스케일링과 같은
클러스터의 자동화 작업을 하려는 관리자를 위한 것이다.

<!-- body -->

## 자발적 중단과 비자발적 중단

파드는 누군가(사람 또는 컨트롤러)가 파괴하거나
불가피한 하드웨어 오류 또는 시스템 소프트웨어 오류가 아니면 사라지지 않는다.

우리는 이런 불가피한 상황을 애플리케이션의 *비자발적 중단* 으로 부른다.
예시:

- 노드를 지원하는 물리 머신의 하드웨어 오류
- 클러스터 관리자의 실수로 VM(인스턴스) 삭제
- 클라우드 공급자 또는 하이퍼바이저의 오류로 인한 VM 장애
- 커널 패닉
- 클러스터 네트워크 파티션의 발생으로 클러스터에서 노드가 사라짐
- 노드의 [리소스 부족](/ko/docs/concepts/scheduling-eviction/node-pressure-eviction/)으로 파드가 축출됨

리소스 부족을 제외한 나머지 조건은 대부분의 사용자가 익숙할 것이다.
왜냐하면
그 조건은 쿠버네티스에 국한되지 않기 때문이다.

우리는 다른 상황을 *자발적인 중단* 으로 부른다.
여기에는 애플리케이션 소유자의 작업과 클러스터 관리자의 작업이 모두 포함된다.
다음은 대표적인 애플리케이션 소유자의 작업이다.

- 디플로이먼트 제거 또는 다른 파드를 관리하는 컨트롤러의 제거
- 재시작을 유발하는 디플로이먼트의 파드 템플릿 업데이트
- 파드를 직접 삭제(예: 우연히)

클러스터 관리자의 작업은 다음을 포함한다.

- 복구 또는 업그레이드를 위한 [노드 드레이닝](/docs/tasks/administer-cluster/safely-drain-node/).
- 클러스터의 스케일 축소를 위한
  노드 드레이닝([클러스터 오토스케일링](https://github.com/kubernetes/autoscaler/#readme)에 대해 알아보기
  ).
- 노드에 다른 무언가를 추가하기 위해 파드를 제거.

위 작업은 클러스터 관리자가 직접 수행하거나 자동화를 통해 수행하며,
클러스터 호스팅 공급자에 의해서도 수행된다.

클러스터에 자발적인 중단을 일으킬 수 있는 어떤 원인이 있는지
클러스터 관리자에게 문의하거나 클라우드 공급자에게 문의하고, 배포 문서를 참조해서 확인해야 한다.
만약 자발적인 중단을 일으킬 수 있는 원인이 없다면 Pod Disruption Budget의 생성을 넘길 수 있다.

{{< caution >}}
모든 자발적인 중단이 Pod Disruption Budget에 연관되는 것은 아니다.
예를 들어 디플로이먼트 또는 파드의 삭제는 Pod Disruption Budget을 무시한다.
{{< /caution >}}

## 중단 다루기

비자발적인 중단으로 인한 영향을 경감하기 위한 몇 가지 방법은 다음과 같다.

- 파드가 필요로 하는 [리소스를 요청](/ko/docs/tasks/configure-pod-container/assign-memory-resource/)하는지 확인한다.
- 고가용성이 필요한 경우 애플리케이션을 복제한다.
  (복제된 [스테이트리스](/ko/docs/tasks/run-application/run-stateless-application-deployment/) 및
  [스테이트풀](/docs/tasks/run-application/run-replicated-stateful-application/) 애플리케이션에 대해 알아보기.)
- 복제된 애플리케이션의 구동 시 훨씬 더 높은 가용성을 위해 랙 전체
  ([안티-어피니티](/ko/docs/concepts/scheduling-eviction/assign-pod-node/#파드간-어피니티와-안티-어피니티) 이용)
  또는 영역 간
  ([다중 영역 클러스터](/ko/docs/setup/best-practices/multiple-zones/)를 이용한다면)에
  애플리케이션을 분산해야 한다.

자발적 중단의 빈도는 다양하다. 기본적인 쿠버네티스 클러스터에서는 자동화된 자발적 중단은 발생하지 않는다(사용자가 지시한 자발적 중단만 발생한다).
그러나 클러스터 관리자 또는 호스팅 공급자가 자발적 중단이 발생할 수 있는 일부 부가 서비스를 운영할 수 있다.
예를 들어 노드 소프트웨어의 업데이트를 출시하는 경우 자발적 중단이 발생할 수 있다.
또한 클러스터(노드) 오토스케일링의 일부 구현에서는
단편화를 제거하고 노드의 효율을 높이는 과정에서 자발적 중단을 야기할 수 있다.
클러스터 관리자 또는 호스팅 공급자는
예측 가능한 자발적 중단 수준에 대해 문서화해야 한다.
파드 스펙 안에 [프라이어리티클래스 사용하기](/ko/docs/concepts/scheduling-eviction/pod-priority-preemption/)와 같은 특정 환경설정 옵션
또한 자발적(+ 비자발적) 중단을 유발할 수 있다.


## 파드 disruption budgets

{{< feature-state for_k8s_version="v1.21" state="stable" >}}

쿠버네티스는 자발적인 중단이 자주 발생하는 경우에도 고 가용성 애플리케이션을
실행하는 데 도움이 되는 기능을 제공한다.

애플리케이션 소유자로써, 사용자는 각 애플리케이션에 대해 PodDisruptionBudget(PDB)을 만들 수 있다.
PDB는 자발적 중단으로
일시에 중지되는 복제된 애플리케이션 파드의 수를 제한한다.
예를 들어, 정족수 기반의 애플리케이션이
실행 중인 레플리카의 수가 정족수 이하로 떨어지지 않도록 한다.
웹 프런트 엔드는 부하를 처리하는 레플리카의 수가
일정 비율 이하로 떨어지지 않도록 보장할 수 있다.

클러스터 관리자와 호스팅 공급자는 직접적으로 파드나 디플로이먼트를 제거하는 대신
[Eviction API](/docs/tasks/administer-cluster/safely-drain-node/#eviction-api)로
불리는 PodDisruptionBudget을 준수하는 도구를 이용해야 한다.

예를 들어, `kubectl drain` 하위 명령을 사용하면 노드를 서비스 중단으로 표시할 수
있다. `kubectl drain` 을 실행하면, 도구는 사용자가 서비스를 중단하는 노드의
모든 파드를 축출하려고 한다. `kubectl` 이 사용자를 대신하여 수행하는
축출 요청은 일시적으로 거부될 수 있으며,
도구는 대상 노드의 모든 파드가 종료되거나
설정 가능한 타임아웃이 도래할 때까지 주기적으로 모든 실패된 요청을 다시 시도한다.

PDB는 애플리케이션이 필요로 하는 레플리카의 수에 상대적으로, 용인할 수 있는 레플리카의 수를 지정한다.
예를 들어 `.spec.replicas: 5` 의 값을 갖는 디플로이먼트는 어느 시점에든 5개의 파드를 가져야 한다.
만약 해당 디플로이먼트의 PDB가 특정 시점에 파드를 4개 허용한다면,
Eviction API는 한 번에 1개(2개의 파드가 아닌)의 파드의 자발적인 중단을 허용한다.

파드 그룹은 레이블 셀렉터를 사용해서 지정한 애플리케이션으로 구성되며
애플리케이션 컨트롤러(디플로이먼트, 스테이트풀셋 등)를 사용한 것과 같다.

파드의 "의도"하는 수량은 해당 파드를 관리하는 워크로드 리소스의 `.spec.replicas` 를
기반으로 계산한다. 컨트롤 플레인은 파드의 `.metadata.ownerReferences` 를 검사하여
소유하는 워크로드 리소스를 발견한다.

[비자발적 중단](#자발적-중단과-비자발적-중단)은 PDB로는 막을 수 없지만, 
버짓은 차감된다.

애플리케이션의 롤링 업그레이드로 파드가 삭제되거나 사용할 수 없는 경우 중단 버짓에 영향을 준다.
그러나 워크로드 리소스(디플로이먼트, 스테이트풀셋과 같은)는
롤링 업데이트 시 PDB의 제한을 받지 않는다. 대신, 애플리케이션 업데이트 중
실패 처리는 특정 워크로드 리소스에 대한 명세에서 구성된다.

Eviction API를 사용하여 파드를 축출하면,
[PodSpec](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#podspec-v1-core)의
`terminationGracePeriodSeconds` 설정을 준수하여 정상적으로 [종료됨](/ko/docs/concepts/workloads/pods/pod-lifecycle/#파드의-종료) 상태가 된다.

## PodDisruptionBudget 예시 {#pdb-example}

`node-1` 부터 `node-3` 까지 3개의 노드가 있는 클러스터가 있다고 하자.
클러스터에는 여러 애플리케이션을 실행하고 있다.
여러 애플리케이션 중 하나는 `pod-a`, `pod-b`, `pod-c` 로 부르는 3개의 레플리카가 있다. 여기에 `pod-x` 라고 부르는 PDB와 무관한 파드가 보인다.
초기에 파드는 다음과 같이 배치된다.

|       node-1         |       node-2        |       node-3       |
|:--------------------:|:-------------------:|:------------------:|
| pod-a  *available*   | pod-b *available*   | pod-c *available*  |
| pod-x  *available*   |                     |                    |

전체 3개 파드는 디플로이먼트의 일부분으로
전체적으로 항상 3개의 파드 중 최소 2개의 파드를 사용할 수 있도록 하는 PDB를 가지고 있다.

예를 들어, 클러스터 관리자가 커널 버그를 수정하기위해 새 커널 버전으로 재부팅하려는 경우를 가정해보자.
클러스터 관리자는 첫째로 `node-1` 을 `kubectl drain` 명령어를 사용해서 비우려 한다.
`kubectl` 은 `pod-a` 과 `pod-x` 를 축출하려고 한다. 이는 즉시 성공한다.
두 파드는 동시에 `terminating` 상태로 진입한다.
이렇게 하면 클러스터는 다음의 상태가 된다.

|   node-1 *draining*  |       node-2        |       node-3       |
|:--------------------:|:-------------------:|:------------------:|
| pod-a  *terminating* | pod-b *available*   | pod-c *available*  |
| pod-x  *terminating* |                     |                    |

디플로이먼트는 한 개의 파드가 중지되는 것을 알게되고, `pod-d` 라는 대체 파드를 생성한다.
`node-1` 은 차단되어 있어 다른 노드에 위치한다.
무언가가 `pod-x` 의 대체 파드로 `pod-y` 도 생성했다.

(참고: 스테이트풀셋은 `pod-0` 처럼 불릴, `pod-a` 를
교체하기 전에 완전히 중지해야 하며, `pod-0` 로 불리지만, 다른 UID로 생성된다.
그렇지 않으면 이 예시는 스테이트풀셋에도 적용된다.)

이제 클러스터는 다음과 같은 상태이다.

|   node-1 *draining*  |       node-2        |       node-3       |
|:--------------------:|:-------------------:|:------------------:|
| pod-a  *terminating* | pod-b *available*   | pod-c *available*  |
| pod-x  *terminating* | pod-d *starting*    | pod-y              |

어느 순간 파드가 종료되고, 클러스터는 다음과 같은 상태가 된다.

|    node-1 *drained*  |       node-2        |       node-3       |
|:--------------------:|:-------------------:|:------------------:|
|                      | pod-b *available*   | pod-c *available*  |
|                      | pod-d *starting*    | pod-y              |

이 시점에서 만약 성급한 클러스터 관리자가 `node-2` 또는 `node-3` 을
비우려고 하는 경우 디플로이먼트에 available 상태의 파드가 2개 뿐이고,
PDB에 필요한 최소 파드는 2개이기 때문에 drain 명령이 차단된다. 약간의 시간이 지나면 `pod-d` 가 available 상태가 된다.

이제 클러스터는 다음과 같은 상태이다.

|    node-1 *drained*  |       node-2        |       node-3       |
|:--------------------:|:-------------------:|:------------------:|
|                      | pod-b *available*   | pod-c *available*  |
|                      | pod-d *available*   | pod-y              |

이제 클러스터 관리자는 `node-2` 를 비우려고 한다.
drain 커멘드는 `pod-b` 에서 `pod-d` 와 같이 어떤 순서대로 두 파드를 축출하려 할 것이다.
drain 커멘드는 `pod-b` 를 축출하는데 성공했다.
그러나 drain 커멘드가 `pod-d` 를 축출하려 하는 경우
디플로이먼트에 available 상태의 파드는 1개로 축출이 거부된다.

디플로이먼트는`pod-b` 를 대체할 `pod-e` 라는 파드를 생성한다.
클러스터에 `pod-e` 를 스케줄하기 위한 충분한 리소스가 없기 때문에
드레이닝 명령어는 차단된다.
클러스터는 다음 상태로 끝나게 된다.

|    node-1 *drained*  |       node-2        |       node-3       | *no node*          |
|:--------------------:|:-------------------:|:------------------:|:------------------:|
|                      | pod-b *terminating* | pod-c *available*  | pod-e *pending*    |
|                      | pod-d *available*   | pod-y              |                    |

이 시점에서 클러스터 관리자는
클러스터에 노드를 추가해서 업그레이드를 진행해야 한다.

쿠버네티스에 중단이 발생할 수 있는 비율을 어떻게 변화시키는지
다음의 사례를 통해 알 수 있다.

- 애플리케이션에 필요한 레플리카의 수
- 인스턴스를 정상적으로 종료하는데 소요되는 시간
- 새 인스턴스를 시작하는데 소요되는 시간
- 컨트롤러의 유형
- 클러스터의 리소스 용량

## 파드 중단 조건 {#pod-disruption-conditions}

{{< feature-state for_k8s_version="v1.26" state="beta" >}}

{{< note >}}
만약 쿠버네티스 {{< skew currentVersion >}} 보다 낮은 버전을 사용하고 있다면,
해당 버전의 문서를 참조하자.
{{< /note >}}

{{< note >}}
클러스터에서 이 동작을 사용하려면 `PodDisruptionConditions`
[기능 게이트](/ko/docs/reference/command-line-tools-reference/feature-gates/)를 
활성화해야 한다.
{{< /note >}}

이 기능이 활성화되면, 파드 전용 `DisruptionTarget` [컨디션](/ko/docs/concepts/workloads/pods/pod-lifecycle/#파드의-컨디션)이 추가되어
{{<glossary_tooltip term_id="disruption" text="중단(disruption)">}}으로 인해 파드가 삭제될 예정임을 나타낸다.
추가로 컨디션의 `reason` 필드는
파드 종료에 대한 다음 원인 중 하나를 나타낸다.

`PreemptionByKubeScheduler`
: 파드는 더 높은 우선순위를 가진 새 파드를 수용하기 위해 스케줄러에 의해 {{<glossary_tooltip term_id="preemption" text="선점(preempted)">}}된다. 자세한 내용은 [파드 우선순위(priority)와 선점(preemption)](/ko/docs/concepts/scheduling-eviction/pod-priority-preemption/)을 참조해보자.

`DeletionByTaintManager`
: 허용하지 않는 `NoExecute` 테인트(taint) 때문에 파드가 테인트 매니저(`kube-controller-manager` 내의 노드 라이프사이클 컨트롤러의 일부)에 의해 삭제될 예정이다. {{<glossary_tooltip term_id="taint" text="taint">}} 기반 축출을 참조해보자.

`EvictionByEvictionAPI`
: 파드에 {{<glossary_tooltip term_id="api-eviction" text="쿠버네티스 API를 이용한 축출">}}이 표시되었다.

`DeletionByPodGC`
: 더 이상 존재하지 않는 노드에 바인딩된 파드는 [파드의 가비지 콜렉션](/ko/docs/concepts/workloads/pods/pod-lifecycle/#pod-garbage-collection)에 의해 삭제될 예정이다.

`TerminationByKubelet`
: {{<glossary_tooltip term_id="node-pressure-eviction" text="노드 압박-축출">}} 또는 [그레이스풀 노드 셧다운](/ko/docs/concepts/architecture/nodes/#graceful-node-shutdown)으로 인해 kubelet이 파드를 종료시켰다.

{{< note >}}
파드 중단은 중단될 수 있다. 컨트롤 플레인은 동일한 파드의 중단을 
계속 다시 시도하지만, 파드의 중단이 보장되지는 않는다. 결과적으로,
`DisruptionTarget` 컨디션이 파드에 추가될 수 있지만, 해당 파드는 사실상 
삭제되지 않았을 수 있다. 이러한 상황에서는, 일정 시간이 지난 뒤에
파드 중단 상태가 해제된다.
{{< /note >}}

잡(또는 크론잡(CronJob))을 사용할 때, 이러한 파드 중단 조건을 잡의 
[파드 실패 정책](/ko/docs/concepts/workloads/controllers/job#pod-failure-policy)의 일부로 사용할 수 있다.

## 클러스터 소유자와 애플리케이션 소유자의 역할 분리

보통 클러스터 매니저와 애플리케이션 소유자는
서로에 대한 지식이 부족한 별도의 역할로 생각하는 것이 유용하다.
이와 같은 책임의 분리는
다음의 시나리오에서 타당할 수 있다.

- 쿠버네티스 클러스터를 공유하는 애플리케이션 팀이 많고, 자연스럽게 역할이 나누어진 경우
- 타사 도구 또는 타사 서비스를 이용해서
  클러스터 관리를 자동화 하는 경우

Pod Disruption Budget은 역할 분리에 따라
역할에 맞는 인터페이스를 제공한다.

만약 조직에 역할 분리에 따른 책임의 분리가 없다면
Pod Disruption Budget을 사용할 필요가 없다.

## 클러스터에서 중단이 발생할 수 있는 작업을 하는 방법

만약 클러스터 관리자라면, 그리고 클러스터 전체 노드에 노드 또는 시스템 소프트웨어 업그레이드와 같은
중단이 발생할 수 있는 작업을 수행하는 경우 다음과 같은 옵션을 선택한다.

- 업그레이드 하는 동안 다운타임을 허용한다.
- 다른 레플리카 클러스터로 장애조치를 한다.
   - 다운타임은 없지만, 노드 사본과
     전환 작업을 조정하기 위한 인력 비용이 많이 발생할 수 있다.
- PDB를 이용해서 애플리케이션의 중단에 견디도록 작성한다.
   - 다운타임 없음
   - 최소한의 리소스 중복
   - 클러스터 관리의 자동화 확대 적용
   - 내결함성이 있는 애플리케이션의 작성은 까다롭지만
     자발적 중단를 허용하는 작업의 대부분은 오토스케일링과
     비자발적 중단를 지원하는 작업과 겹친다.




## {{% heading "whatsnext" %}}


* [Pod Disruption Budget 설정하기](/docs/tasks/run-application/configure-pdb/)의 단계를 따라서 애플리케이션을 보호한다.

* [노드 비우기](/docs/tasks/administer-cluster/safely-drain-node/)에 대해 자세히 알아보기

* 롤아웃 중에 가용성을 유지하는 단계를 포함하여
  [디플로이먼트 업데이트](/ko/docs/concepts/workloads/controllers/deployment/#디플로이먼트-업데이트)에 대해 알아보기

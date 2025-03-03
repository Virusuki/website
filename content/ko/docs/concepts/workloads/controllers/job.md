---
# reviewers:
# - alculquicondor
# - erictune
# - soltysh
title: 잡
content_type: concept
feature:
  title: 배치 실행
  description: >
    쿠버네티스는 서비스 외에도 배치(batch)와 CI 워크로드를 관리할 수 있으며, 원하는 경우 실패한 컨테이너를 교체할 수 있다.
weight: 50
---

<!-- overview -->

잡에서 하나 이상의 파드를 생성하고 지정된 수의 파드가 성공적으로 종료될 때까지 계속해서 파드의 실행을 재시도한다.
파드가 성공적으로 완료되면, 성공적으로 완료된 잡을 추적한다.  지정된 수의
성공 완료에 도달하면, 작업(즉, 잡)이 완료된다.  잡을 삭제하면 잡이 생성한
파드가 정리된다. 작업을 일시 중지하면 작업이 다시 재개될 때까지 활성 파드가
삭제된다.

간단한 사례는 잡 오브젝트를 하나 생성해서 파드 하나를 안정적으로 실행하고 완료하는 것이다.
첫 번째 파드가 실패 또는 삭제된 경우(예로는 노드 하드웨어의 실패 또는
노드 재부팅) 잡 오브젝트는 새로운 파드를 기동시킨다.

잡을 사용하면 여러 파드를 병렬로 실행할 수도 있다.

잡을 스케줄에 따라 구동하고 싶은 경우(단일 작업이든, 여러 작업의 병렬 수행이든), 
[크론잡(CronJob)](/ko/docs/concepts/workloads/controllers/cron-jobs/)을 참고한다.

<!-- body -->

## 예시 잡 실행하기

다음은 잡 설정 예시이다.  예시는 파이(π)의 2000 자리까지 계산해서 출력한다.
이를 완료하는 데 약 10초가 소요된다.

{{< codenew file="controllers/job.yaml" >}}

이 명령으로 예시를 실행할 수 있다.

```shell
kubectl apply -f https://kubernetes.io/examples/controllers/job.yaml
```

출력 결과는 다음과 같다.

```
job.batch/pi created
```

`kubectl` 을 사용해서 잡 상태를 확인한다.

{{< tabs name="Check status of Job" >}}
{{< tab name="kubectl describe job pi" codelang="bash" >}}
Name:           pi
Namespace:      default
Selector:       controller-uid=c9948307-e56d-4b5d-8302-ae2d7b7da67c
Labels:         controller-uid=c9948307-e56d-4b5d-8302-ae2d7b7da67c
                job-name=pi
Annotations:    kubectl.kubernetes.io/last-applied-configuration:
                  {"apiVersion":"batch/v1","kind":"Job","metadata":{"annotations":{},"name":"pi","namespace":"default"},"spec":{"backoffLimit":4,"template":...
Parallelism:    1
Completions:    1
Start Time:     Mon, 02 Dec 2019 15:20:11 +0200
Completed At:   Mon, 02 Dec 2019 15:21:16 +0200
Duration:       65s
Pods Statuses:  0 Running / 1 Succeeded / 0 Failed
Pod Template:
  Labels:  controller-uid=c9948307-e56d-4b5d-8302-ae2d7b7da67c
           job-name=pi
  Containers:
   pi:
    Image:      perl:5.34.0
    Port:       <none>
    Host Port:  <none>
    Command:
      perl
      -Mbignum=bpi
      -wle
      print bpi(2000)
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From            Message
  ----    ------            ----  ----            -------
  Normal  SuccessfulCreate  14m   job-controller  Created pod: pi-5rwd7
{{< /tab >}}
{{< tab name="kubectl get job pi -o yaml" codelang="bash" >}}
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"batch/v1","kind":"Job","metadata":{"annotations":{},"name":"pi","namespace":"default"},"spec":{"backoffLimit":4,"template":{"spec":{"containers":[{"command":["perl","-Mbignum=bpi","-wle","print bpi(2000)"],"image":"perl","name":"pi"}],"restartPolicy":"Never"}}}}
  creationTimestamp: "2022-06-15T08:40:15Z"
  generation: 1
  labels:
    controller-uid: 863452e6-270d-420e-9b94-53a54146c223
    job-name: pi
  name: pi
  namespace: default
  resourceVersion: "987"
  uid: 863452e6-270d-420e-9b94-53a54146c223
spec:
  backoffLimit: 4
  completionMode: NonIndexed
  completions: 1
  parallelism: 1
  selector:
    matchLabels:
      controller-uid: 863452e6-270d-420e-9b94-53a54146c223
  suspend: false
  template:
    metadata:
      creationTimestamp: null
      labels:
        controller-uid: 863452e6-270d-420e-9b94-53a54146c223
        job-name: pi
    spec:
      containers:
      - command:
        - perl
        - -Mbignum=bpi
        - -wle
        - print bpi(2000)
        image: perl:5.34.0
        imagePullPolicy: Always
        name: pi
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Never
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  active: 1
  ready: 1
  startTime: "2022-06-15T08:40:15Z"
{{< /tab >}}
{{< /tabs >}}

`kubectl get pods` 를 사용해서 잡의 완료된 파드를 본다.

잡에 속하는 모든 파드를 기계적으로 읽을 수 있는 양식으로 나열하려면, 다음과 같은 명령을 사용할 수 있다.

```shell
pods=$(kubectl get pods --selector=job-name=pi --output=jsonpath='{.items[*].metadata.name}')
echo $pods
```

출력 결과는 다음과 같다.

```
pi-5rwd7
```

여기서 셀렉터는 잡의 셀렉터와 동일하다. `--output=jsonpath` 옵션은 반환된
목록에 있는 각 파드의 이름으로 표현식을 지정한다.

파드 중 하나를 표준 출력으로 본다.

```shell
kubectl logs $pods
```

출력 결과는 다음과 같다.

```
3.1415926535897932384626433832795028841971693993751058209749445923078164062862089986280348253421170679821480865132823066470938446095505822317253594081284811174502841027019385211055596446229489549303819644288109756659334461284756482337867831652712019091456485669234603486104543266482133936072602491412737245870066063155881748815209209628292540917153643678925903600113305305488204665213841469519415116094330572703657595919530921861173819326117931051185480744623799627495673518857527248912279381830119491298336733624406566430860213949463952247371907021798609437027705392171762931767523846748184676694051320005681271452635608277857713427577896091736371787214684409012249534301465495853710507922796892589235420199561121290219608640344181598136297747713099605187072113499999983729780499510597317328160963185950244594553469083026425223082533446850352619311881710100031378387528865875332083814206171776691473035982534904287554687311595628638823537875937519577818577805321712268066130019278766111959092164201989380952572010654858632788659361533818279682303019520353018529689957736225994138912497217752834791315155748572424541506959508295331168617278558890750983817546374649393192550604009277016711390098488240128583616035637076601047101819429555961989467678374494482553797747268471040475346462080466842590694912933136770289891521047521620569660240580381501935112533824300355876402474964732639141992726042699227967823547816360093417216412199245863150302861829745557067498385054945885869269956909272107975093029553211653449872027559602364806654991198818347977535663698074265425278625518184175746728909777727938000816470600161452491921732172147723501414419735685481613611573525521334757418494684385233239073941433345477624168625189835694855620992192221842725502542568876717904946016534668049886272327917860857843838279679766814541009538837863609506800642251252051173929848960841284886269456042419652850222106611863067442786220391949450471237137869609563643719172874677646575739624138908658326459958133904780275901
```

## 잡 사양 작성하기

다른 쿠버네티스의 설정과 마찬가지로 잡에는 `apiVersion`, `kind` 그리고 `metadata` 필드가 필요하다.
잡의 이름은 유효한 [DNS 서브도메인 이름](/ko/docs/concepts/overview/working-with-objects/names/#dns-서브도메인-이름)이어야 한다.

잡에는 [`.spec` 섹션](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status)도 필요하다.

### 파드 템플릿

`.spec.template` 은 `.spec` 의 유일한 필수 필드이다.


`.spec.template` 은 [파드 템플릿](/ko/docs/concepts/workloads/pods/#파드-템플릿)이다. 이것은 `apiVersion` 또는 `kind` 가 없다는 것을 제외한다면 {{< glossary_tooltip text="파드" term_id="pod" >}}와 정확하게 같은 스키마를 가지고 있다.

추가로 파드의 필수 필드 외에도 잡의 파드 템플릿은 적절한
레이블([파드 셀렉터](#파드-셀렉터)를 본다)과 적절한 재시작 정책을 명시해야 한다.

`Never` 또는 `OnFailure` 와 같은 [`RestartPolicy`](/ko/docs/concepts/workloads/pods/pod-lifecycle/#재시작-정책)만 허용된다.

### 파드 셀렉터

`.spec.selector` 필드는 선택 사항이다.  대부분의 케이스에서 지정해서는 안된다.
[자신의 파드 셀렉터를 지정하기](#자신의-파드-셀렉터를-지정하기) 섹션을 참고한다.


### 잡에 대한 병렬 실행 {#parallel-jobs}

잡으로 실행하기에 적합한 작업 유형은 크게 세 가지가 있다.

1. 비-병렬(Non-parallel) 잡:
   - 일반적으로, 파드가 실패하지 않은 한, 하나의 파드만 시작된다.
   - 파드가 성공적으로 종료하자마자 즉시 잡이 완료된다.
1. *고정적(fixed)인 완료 횟수* 를 가진 병렬 잡:
   - `.spec.completions` 에 0이 아닌 양수 값을 지정한다.
   - 잡은 전체 작업을 나타내며, `.spec.completions` 성공한 파드가 있을 때 완료된다.
   - `.spec.completionMode="Indexed"` 를 사용할 때, 각 파드는 0에서 `.spec.completions-1` 범위 내의 서로 다른 인덱스를 가져온다.
1. *작업 큐(queue)* 가 있는 병렬 잡:
   - `.spec.completions` 를 지정하지 않고, `.spec.parallelism` 를 기본으로 한다.
   - 파드는 각자 또는 외부 서비스 간에 조정을 통해 각각의 작업을 결정해야 한다. 예를 들어 파드는 작업 큐에서 최대 N 개의 항목을 일괄로 가져올(fetch) 수 있다.
   - 각 파드는 모든 피어들의 작업이 완료되었는지 여부를 독립적으로 판단할 수 있으며, 결과적으로 전체 잡이 완료되게 한다.
   - 잡의 _모든_ 파드가 성공적으로 종료되면, 새로운 파드는 생성되지 않는다.
   - 하나 이상의 파드가 성공적으로 종료되고, 모든 파드가 종료되면 잡은 성공적으로 완료된다.
   - 성공적으로 종료된 파드가 하나라도 생긴 경우, 다른 파드들은 해당 작업을 지속하지 않아야 하며 어떠한 출력도 작성하면 안 된다.  파드들은 모두 종료되는 과정에 있어야 한다.

_비-병렬_ 잡은 `.spec.completions` 와 `.spec.parallelism` 모두를 설정하지 않은 채로 둘 수 있다. 이때 둘 다
설정하지 않은 경우 1이 기본으로 설정된다.

_고정적인 완료 횟수_ 잡은 `.spec.completions` 을 필요한 완료 횟수로 설정해야 한다.
`.spec.parallelism` 을 설정할 수 있고, 설정하지 않으면 1이 기본으로 설정된다.

_작업 큐_ 잡은 `.spec.completions` 를 설정하지 않은 상태로 두고, `.spec.parallelism` 을
음수가 아닌 정수로 설정해야 한다.

다른 유형의 잡을 사용하는 방법에 대한 더 자세한 정보는 [잡 패턴](#잡-패턴) 섹션을 본다.

#### 병렬 처리 제어하기

요청된 병렬 처리(`.spec.parallelism`)는 음수가 아닌 값으로 설정할 수 있다.
만약 지정되지 않은 경우에는 1이 기본이 된다.
만약 0으로 지정되면 병렬 처리가 증가할 때까지 사실상 일시 중지된다.

실제 병렬 처리(모든 인스턴스에서 실행되는 파드의 수)는 여러가지 이유로 요청된
병렬 처리보다 많거나 적을 수 있다.

- _고정적인 완료 횟수(fixed completion count)_ 잡의 경우, 병렬로 실행 중인 파드의 수는 남은 완료 수를
  초과하지 않는다.  `.spec.parallelism` 의 더 큰 값은 사실상 무시된다.
- _작업 큐_ 잡은 파드가 성공한 이후에 새로운 파드가 시작되지 않는다. 그러나 나머지 파드는 완료될 수 있다.
- 만약 잡 {{< glossary_tooltip term_id="controller" text="컨트롤러" >}} 가 반응할 시간이 없는 경우
- 만약 잡 컨트롤러가 어떤 이유(`ResourceQuota` 의 부족, 권한 부족 등)로든 파드 생성에 실패한 경우,
  요청한 것보다 적은 수의 파드가 있을 수 있다.
- 잡 컨트롤러는 동일한 잡에서 과도하게 실패한 이전 파드들로 인해 새로운 파드의 생성을 조절할 수 있다.
- 파드가 정상적으로(gracefully) 종료되면, 중지하는데 시간이 소요된다.

### 완료 모드

{{< feature-state for_k8s_version="v1.24" state="stable" >}}

완료 횟수가 _고정적인 완료 횟수_ 즉, null이 아닌 `.spec.completions` 가 있는 잡은
`.spec.completionMode` 에 지정된 완료 모드를 가질 수 있다.

- `NonIndexed` (기본값): `.spec.completions` 가 성공적으로
  완료된 파드가 있는 경우 작업이 완료된 것으로 간주된다. 즉, 각 파드
  완료는 서로 상동하다(homologous). null `.spec.completions` 가 있는
  잡은 암시적으로 `NonIndexed` 이다.
- `Indexed`: 잡의 파드는 연결된 완료 인덱스를 0에서 `.spec.completions-1` 까지 
  가져온다. 이 인덱스는 다음의 세 가지 메카니즘으로 얻을 수 있다.
  - 파드 어노테이션 `batch.kubernetes.io/job-completion-index`.
  - 파드 호스트네임 중 일부(`$(job-name)-$(index)` 형태).
    인덱스된(Indexed) 잡과 
    {{< glossary_tooltip text="서비스" term_id="Service" >}}를 결합하여 사용하고 있다면, 잡에 속한 파드는
    DNS를 이용하여 서로를 디스커버 하기 위해 사전에 결정된 호스트네임을 사용할 수 있다.
    이를 어떻게 설정하는지에 대해 궁금하다면, [파드 간 통신을 위한 잡](/docs/tasks/job/job-with-pod-to-pod-communication/)를 확인한다.
  - 컨테이너화된 태스크의 경우, 환경 변수 `JOB_COMPLETION_INDEX`에 있다.

  각 인덱스에 대해 성공적으로 완료된 파드가 하나 있으면 작업이 완료된 것으로
  간주된다. 이 모드를 사용하는 방법에 대한 자세한 내용은
  [정적 작업 할당을 사용한 병렬 처리를 위해 인덱싱된 잡](/ko/docs/tasks/job/indexed-parallel-processing-static/)을 참고한다.
  참고로, 드물기는 하지만, 동일한 인덱스에 대해 둘 이상의 파드를 시작할 수
  있지만, 그 중 하나만 완료 횟수에 포함된다.


## 파드와 컨테이너 장애 처리하기

파드내 컨테이너의 프로세스가 0이 아닌 종료 코드로 종료되었거나 컨테이너 메모리 제한을
초과해서 죽는 등의 여러가지 이유로 실패할 수 있다.  만약 이런 일이
발생하고 `.spec.template.spec.restartPolicy = "OnFailure"` 라면 파드는
노드에 그대로 유지되지만, 컨테이너는 다시 실행된다.  따라서 프로그램은 로컬에서 재시작될 때의
케이스를 다루거나 `.spec.template.spec.restartPolicy = "Never"` 로 지정해야 한다.
더 자세한 정보는 [파드 라이프사이클](/ko/docs/concepts/workloads/pods/pod-lifecycle/#상태-예제)의 `restartPolicy` 를 본다.

파드가 노드에서 내보내지는 경우(노드 업그레이드, 재부팅, 삭제 등) 또는 파드의 컨테이너가 실패
되고 `.spec.template.spec.restartPolicy = "Never"` 로 설정됨과 같은 여러 이유로
전체 파드가 실패할 수 있다.  파드가 실패하면 잡 컨트롤러는
새 파드를 시작한다.  이 의미는 애플리케이션이 새 파드에서 재시작될 때 이 케이스를 처리해야
한다는 점이다.  특히, 이전 실행으로 인한 임시파일, 잠금, 불완전한 출력 그리고 이와 유사한
것들을 처리해야 한다.

기본적으로 파드의 실패는 `.spec.backoffLimit` 제한값으로 계산되며,
자세한 내용은 [파드 백오프(backoff) 실패 정책](#파드-백오프-backoff-실패-정책)을 확인한다. 그러나,
잡의 [파드 실패 정책](#pod-failure-policy)을 설정하여 파드의 실패 수준을 조절하여 사용할 수도 있다.

`.spec.parallelism = 1`, `.spec.completions = 1` 그리고
`.spec.template.spec.restartPolicy = "Never"` 를 지정하더라도 같은 프로그램을
두 번 시작하는 경우가 있다는 점을 참고한다.

`.spec.parallelism` 그리고 `.spec.completions` 를 모두 1보다 크게 지정한다면 한번에
여러 개의 파드가 실행될 수 있다.  따라서 파드는 동시성에 대해서도 관대(tolerant)해야 한다.

만약 [기능 게이트](/ko/docs/reference/command-line-tools-reference/feature-gates/)인
`PodDisruptionConditions`와 `JobPodFailurePolicy`가 모두 활성화되어 있고,
`.spec.podFailurePolicy` 필드가 설정되어 있다면, 잡 컨트롤러는 종료 중인
파드(`.metadata.deletionTimestamp` 필드가 설정된 파드)가 종료 상태(`.status.phase` 값이 `Failed` 혹은 `Succeeded`)가 될 때 까지 
해당 파드를 실패로 간주하지 않는다. 그러나, 잡 컨트롤러는
파드가 명백히 종료되었다고 판단하면 곧바로 대체 파드를 생성한다.
파드가 한 번 종료되면, 잡 컨트롤러는 방금 종료된 파드를 고려하여
관련 작업에 대해 `.backoffLimit`과 `.podFailurePolicy`를 평가한다.

이러한 조건들이 하나라도 충족되지 않을 경우, 잡 컨트롤러는
종료 중인 파드가 추후 `phase: "Succeded"`로 종료된다고 할지라도,
해당 파드를 실패한 파드로 즉시 간주한다.

### 파드 백오프(backoff) 실패 정책

구성에 논리적 오류가 포함되어 있어서 몇 회의 재시도 이후에
잡이 실패되도록 만들어야 하는 경우가 있다.
이렇게 하려면 `.spec.backoffLimit`의 값에
재시도(잡을 실패로 처리하기 이전까지) 횟수를 설정한다. 백오프 제한은 기본적으로 6으로 설정되어 있다.
잡에 연계된 실패 상태 파드는 6분 내에서 지수적으로 증가하는
백-오프 지연(10초, 20초, 40초 ...)을 적용하여, 잡 컨트롤러에 의해 재생성된다. 

재시도 횟수는 다음 두 가지 방법으로 계산된다.
- `.status.phase = "Failed"`인 파드의 수.
- `restartPolicy = "OnFailure"`를 사용하는 경우, `.status.phase`가 
  `Pending`이거나 `Running`인 파드들이 가지고 있는 모든 컨테이너의 수.

계산 중 하나가 `.spec.backoffLimit`에 도달하면, 잡이
실패한 것으로 간주한다.

{{< note >}}
만약 잡에 `restartPolicy = "OnFailure"` 가 있는 경우 잡 백오프 한계에
도달하면 잡을 실행 중인 파드가 종료된다. 이로 인해 잡 실행 파일의 디버깅이
더 어려워질 수 있다. 디버깅하거나 로깅 시스템을 사용해서 실패한 작업의 결과를 실수로 손실되지 않도록
하려면 `restartPolicy = "Never"` 로 설정하는 것을 권장한다.
{{< /note >}}

## 잡의 종료와 정리

잡이 완료되면 파드가 더 이상 생성되지도 않지만, [일반적으로는](#pod-backoff-failure-policy) 삭제되지도 않는다.  
이를 유지하면
완료된 파드의 로그를 계속 보며 에러, 경고 또는 다른 기타 진단 출력을 확인할 수 있다.
잡 오브젝트는 완료된 후에도 상태를 볼 수 있도록 남아 있다. 상태를 확인한 후 이전 잡을 삭제하는 것은 사용자의 몫이다.
`kubectl` 로 잡을 삭제할 수 있다 (예: `kubectl delete jobs/pi` 또는 `kubectl delete -f ./job.yaml`). `kubectl` 을 사용해서 잡을 삭제하면 생성된 모든 파드도 함께 삭제된다.

기본적으로 파드의 실패(`restartPolicy=Never`) 또는 컨테이너가 오류(`restartPolicy=OnFailure`)로 종료되지 않는 한, 잡은 중단되지 않고 실행되고
이때 위에서 설명했던 `.spec.backoffLimit` 까지 연기된다. `.spec.backoffLimit` 에 도달하면 잡은 실패로 표기되고 실행 중인 모든 파드는 종료된다.

잡을 종료하는 또 다른 방법은 유효 데드라인을 설정하는 것이다.
잡의 `.spec.activeDeadlineSeconds` 필드를 초 단위로 설정하면 된다.
`activeDeadlineSeconds` 는 생성된 파드의 수에 관계 없이 잡의 기간에 적용된다.
잡이 `activeDeadlineSeconds` 에 도달하면, 실행 중인 모든 파드가 종료되고 잡의 상태는 `reason: DeadlineExceeded` 와 함께 `type: Failed` 가 된다.

잡의 `.spec.activeDeadlineSeconds` 는 `.spec.backoffLimit` 보다 우선한다는 점을 참고한다. 따라서 하나 이상 실패한 파드를 재시도하는 잡은 `backoffLimit` 에 도달하지 않은 경우에도 `activeDeadlineSeconds` 에 지정된 시간 제한에 도달하면 추가 파드를 배포하지 않는다.

예시:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-with-timeout
spec:
  backoffLimit: 5
  activeDeadlineSeconds: 100
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34.0
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
```

잡의 사양과 잡의 [파드 템플릿 사양](/ko/docs/concepts/workloads/pods/init-containers/#자세한-동작)에는 모두 `activeDeadlineSeconds` 필드가 있다는 점을 참고한다. 이 필드를 적절한 레벨로 설정해야 한다.

`restartPolicy` 는 잡 자체에 적용되는 것이 아니라 파드에 적용된다는 점을 유념한다. 잡의 상태가 `type: Failed` 이 되면, 잡의 자동 재시작은 없다.
즉, `.spec.activeDeadlineSeconds` 와 `.spec.backoffLimit` 로 활성화된 잡의 종료 메커니즘은 영구적인 잡의 실패를 유발하며 이를 해결하기 위해 수동 개입이 필요하다.

## 완료된 잡을 자동으로 정리 {#clean-up-finished-jobs-automatically}

완료된 잡은 일반적으로 시스템에서 더 이상 필요로 하지 않는다. 시스템 내에
이를 유지한다면 API 서버에 부담이 된다.
만약 [크론잡](/ko/docs/concepts/workloads/controllers/cron-jobs/)과
같은 상위 레벨 컨트롤러가 잡을 직접 관리하는 경우,
지정된 용량 기반 정리 정책에 따라 크론잡이 잡을 정리할 수 있다.

### 완료된 잡을 위한 TTL 메커니즘

{{< feature-state for_k8s_version="v1.23" state="stable" >}}

완료된 잡 (`Complete` 또는 `Failed`)을 자동으로 정리하는 또 다른 방법은
잡의 `.spec.ttlSecondsAfterFinished` 필드를 지정해서 완료된 리소스에 대해
[TTL 컨트롤러](/ko/docs/concepts/workloads/controllers/ttlafterfinished/)에서
제공하는 TTL 메커니즘을 사용하는
것이다.

TTL 컨트롤러는 잡을 정리하면 잡을 계단식으로 삭제한다.
즉, 잡과 함께 파드와 같은 종속 오브젝트를 삭제한다. 잡을
삭제하면 finalizer와 같은 라이프사이클 보증이 보장되는 것을
참고한다.

예시:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-with-ttl
spec:
  ttlSecondsAfterFinished: 100
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34.0
        command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
```

`pi-with-ttl` 잡은 완료 후 `100` 초 이후에
자동으로 삭제될 수 있다.

만약 필드를 `0` 으로 설정하면, 잡이 완료된 직후에 자동으로
삭제되도록 할 수 있다. 만약 필드를 설정하지 않으면, 이 잡이 완료된
후에 TTL 컨트롤러에 의해 정리되지 않는다.

{{< note >}}
`ttlSecondsAfterFinished` 필드를 설정하는 것을 권장하는데, 
이는 관리되지 않는 잡(직접 생성한, 
크론잡 등 다른 워크로드 API를 통해 간접적으로 생성하지 않은 잡)의 
기본 삭제 정책이 `orphanDependents`(관리되지 않는 잡이 완전히 삭제되어도 
해당 잡에 의해 생성된 파드를 남겨둠)이기 때문이다.
삭제된 잡의 파드가 실패하거나 완료된 뒤 
{{< glossary_tooltip text="컨트롤 플레인" term_id="control-plane" >}}이 언젠가 
[가비지 콜렉션](/ko/docs/concepts/workloads/pods/pod-lifecycle/#pod-garbage-collection)을 한다고 해도, 
이렇게 남아 있는 파드는 클러스터의 성능을 저하시키거나 
최악의 경우에는 이 성능 저하로 인해 클러스터가 중단될 수도 있다.

[리밋 레인지(Limit Range)](/ko/docs/concepts/policy/limit-range/)와 
[리소스 쿼터](/ko/docs/concepts/policy/resource-quotas/)를 사용하여 
특정 네임스페이스가 사용할 수 있는 자원량을 제한할 수 
있다.
{{< /note >}}


## 잡 패턴

잡 오브젝트를 사용해서 신뢰할 수 있는 파드의 병렬 실행을 지원할 수 있다.  잡 오브젝트는 과학
컴퓨팅(scientific computing)에서 일반적으로 사용되는 밀접하게 통신하는 병렬 프로세스를 지원하도록
설계되지 않았다. 잡 오브젝트는 독립적이지만 관련된 *작업 항목* 집합의 병렬 처리를 지원한다.
여기에는 전송할 이메일들, 렌더링할 프레임, 코드 변환이 필요한 파일, NoSQL 데이터베이스에서의
키 범위 스캔 등이 있다.

복잡한 시스템에는 여러 개의 다른 작업 항목 집합이 있을 수 있다.  여기서는 사용자와
함께 관리하려는 하나의 작업 항목 집합 &mdash; *배치 잡* 을 고려하고 있다.

병렬 계산에는 몇몇 다른 패턴이 있으며 각각의 장단점이 있다.
트레이드오프는 다음과 같다.

- 각 작업 항목에 대한 하나의 잡 오브젝트 vs 모든 작업 항목에 대한 단일 잡 오브젝트.  후자는
  작업 항목 수가 많은 경우 더 적합하다.  전자는 사용자와 시스템이 많은 수의 잡 오브젝트를
  관리해야 하는 약간의 오버헤드를 만든다.
- 작업 항목과 동일한 개수의 파드 생성 vs 각 파드에서 다수의 작업 항목을 처리.
  전자는 일반적으로 기존 코드와 컨테이너를 거의 수정할 필요가 없다.  후자는
  이전 글 머리표(-)와 비슷한 이유로 많은 수의 작업 항목에 적합하다.
- 여러 접근 방식이 작업 큐를 사용한다.  이를 위해서는 큐 서비스를 실행하고,
  작업 큐를 사용하도록 기존 프로그램이나 컨테이너를 수정해야 한다.
  다른 접근 방식들은 기존에 컨테이너화된 애플리케이션에 보다 쉽게 적용할 수 있다.


여기에 트레이드오프가 요약되어 있고, 2열에서 4열까지가 위의 트레이드오프에 해당한다.
패턴 이름은 예시와 더 자세한 설명을 위한 링크이다.

|                  패턴                            | 단일 잡 오브젝트     | 작업 항목보다 파드가 적은가?      | 수정되지 않은 앱을 사용하는가? |
| ----------------------------------------------- |:-----------------:|:---------------------------:|:-------------------:|
| [작업 항목 당 파드가 있는 큐]                        |         ✓         |                             |      때때로           |
| [가변 파드 수를 가진 큐]                            |         ✓         |             ✓               |                     |
| [정적 작업 할당을 사용한 인덱싱된 잡]                  |         ✓         |                             |          ✓          |
| [잡 템플릿 확장]                                   |                   |                             |          ✓          |
| [파드 간 통신을 위한 잡]                              |         ✓         |           때때로             |      때때로           |

`.spec.completions` 로 완료를 지정할 때, 잡 컨트롤러에 의해 생성된 각 파드는
동일한 [`사양`](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status)을 갖는다.  이 의미는
작업의 모든 파드는 동일한 명령 줄과 동일한 이미지,
동일한 볼륨, (거의) 동일한 환경 변수를 가진다는 점이다.  이 패턴은
파드가 다른 작업을 수행하도록 배열하는 다른 방법이다.

이 표는 각 패턴에 필요한 `.spec.parallelism` 그리고 `.spec.completions` 설정을 보여준다.
여기서 `W` 는 작업 항목의 수이다.

|             패턴                                 | `.spec.completions` |  `.spec.parallelism` |
| ----------------------------------------------- |:-------------------:|:--------------------:|
| [작업 항목 당 파드가 있는 큐]                        |          W          |        any           |
| [가변 파드 수를 가진 큐]                            |         null        |        any           |
| [정적 작업 할당을 사용한 인덱싱된 잡]                  |          W          |        any           |
| [잡 템플릿 확장]                                   |          1          |     1이어야 함         |
| [파드 간 통신을 위한 잡]                            |          W          |         W            |

[작업 항목 당 파드가 있는 큐]: /ko/docs/tasks/job/coarse-parallel-processing-work-queue/
[가변 파드 수를 가진 큐]: /ko/docs/tasks/job/fine-parallel-processing-work-queue/
[정적 작업 할당을 사용한 인덱싱된 잡]: /ko/docs/tasks/job/indexed-parallel-processing-static/
[잡 템플릿 확장]: /ko/docs/tasks/job/parallel-processing-expansion/
[파드 간 통신을 위한 잡]: /docs/tasks/job/job-with-pod-to-pod-communication/

## 고급 사용법

### 잡 일시 중지

{{< feature-state for_k8s_version="v1.24" state="stable" >}}

잡이 생성되면, 잡 컨트롤러는 잡의 요구 사항을 충족하기 위해
즉시 파드 생성을 시작하고 잡이 완료될 때까지
계속한다. 그러나, 잡의 실행을 일시적으로 중단하고 나중에
재개하거나, 잡을 중단 상태로 생성하고 언제 시작할지를 
커스텀 컨트롤러가 나중에 결정하도록 하고 싶을 수도 있다. 

잡을 일시 중지하려면, 잡의 `.spec.suspend` 필드를 true로
업데이트할 수 있다. 이후에, 다시 재개하려면, false로 업데이트한다.
`.spec.suspend` 로 설정된 잡을 생성하면 일시 중지된 상태로
생성된다.

잡이 일시 중지에서 재개되면, 해당 `.status.startTime` 필드가
현재 시간으로 재설정된다. 즉, 잡이 일시 중지 및 재개되면 `.spec.activeDeadlineSeconds`
타이머가 중지되고 재설정된다.

잡을 일시 중지하면, `Completed` 상태가 아닌 모든 실행중인 파드가 SIGTERM 시그널로 [종료된다](/ko/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination).
파드의 정상 종료 기간이 적용되며 사용자의 파드는 이 기간 동안에
이 시그널을 처리해야 한다. 나중에 진행 상황을 저장하거나
변경 사항을 취소하는 작업이 포함될 수 있다. 이 방법으로 종료된 파드는
잡의 `completions` 수에 포함되지 않는다.

일시 중지된 상태의 잡 정의 예시는 다음과 같다.

```shell
kubectl get job myjob -o yaml
```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: myjob
spec:
  suspend: true
  parallelism: 1
  completions: 5
  template:
    spec:
      ...
```

명령 줄에서 잡을 패치하여 잡 일시 중지를 전환할 수 있다.

활성화된 잡 일시 중지

```shell
kubectl patch job/myjob --type=strategic --patch '{"spec":{"suspend":true}}'
```

일시 중지된 잡 재개

```shell
kubectl patch job/myjob --type=strategic --patch '{"spec":{"suspend":false}}'
```

잡의 상태를 사용하여 잡이 일시 중지되었는지 또는 과거에 일시 중지되었는지
확인할 수 있다.

```shell
kubectl get jobs/myjob -o yaml
```

```yaml
apiVersion: batch/v1
kind: Job
# .metadata and .spec omitted
status:
  conditions:
  - lastProbeTime: "2021-02-05T13:14:33Z"
    lastTransitionTime: "2021-02-05T13:14:33Z"
    status: "True"
    type: Suspended
  startTime: "2021-02-05T13:13:48Z"
```

"True" 상태인 "Suspended" 유형의 잡의 컨디션은 잡이
일시 중지되었음을 의미한다. 이 `lastTransitionTime` 필드는 잡이 일시 중지된
기간을 결정하는 데 사용할 수 있다. 해당 컨디션의 상태가 "False"이면, 잡이
이전에 일시 중지되었다가 현재 실행 중이다. 이러한 컨디션이
잡의 상태에 없으면, 잡이 중지되지 않은 것이다.

잡이 일시 중지 및 재개될 때에도 이벤트가 생성된다.

```shell
kubectl describe jobs/myjob
```

```
Name:           myjob
...
Events:
  Type    Reason            Age   From            Message
  ----    ------            ----  ----            -------
  Normal  SuccessfulCreate  12m   job-controller  Created pod: myjob-hlrpl
  Normal  SuccessfulDelete  11m   job-controller  Deleted pod: myjob-hlrpl
  Normal  Suspended         11m   job-controller  Job suspended
  Normal  SuccessfulCreate  3s    job-controller  Created pod: myjob-jvb44
  Normal  Resumed           3s    job-controller  Job resumed
```

마지막 4개의 이벤트, 특히 "Suspended" 및 "Resumed" 이벤트는
`.spec.suspend` 필드를 전환한 결과이다. 이 두 이벤트 사이의 시간동안
파드가 생성되지 않았지만, 잡이 재개되자마자 파드 생성이 다시
시작되었음을 알 수 있다.

### 가변적 스케줄링 지시

{{< feature-state for_k8s_version="v1.23" state="beta" >}}

{{< note >}}
이 기능을 사용하려면, 
[API 서버](/docs/reference/command-line-tools-reference/kube-apiserver/)에 
`JobMutableNodeSchedulingDirectives` [기능 게이트](/ko/docs/reference/command-line-tools-reference/feature-gates/)를 활성화해야 한다.
이 기능은 기본적으로 활성화되어 있다.
{{< /note >}}

병렬 잡에서 대부분의 경우 파드를 특정 제약 조건 하에서 실행하고 싶을 것이다. 
(예를 들면 동일 존에서 실행하거나, 또는 GPU 모델 x 또는 y를 사용하지만 둘을 혼합하지는 않는 등)

[suspend](#suspending-a-job) 필드는 이러한 목적을 달성하기 위한 첫 번째 단계이다. 
이 필드를 사용하면 커스텀 큐(queue) 컨트롤러가 잡이 언제 시작될지를 결정할 수 있다. 
그러나, 잡이 재개된 이후에는, 커스텀 큐 컨트롤러는 잡의 파드가 실제로 어디에 할당되는지에 대해서는 영향을 주지 않는다.

이 기능을 이용하여 잡이 실행되기 전에 잡의 스케줄링 지시를 업데이트할 수 있으며, 
이를 통해 커스텀 큐 컨트롤러가 파드 배치에 영향을 줌과 동시에 
노드로의 파드 실제 할당 작업을 kube-scheduler로부터 경감시켜 줄 수 있도록 한다. 
이는 이전에 재개된 적이 없는 중지된 잡에 대해서만 허용된다.

잡의 파드 템플릿 필드 중, 노드 어피니티(node affinity), 노드 셀렉터(node selector), 
톨러레이션(toleration), 레이블(label), 어노테이션(annotation)은 업데이트가 가능하다.

### 자신의 파드 셀렉터를 지정하기

일반적으로 잡 오브젝트를 생성할 때 `.spec.selector` 를 지정하지 않는다.
시스템의 기본적인 로직은 잡이 생성될 때 이 필드를 추가한다.
이것은 다른 잡과 겹치지 않는 셀렉터 값을 선택한다.

그러나, 일부 케이스에서는 이 자동화된 설정 셀렉터를 재정의해야 할 수도 있다.
이를 위해 잡의 `.spec.selector` 를 설정할 수 있다.

이 것을 할 때는 매우 주의해야 한다.  만약 해당 잡의 파드에 고유하지
않고 연관이 없는 파드와 일치하는 레이블 셀렉터를 지정하면, 연관이 없는 잡의 파드가 삭제되거나,
해당 잡이 다른 파드가 완료한 것으로 수를 세거나, 하나 또는
양쪽 잡 모두 파드 생성이나 실행 완료를 거부할 수도 있다.  만약 고유하지 않은 셀렉터가
선택된 경우, 다른 컨트롤러(예: 레플리케이션 컨트롤러)와 해당 파드는
예측할 수 없는 방식으로 작동할 수 있다.  쿠버네티스는 당신이 `.spec.selector` 를 지정할 때
발생하는 실수를 막을 수 없을 것이다.

다음은 이 기능을 사용하려는 경우의 예시이다.

잡 `old` 가 이미 실행 중이다.  기존 파드가 계속
실행되기를 원하지만, 잡이 생성한 나머지 파드에는 다른
파드 템플릿을 사용하고 잡으로 하여금 새 이름을 부여하기를 원한다.
그러나 관련된 필드들은 업데이트가 불가능하기 때문에 잡을 업데이트할 수 없다.
따라서 `kubectl delete jobs/old --cascade=orphan` 명령을 사용해서
잡 `old` 를 삭제하지만, _파드를 실행 상태로 둔다_.
삭제하기 전에 어떤 셀렉터를 사용하는지 기록한다.

```shell
kubectl get job old -o yaml
```

출력 결과는 다음과 같다.

```yaml
kind: Job
metadata:
  name: old
  ...
spec:
  selector:
    matchLabels:
      controller-uid: a8f3d00d-c6d2-11e5-9f87-42010af00002
  ...
```

그런 이후에 이름이 `new` 인 새 잡을 생성하고, 동일한 셀렉터를 명시적으로 지정한다.
기존 파드에는 `controller-uid=a8f3d00d-c6d2-11e5-9f87-42010af00002`
레이블이 있기에 잡 `new` 에 의해서도 제어된다.

시스템이 일반적으로 자동 생성하는 셀렉터를 사용하지 않도록 하기 위해
새 잡에서 `manualSelector: true` 를 지정해야 한다.

```yaml
kind: Job
metadata:
  name: new
  ...
spec:
  manualSelector: true
  selector:
    matchLabels:
      controller-uid: a8f3d00d-c6d2-11e5-9f87-42010af00002
  ...
```

새 잡 자체는 `a8f3d00d-c6d2-11e5-9f87-42010af00002` 와 다른 uid 를 가지게 될 것이다.
`manualSelector: true` 를 설정하면 시스템에게 사용자가 무엇을 하는지 알고 있으며 
이런 불일치를 허용한다고 알릴 수 있다.

### 파드 실패 정책{#pod-failure-policy}

{{< feature-state for_k8s_version="v1.26" state="beta" >}}

{{< note >}}
잡(Job)에 대한 파드 실패 정책은
`JobPodFailurePolicy` [기능 게이트](/ko/docs/reference/command-line-tools-reference/feature-gates/)가
클러스터에서 활성화됐을 경우에만 구성할 수 있다. 추가적으로, 
파드 장애 정책의 파드 중단 조건 (참조:
[파드 중단 조건](/ko/docs/concepts/workloads/pods/disruptions#pod-disruption-conditions))을 
감지하고 처리할 수 있도록 `PodDisruptionConditions` 기능 게이트를 활성화하는 것을 권장한다. 두 기능 게이트 모두
쿠버네티스 {{< skew currentVersion >}}에서 사용할 수 있다.
{{< /note >}}

`.spec.podFailurePolicy` 필드로 정의되는 파드 실패 정책은, 클러스터가
컨테이너 종료 코드와 파드 상태를 기반으로 파드의 실패를 
처리하도록 활성화한다.

어떤 상황에서는, 파드의 실패를 처리할 때 잡(Job)의 `.spec.backoffLimit`을 기반으로 하는 
[파드 백오프(backoff) 실패 정책](#pod-backoff-failure-policy)에서 
제공하는 제어보다 더 나은 제어를 원할 수 있다. 다음은 사용 사례의 몇 가지 예시다.
* 불필요한 파드 재시작을 방지하여 워크로드 실행 비용을 최적화하려면, 
  파드 중 하나가 소프트웨어 버그를 나타내는 종료 코드와 함께 실패하는 즉시 
  잡을 종료할 수 있다.
* 중단이 있더라도 잡이 완료되도록 하려면, 
  중단(예: {{< glossary_tooltip text="선점(preemption)" term_id="preemption" >}},
  {{< glossary_tooltip text="API를 이용한 축출(API-initiated Eviction)" term_id="api-eviction" >}}
  또는 축출 기반 {{< glossary_tooltip text="테인트(Taints)" term_id="taint" >}})으로 인한 
  파드 실패를 무시하여 `.spec.backoffLimit` 재시도 한도에 포함되지 않도록 할 수 있다.

위의 사용 사례를 충족하기 위해 
`.spec.podFailurePolicy` 필드에 파드 실패 정책을 구성할 수 있다.
이 정책은 컨테이너 종료 코드 및 파드 상태를 기반으로 파드 실패를 처리할 수 있다.

다음은 `podFailurePolicy`를 정의하는 잡의 매니페스트이다.

{{< codenew file="/controllers/job-pod-failure-policy-example.yaml" >}}

위 예시에서, 파드 실패 정책의 첫 번째 규칙은 `main` 컨테이너가 42 종료코드와 
함께 실패하면 잡도 실패로 표시되는 것으로 
지정한다. 다음은 구체적으로 `main` 컨테이너에 대한 규칙이다.

- 종료 코드 0은 컨테이너가 성공했음을 의미한다.
- 종료 코드 42는 **전체 잡**이 실패했음을 의미한다.
- 다른 모든 종료 코드는 컨테이너 및 전체 파드가 실패했음을 
  나타낸다. 재시작 횟수인 `backoffLimit`까지 파드가 
  다시 생성된다. 만약 `backoffLimit`에 도달하면 **전체 잡**이 실패한다.

{{< note >}}
파드 템플릿이 `restartPolicy: Never`로 지정되었기 때문에,
kubelet은 특정 파드에서 `main` 컨테이너를 재시작하지 않는다.
{{< /note >}}

`DisruptionTarget` 컨디션을 갖는 실패한 파드에 대해 
`Ignore` 동작을 하도록 명시하고 있는 파드 실패 정책의 두 번째 규칙으로 인해, 
`.spec.backoffLimit` 재시도 한도 계산 시 파드 중단(disruption)은 횟수에서 제외된다.

{{< note >}}
파드 실패 정책 또는 파드 백오프 실패 정책에 의해 잡이 실패하고,
잡이 여러 파드를 실행중이면, 쿠버네티스는 아직 보류(Pending) 또는 
실행(Running) 중인 해당 잡의 모든 파드를 종료한다.
{{< /note >}}

다음은 API의 몇 가지 요구 사항 및 의미이다.
- 잡에 `.spec.podFailurePolicy` 필드를 사용하려면,
  `.spec.restartPolicy`가 `Never`로 설정된 잡의 파드 템플릿 또한 정의해야 한다.
- `spec.podFailurePolicy.rules`에 기재한 파드 실패 정책 규칙은 기재한 순서대로 평가된다. 
  파드 실패 정책 규칙이 파드 실패와 매치되면 나머지 규칙은 무시된다. 
  파드 실패와 매치되는 파드 실패 정책 규칙이 없으면 
  기본 처리 방식이 적용된다.
- `spec.podFailurePolicy.rules[*].containerName`에 컨테이너 이름을 지정하여 파드 실패 규칙을 특정 컨테이너에게만 제한할 수 있다. 
  컨테이너 이름을 지정하지 않으면 파드 실패 규칙은 모든 컨테이너에 적용된다. 
  컨테이너 이름을 지정한 경우, 
  이는 파드 템플릿의 컨테이너 또는 `initContainer` 이름 중 하나와 일치해야 한다.
- 파드 실패 정책이 `spec.podFailurePolicy.rules[*].action`과 일치할 때 취할 동작을 지정할 수 있다.
  사용 가능한 값은 다음과 같다.
  - `FailJob`: 파드의 잡을 `Failed`로 표시하고 
    실행 중인 모든 파드를 종료해야 함을 나타낸다.
  - `Ignore`: `.spec.backoffLimit`에 대한 카운터가 증가하지 않아야 하고 
    대체 파드가 생성되어야 함을 나타낸다.
  - `Count`: 파드가 기본 방식으로 처리되어야 함을 나타낸다.
      `.spec.backoffLimit`에 대한 카운터가 증가해야 한다.

### 종료자(finalizers)를 이용한 잡 추적

{{< feature-state for_k8s_version="v1.26" state="stable" >}}

{{< note >}}
컨트롤 플레인을 1.26으로 업그레이드하더라도,
기능 게이트 `JobTrackingWithFinalizers`가 비활성화되어 있을 때 생성된 잡이라면,
컨트롤 플레인은 종료자를 사용하는 잡을 추적하지 않는다.
{{< /note >}}

컨트롤 플레인은 잡에 속한 파드들을 계속 추적하고,
API 서버로부터 제거된 파드가 있는지에 대해 알려준다. 이를 위해 잡 컨트롤러는
`batch.kubernetes.io/job-tracking` 종료자를 갖는 파드를 생성한다.
컨트롤러는 파드가 잡 상태로 처리된 이후에만 종료자를 제거하여,
다른 컨트롤러나 사용자가 파드를 제거할 수 있도록 한다.

쿠버네티스를 1.26으로 업그레이드하기 전이나, 기능 게이트
`JobTrackingWithFinalizers`를 활성화시키기 전에 생성한 잡은 파드 종료자를 사용하지 않고
추적된다.
잡 {{< glossary_tooltip term_id="controller" text="컨트롤러" >}}는
클러스터에 존재하는 파드들에 대해서만 `succeded`와 `failed` 파드들에 대한 상태 카운터를 갱신한다.
만약 파드가 클러스터에서 제거된다면,
컨트롤 플레인은 잡의 진행 상황을 제대로 추적하지 못할 수 있다.

잡이 `batch.kubernetes.io/job-tracking` 어노테이션을 가지고 있는지 확인함으로써
컨트롤 플레인이 파드 종료자를 사용하여 잡을 추적하고 있는지 알 수 있다.
따라서 잡으로부터 이 어노테이션을 수동으로 추가하거나 제거해서는 **안 된다**.
그 대신, 파드 종료자를 사용하여
추적이 가능한 잡을 재생성하면 된다.

## 대안

### 베어(Bare) 파드

파드가 실행 중인 노드가 재부팅되거나 실패하면 파드가 종료되고
재시작되지 않는다.  그러나 잡은 종료된 항목을 대체하기 위해 새 파드를 생성한다.
따라서, 애플리케이션에 단일 파드만 필요한 경우에도 베어 파드 대신
잡을 사용하는 것을 권장한다.

### 레플리케이션 컨트롤러

잡은 [레플리케이션 컨트롤러](/ko/docs/concepts/workloads/controllers/replicationcontroller/)를 보완한다.
레플리케이션 컨트롤러는 종료하지 않을 파드(예: 웹 서버)를 관리하고, 잡은 종료될 것으로
예상되는 파드(예: 배치 작업)를 관리한다.

[파드 라이프사이클](/ko/docs/concepts/workloads/pods/pod-lifecycle/)에서 설명한 것처럼, `잡` 은 *오직*
`OnFailure` 또는 `Never` 와 같은 `RestartPolicy` 를 사용하는 파드에만 적절하다.
(참고: `RestartPolicy` 가 설정되지 않은 경우에는 기본값은 `Always` 이다.)

### 단일 잡으로 컨트롤러 파드 시작

또 다른 패턴은 단일 잡이 파드를 생성한 후 다른 파드들을 생성해서 해당 파드들에
일종의 사용자 정의 컨트롤러 역할을 하는 것이다.  이를 통해 최대한의 유연성을 얻을 수 있지만,
시작하기에는 다소 복잡할 수 있으며 쿠버네티스와의 통합성이 낮아진다.

이 패턴의 한 예시는 파드를 시작하는 잡이다. 파드는 스크립트를 실행해서
스파크(Spark) 마스터 컨트롤러 ([스파크 예시](https://github.com/kubernetes/examples/tree/master/staging/spark/README.md)를 본다)를 시작하고,
스파크 드라이버를 실행한 다음, 정리한다.

이 접근 방식의 장점은 전체 프로세스가 잡 오브젝트의 완료를 보장하면서도,
파드 생성과 작업 할당 방법을 완전히 제어하고 유지한다는 것이다.

## {{% heading "whatsnext" %}}

* [파드](/ko/docs/concepts/workloads/pods)에 대해 배운다.
* 다른 방식으로 잡을 구동하는 방법에 대해서 읽는다.
  * [작업 대기열을 사용한 거친 병렬 처리](/ko/docs/tasks/job/coarse-parallel-processing-work-queue/)
  * [작업 대기열을 사용한 정밀 병렬 처리](/ko/docs/tasks/job/fine-parallel-processing-work-queue/)
  * [병렬 처리를 위한 정적 작업 할당으로 인덱스된 잡](/ko/docs/tasks/job/indexed-parallel-processing-static/) 사용
  * 템플릿 기반으로 복수의 잡 생성: [확장을 사용한 병렬 처리](/ko/docs/tasks/job/parallel-processing-expansion/)
* [완료된 잡을 자동으로 정리](#clean-up-finished-jobs-automatically) 섹션 내 링크를 따라서
  클러스터가 완료되거나 실패된 태스크를 어떻게 정리하는지에 대해 더 배운다.
* `Job`은 쿠버네티스 REST API의 일부이다.
  잡 API에 대해 이해하기 위해
  {{< api-reference page="workload-resources/job-v1" >}}
  오브젝트 정의를 읽는다.
* 스케줄을 기반으로 실행되는 일련의 잡을 정의하는데 사용할 수 있고, 유닉스 툴 `cron`과 유사한
  [`CronJob`](/ko/docs/concepts/workloads/controllers/cron-jobs/)에 대해 읽는다.
* 단계별로 구성된 [예제](/docs/tasks/job/pod-failure-policy/)를 통해,
  `podFailurePolicy`를 사용하여 재시도 가능 및 재시도 불가능 파드의 실패 처리를 하기위한 구성 방법을 연습한다.

# [김태헌] 1/29 CKA STUDY 발표 자료

# 이전 내용 - Pod

Pod는 컨테이너를 쿠버네티스 환경에서 실행할 수 있는 가장 작은 단위입니다.

---

# Controller

## Controller란?

- 뭔가를 제어하는 역할을 하는 것을 의미
- Kubernetes에서의 Controller는 Pod의 개수를 보장해주는 구성요소
    - 즉 특정 애플리케이션을 실행해주는 Pod를 몇 개 운영할 것인지를 결정하고 보장
- Pod의 개수를 보장하기 위한 과정(ex. node가 3개, nginx 이미지)
    1. kube-api가 etcd에서 정보를 가져와서 scheduler에게 요청
    2. scheduler가 node1, node2,node3에다가 컨테이너를 배치하라고 시킴
    
    → 여기서 controller의 역할은 scheduler가 node1, node2에게 시키기전에 nginx 서버 3개를 보장하라고 시키게 되고, 컨트롤러는 3개의 nginx가 잘 작동하고 있는지 확인하고 문제가 생기면 api에게 응답으로 주게 됩니다.
    

**즉, api, etcd, scheduler, controller가 함께 협업하면서 pod의 개수를 보장합니다.**

## Controller의 종류

![Untitled](%5B%E1%84%80%E1%85%B5%E1%86%B7%E1%84%90%E1%85%A2%E1%84%92%E1%85%A5%E1%86%AB%5D%201%2029%20CKA%20STUDY%20%E1%84%87%E1%85%A1%E1%86%AF%E1%84%91%E1%85%AD%20%E1%84%8C%E1%85%A1%E1%84%85%E1%85%AD%20c1ec557178cd471eac9ff8478fd2fbfa/Untitled.png)

1. Replication Controller
    1. Controller의 가장 basic한 구조
    2. 제일 초기인 Kubernetes Version1에서 만들어짐
2. Replicaset
    1. Replication Controller를 운영하다가 부족하다고 느껴서 만듬
    2. Replicaset의 부모역할을 하는 것이 Deployment
3. Daemonset
4. Statefulset
5. Job

---

# Replication Controller

## Replication Controller의 특징

- Controller 본연의 역할을 제일 잘 수행하는 Controller
- 요구하는 Pod의 개수를 보장하며 파드 집합의 실행을 항상 안정적으로 유지하는 것을 목표
    - 요구하는 Pod의 개수가 부족하면 template를 이용해 Pod를 추가
        - 없다면 pod template 사용해서 만듬
    - 요구하는 Pod 수보다 많으면 최근에 생성된 Pod를 삭제(Terminate)
- 기본 구성(ReplicationController안에 들어있는 템플릿)
    - selector
    - replicas
    - template

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
 name: <RC_이름>
spec:
	replicas: <배포갯수>
	selector:
		key: value
	template:
		<컨테이너 템플릿>
```

## Replication Controller가 필요한 이유

먼저 단일 포드가 있는 경우, 만약 어떤 이유로 인해 애플리케이션이 충돌하고 Pod가 실패하면 사용자는 더 이상 다음을 수행할 수 없게 됩니다.

사용자가 애플리케이션에 대한 액세스 할 수 있도록 하려면 동시에 작동하는 인스턴스를 두 개 이상 갖고 있어야 합니다.

---

만약에 Node에 Pod를 실행하고 있는데 수요가 증가하여 첫번째 노드의 리소스가 부족해지면 클러스터의 다른 노드에 추가 부분을 배포해야 한다.

ReplicationController는 클러스터의 여러 노드에 걸쳐 있다. 그래서 서로 다른 노드의 여러 경로에 걸쳐 LoadBalancing을 하고 Scale Out을 하는데 도움이 된다.

---

1. **Replication Controller는 단일 Pod가 multiple instatnces를 실행하는 것을 도와주어서 High availability를 제공해줍니다.**
    
    (Replication Controller를 사용한다고 single pod를 사용할 수 없는 것은 아님)
    
2. Replication Controller는 **Load Balancing과 Scaling**을 할 수 있게 해줍니다.

### 예상 시나리오 1

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: rc-nginx
spec:
  replicas: 3
  selector:
    app: webui
  template:
    metadata:
      name: nginx-pod
      labels:
        app: webui
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.14
```

→ ReplicationController는 selector를 보고 key, value가 app, webui인 것을 확인해서 replicas의 개수에 맞게 pod가 동작하고 있는지 확인합니다.

만약에 여기서 pod의 개수가 3개보다 작다면 template의 label에서 app: webui를 확인하여 pod를 새로 만들어줍니다.

### Command

현재 replicationcontroller에 해당하는 pod 확인하기(아래 3가지 명령어)

`kubectl get replicationcontrollers`

`kubectl get rc`

`kubectl describe rc rc-nginx`

yaml을 직접 작성하지 않고 실행하는 방법(default 버전으로 생성됨)

`kubectl run redis --image=redis --labels=app=webui --dry-run > redis.yaml`

### 예상 시나리오 2

- 만약에 replicas=3으로  label이 app: webui인 것을 실행하고 있을 때, 새로운 redis.yaml을 하나 만들어서, label을 app: webui로 등록하여 새로운 pod를 실행한다면 어떻게 될까요?
    
    STATUS가 바로 Terminating으로 실행이 되지 않는다. 또 실행되려고 하니까 controller가 바로 죽여버린다.
    

### 예상 시나리오 3

- 현재 3개가 잘 돌아가고 있는데 editor로 이미지를 nginx:1.14에서 nginx:1.15로 바꾼다면 어떤 일이 일어날까요
    
    아무런 변화도 없다. RestController는 select를 확인하여 label이 바뀌지 않았기 때문에 아무런 변화가 없다. (3개가 잘 돌아가고 있다고 생각함)
    

---

# ReplicaSet

## ReplicaSet의 특징

- ReplicationController와 같은 역할을 하는 Controller
(ReplicationController와 동일하게 pod의 개수는 보장한다)
- ReplicationController보다 풍부한 selector를 제공
(matchLabels, matchExpressions)

```yaml
selector:
	matchLabels:
		component: redis
	matchExpressions:
		- {key: tier, operator: In, values: [cache]}
		- {key: environment, operator: Notln, values: [dev]}
```

- Pod를 죽이면 다시 만들어지지만, controller 삭제하면 pod까지 같이 삭제됨.
    
    `kubectl delete rc rc-nginx`
    
- Pod는 죽이지 않고 Controller만 삭제하려면 어떻게 해야할까요?
    
    `kubectl delete rs rs-nginx --cascade=false`
    

### matchExpressions 연산자 특징

- 4가지 형태의 Expression을 제공해주고 있다.
- In: key와 values를 지정하여 key, value가 일치하는 Pod만 연결
- NotIn: key는 일치하고 value는 일치하지 않는 Pod에 연결
- Exists : key에 맞는 label의 pod를 연결(존재하기만 하면 됨)
- DoesNotExist : key와 다른 label의 pod를 연결(존재하지 않기만 하면 됨)

### 사용 예시

다음 두개의 식은 거의 동일하지만, 아래 ReplicaSet은 version 2.2도 허용한다.

```yaml
# ReplicationController
apiVersion: v1
kind: ReplicationController
metadata:
	name: rc-nginx
spec:
	replicas: 3
	selector:
		app: webui
		version: "2.1"
	template:
		metadata: nginx-pod
		labels:
			app: webui
		spec:
			containers:
			- name: nginx-container
			  imtage: nginx:1.14
```

```yaml
#ReplicaSet
apiVersion: apps/v1
kind: ReplicaSet
metadata:
	name: rs-nginx
spec:
	replicas: 3
	selector:
		matchLabels:
			app: webui
		matchExpressions:
		- {key: version: 2.1, operator: In, values: ["2.1", "2.2"]}
	template:
		metadata: nginx-pod
		labels:
			app: webui
		spec:
			containers:
			- name: nginx-container
			  imtage: nginx:1.14
```

## ReplicationController vs ReplicaSet

- 동일한 목적을 갖고 있지만 동일하지 않습니다.
- ReplicationController는 ReplicaSet으로 대체되는 오래된 기술

```yaml
# rc-definition.yml
apiVersion: v1
kind: ReplicationController
metadata: # ReplicationController용 metadata 섹션
		name: myapp-rc
		labels:
			app: myapp
			type: front-end
spec:
	template: # Pod 정의 파일을 template 섹션으로 이동
		metadata:
			name: myapp-pod
			labels:
				app: myapp
				type: front-end
		spec: # Pod용 섹션
			containers:
			- name: nginx-container
				image: nginx
replicas: 3
```

```yaml
# rs-definition.yml
apiVersion: apps/v1
kind: ReplicationSet
metadata: 
		name: myapp-replicaset
		labels:
			app: myapp
			type: front-end
spec:
	template: # Pod 정의 파일을 template 섹션으로 이동
		metadata:
			name: myapp-pod
			labels:
				app: myapp
				type: front-end
		spec:
			containers:
			- name: nginx-container
				image: nginx
replicas: 3
selector:
	matchLabels:
		type: front-end
```

- ReplicaController와 ReplicaSet의 최대 차이는 `selector` 에서 나옵니다.
- `matchLabels` 는 지정된 레이블의 Pod의 레이블과 일치시켜야 합니다.
- selector는 다양한 옵션을 제공합니다.

### Labels and Selectors

- Pod의 개수 보장은 Pod의 Label을 보고 모니터링할 Pod를 결정

### Scale

Pod의 수를 3개에서 6개로 증가시키려고 하면 여러 방법이 있다.

1. `replicas` 를 3에서 6으로 증가시킴(에디터를 이용합니다)
`kubectl replace -f replicaset-definition.yml` 을 적용
2. kube Control scale 명령 실행하는 방법
`kubectl scale —-replicas=6 -f replicaset-definition.yml`
    
    or
    
    `kubectl scale --replicase=6 replicaset myapp-replicaset`
    

### Command

1. `kubectl create -f replicaset-definition.yml` : ReplicaSet이나 file input에 따라 객체를 생성하는데 사용
→ `-f` 매개변수를 사용하여 입력할 파일을 제공
2. `kubectl get replicaset` : 생성된 replicaset을 확인
3. `kubectl delete replicaset` : replicaset을 삭제
4. `kubectl replace -f replicaset-definition.yml` : replicaset을 교체하거나 업데이트
5. `kubectl scale --replicas=6 -f replicaset-definition.yml` : 파일 수정하지 않고 command line에서 간단히 replicaset 크기를 조정하는 명령
6. `replicaset edit replicaset myapp-replicaset` : 텍스트 편집기로 수정
7. `kubectl scale replicaset myapp-replicaset --replicas=2`

---

# Deployment

## Deployment의 특징

- DEPLOYMENT : Replicaset을 제어해주는 부모 역할
- 목적 : Rolling Update & Rolling Back
- ReplicaSet을 컨트롤해서 Pod 수를 조절
- 생성과정
    1. Deploy 하나를 만들게 되면, Deploy가 ReplicaSet 하나를 create
    2. ReplicaSet에게 실행시킬 Pod의 버전과 Replica 수를 할당해주면 ReplicaSet은 앞서 Controller가 동작했던 방식으로 Pod 수를 보장
- **즉, Deploy가 ReplicaSet을 컨트롤하게 되고, ReplicaSet이 Pod를 컨트롤하게 되는 방식**

## Rolling Update란?

- Rolling Update는 Pod 인스턴스를 점진적으로 새로운 것으로 업데이트하여 Deployment 업데이트가 **서비스 중단 없이** 이루어질 수 있도록 해주는 것입니다.
- Deploy는 Rolling Update를 위해서 만들어진 API resource라고도 할 수 있습니다.
- 동작 방식
    1. Deploy를 동작시키면 Deploy가 ReplicaSet을 통해서 Application(예를 들어 nginx-1.14)을 실행
    2. Deploy에게 ‘1.15버전으로 롤링업데이트해줘’ 라고 하면 
    3. Deploy는 새로운 ReplicaSet을 만들어서 1.15버전으로 버전 업데이트를 해줍니다.

→ 즉, 서비스 운영 중에 업데이트 해주고 고객은 이 사실을 모르게 됩니다.

## Rolling Update의 과정

`kubectl set image deployment <deploy_name> <container_name>=<new_version_image>`

`kubectl set image deployement app-deploy app=nginx:1.15 --record`

→ 과정은 이렇게 된다. 새로운 ReplicaSet에 Pod를 하나씩 만들면서 새로 생긴다면 원래의 ReplicaSet의 Pod를 하나씩 줄이면서 새로운 ReplicaSet을 생성하게 됩니다.
(몇 개를 유지할 것인지는 maxSurge와 maxUnavailable에 따릅니다)

### 예상 시나리오

```yaml
# deployment-definition.yaml
apiVersion: apps/v1
kind: Deployment
metadat:
  name: deploy-nginx
spec:
  progressDeadlineSeconds: 600
  revisionHistoryLimit: 10
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  replicas: 4
  selector:
    matchLabels:
      app: web 
...
  
```

- `progressDeadlineSeconds` 는 600초동안 업데이트를 진행하지 못하면 방금 전에 실행했던 update를 취소하겠다는 뜻
- `maxSurge: 25%` : 4 + 4*0.25 = 5 ⇒ update를 할 때 6개가 Running 중이면 하나를 삭제하겠다는 의미입니다. 즉, running 중인것을 5개까지만 유지하겠다는 것입니다.
    - 이렇게 되면 update를 6개까지 하고, 2개를 한번에 죽이는 과정이 이루어집니다.
- `maxUnavailable : 25%` : terminate하는 것을 조절하는 것입니다.
    - 새로운 버전으로 업데이트하는 동안 현재 버전의 Pod 중 최대 25%까지는 동시에 비활성화될 수 있습니다. 즉, 1개까지는 비활성화될 수 있다는 의미입니다.
- 이런 명령어를 통해 얼마나 빠르게 update할 수 있는지 customize할 수 있습니다.

### command

1. `kubectl create -f deployment-definition.yml`
2. `kubectl get deployments`
3. `kubectl get replicaset`
4. `kubectl get pods`
5. 생성된 객체를 모두 보고 싶을 때 : `kubectl get all`
6. `kubectl create deployment httpd-frontend --image=httpd:2.4-alpine --replicas=3 deployment.apps/httpd-frontend created`

## Rollout

업데이트를 일시정지하거나 재시작할 때 사용합니다.

`kubectl rollout pause deployment app-deploy` : update 도중 일시정지

`kubectl rollout resume deployment app-deploy`: update 재시작

### kubectl rollout undo

- 아무것도 입력 없을 때 : 업데이트 롤백 커맨드(history 기준 한 단계 전)
- 롤백은 `undo` 명령어를 사용합니다.
    
    `kubectl rollout undo deployment app-deploy` 
    
    **`—to-revision=[REVISION 값]` 옵션**
    
- [REVISION 값] 으로 롤백
    
    `kubectl rollout undo deployment app-deploy --to-revision=3`
    

## Command

- `kubectl create -f deployment-definition.yml`
- `kubectl get deployments`
- `kubectl apply -f deployment-definition.yml`
- `kubectl set image deployment/myapp-deployment nginx=nginx:1.9.1`
- `kubectl rollout status deployment/myapp-deployment`
- `kubectl rollout history deployment/myapp-deployment`
- `kubectl rollout undo deployment/myapp-deployment`
- `kubectl set image deployment myapp-deployment nginx=nginx:1.19-perl --record`
- `kubectl get pods`
- `kubectl rollout undo deployment/myapp-deployment`

# Basic Network of Kubernetes

쿠버네티스에서는 IP 주소가 Pod에 할당되며, 각 Pod는 자체 내부 IP 주소를 가집니다. 이 IP 주소는 Pod 간 통신을 위해 사용됩니다. 다음은 쿠버네티스에서의 IP 주소 할당과 관련된 몇 가지 주요 개념입니다.

## **단일 노드 쿠버네티스 클러스터**

쿠버네티스 클러스터에서 하나의 노드만 있는 경우, 해당 노드는 클러스터에 유일한 IP 주소를 가지게 됩니다. 이 IP 주소는 쿠버네티스 노드에 접근할 때 사용됩니다.

## **Pod의 IP 주소**

Pod는 독립된 네트워크 공간에서 실행되며, 각 Pod는 자체적인 IP 주소를 할당 받습니다. 이 IP 주소는 Pod 내부에서 사용되며, 해당 Pod에서 실행 중인 컨테이너들은 이 내부 IP 주소를 통해 서로 통신합니다.

## **사설 네트워크**

쿠버네티스가 처음 설정될 때, 클러스터는 사설 네트워크를 만들게 됩니다. 이는 각 노드 및 Pod에 대한 내부 통신을 위한 격리된 네트워크 환경을 제공합니다.

이 IP를 통해 Pod는 서로 통신할 수 있지만, 내부 IP주소를 이용해 다른 Pod에 접근하는 것은 좋은 방법이 아닙니다. 왜냐하면 Pod가 recreate되면 바뀌기 때문입니다.

## **다중 Pod 및 IP 할당**

다수의 Pod를 배포하면 각 Pod는 네트워크에서 할당된 개별적인 IP 주소를 갖게 됩니다. 이를 통해 다수의 Pod 간에 독립적인 통신이 가능합니다.

## **주의사항**

내부 IP 주소를 이용하여 다른 Pod에 직접적으로 접근하는 것은 권장되지 않습니다. 왜냐하면 Pod가 재생성되면 IP 주소가 변경될 수 있기 때문입니다. 대신, 서비스(Service) 등을 이용하여 Pod에 접근하는 것이 안정적인 방법입니다.

### 외부 통신

- 외부 사용자가 웹 페이지에 접속하는 경우:
    - 쿠버네티스 노드의 IP 주소가 192.168.1.2이며, 노트북은 같은 네트워크에 있고 192.168.1.10입니다.
    - 내부 Pod 네트워크는 10.244.0.0이며, Pod의 IP 주소는 10.244.0.2입니다.
    - 10.244.0.2는 분리된 네트워크이기 때문에 직접적인 접속이 불가능합니다.
- 웹 페이지를 볼 수 있는 옵션:
    - 192.168.1.2의 쿠버네티스 노드에 SSH로 접속하여 curl을 통해 Pod의 웹 페이지에 접근할 수 있습니다.
    - 하지만 이는 쿠버네티스 노드 내부에서만 가능한 방법으로 원하는 것이 아닙니다.
    - 원하는 것은 쿠버네티스 노드로부터 직접적인 접속 없이 노트북에서 쿠버네티스 노드의 IP를 통해 웹 서버에 접근하는 것입니다.
    - 이를 위해 쿠버네티스 서비스가 중간에서 도움을 주어 노트북에서 노드를 통해 웹 컨테이너가 동작 중인 Pod로 요청을 매핑할 수 있도록 돕습니다.

---

# Service

## Kubernetes 서비스의 의미

서비스 = 쿠버네티스 네트워크

### Kubernetes Service 개념

- 동일한 서비스를 제공하는 Pod 그룹의 단일 진입점을 제공
    - [시나리오] 3개의 동일한 노드가 진행되고 있을 때, 하나의 노드에서 진행되는게 아니라 1번, 2번, 3번 한번씩 이런식으로 진행하기 위해서는 Service가 필요
    - Service라는 API가 있어서 API Service 구조로 쿠버네티스에게 명령을 줍니다.
    → ‘쿠버야 webui 파드들을 하나의 IP로 묶어서 관리해줘!’(단일 진입점을 만들어줘)’
    - 그럼 쿠버네티스는 pod들의 label을 보고, 하나로 묶어서 virtual한 ip하나를 만들어줍니다.
    - 이렇게 만들어진 ip address를 단일 진입점이라 하고, 3개 중의 하나로 연결해주는 **로드밸런서 IP** 가 생성되는 것이다.
    - 이 정보를 etcd에다가 기록하게 해주는 역할을 하는것이 service API 역할입니다.
- 실제로 이런 Service API를 구현한 Definition

```yaml
# Deployment-definition
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webui
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webui
  template:
    metadata:
      name: nginx-pod
      labels:
        app: webui
    spec:
      containers:
      - name: nginx-container
        image: nginx
      
```

```yaml
# Service-definition
apiVersion: v1
kind: Service
metadata:
  name: webui-svc
spec:
  clusterIP: 10.96.100.100 # VirtualIP(LoadBalancerIP) -> 생략해도 되고 써도 된다.
  selector:
    app: webui
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

![Untitled](%5B%E1%84%80%E1%85%B5%E1%86%B7%E1%84%90%E1%85%A2%E1%84%92%E1%85%A5%E1%86%AB%5D%201%2029%20CKA%20STUDY%20%E1%84%87%E1%85%A1%E1%86%AF%E1%84%91%E1%85%AD%20%E1%84%8C%E1%85%A1%E1%84%85%E1%85%AD%20c1ec557178cd471eac9ff8478fd2fbfa/Untitled%201.png)

## Kubernetes 서비스 타입

### 4가지 Type 지원

## ClusterIP(default)

- Pod 그룹의 단일 진입점(Virtual IP) 생성
- 가장 basic한 service 구조 만들어줌
- 서비스가 클러스터 내부에서 가상 IP 주소를 할당받습니다.
- 클러스터 내의 다른 서비스나 파드는 해당 가상 IP 주소를 사용하여 서비스에 접근할 수 있습니다.
- 외부에서는 이 가상 IP를 통한 직접적인 접근이 불가능합니다.

## NodePort

![Untitled](%5B%E1%84%80%E1%85%B5%E1%86%B7%E1%84%90%E1%85%A2%E1%84%92%E1%85%A5%E1%86%AB%5D%201%2029%20CKA%20STUDY%20%E1%84%87%E1%85%A1%E1%86%AF%E1%84%91%E1%85%AD%20%E1%84%8C%E1%85%A1%E1%84%85%E1%85%AD%20c1ec557178cd471eac9ff8478fd2fbfa/Untitled%202.png)

- 서비스가 클러스터의 각 노드에서 지정된 포트로 노출됩니다.
- 노드의 IP 주소와 할당된 포트를 통해 외부에서 서비스에 접근할 수 있습니다.
- 외부 트래픽이 노드포트로 유입되면 해당 노드에 있는 서비스로 포워딩됩니다.
    - ClusterIP가 생성된 후, 모든 Worker Node에 외부에서 접속가능한 포트가 예약
    - 각각의 노드의 port를 열어줌 → client 사용자가 connection 요청하면 해당하는 곳에서 또 로드밸런서 서비스로 3개 중 하나로 연결시켜줌

```jsx
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: NodePort
  ports:
    - targetPort: 80
      port: 80
      nodePort: 30008
  selector:
    app: myapp
    type: front-end
```

- `kubectl create -f service-definition.yml`
- `kubectl get services`
- `curl [http://192.168.1.2:30008](http://192.168.1.2:30008)` → 웹서버 access

## LoadBalancer

- 클라우드 인프라스트럭처(AWS, Azure, GCP 등)나 오픈스택 클라우드에 적용
- LoadBalancer를 자동으로 프로 비전하는 기능 지원
- LB 장비가 생기면서 연결된다. → 실제 없었던 것이 생기는 게 아니라 실제 존재하는 장비의 ip address와 NodePort를 연결해주는 것
    - LB 장비는 해당하는 플랫폼에서만 사용할 수 있게 하는 것
- 클라우드 환경에서만 사용 가능한 유형으로, 외부 로드밸런서를 프로비저닝합니다.
- 서비스에 고유한 외부 IP 주소가 할당되며, 외부에서 이 IP 주소를 통해 서비스에 접근할 수 있습니다.
- 클라우드 제공업체의 로드밸런서를 활용하여 트래픽을 클러스터 내부의 서비스로 분산시킵니다.

## (ExternalName)

- 클러스터 안에서 외부에 접속 시 사용할 도메인을 등록해서 사용
- 클러스터 도메인이 실제 외부 도메인으로 치환되어 동작
- 다른 것들과 좀 다름
    - 네이밍 서비스를 제공해준다.
    - 서비스를 만들어주는데 external ip이름을 Google.com이라 등록했다고 해보자.
    - external name default cluster 이름을 입력해주면 도메인이 google.com으로 바껴서 외부 통신을 할 수 있다.
    - DNS 서비스처럼 네이밍 서비스를 Pod 내부에서, cluster 내부에서 사용할 수 있도록 지원해주는 것
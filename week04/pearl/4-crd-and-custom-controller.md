# CRD & Custom Controller (2025 updated)
이 파트는 쿠버네티스를 단순히 사용하는 것을 넘어, 확장할 때 발생하는 보안 문제를 다룬다. 해당 문제는 현대 쿠버네티스 운영에서 중요하다.

## 필요성
- 쿠버네티스의 편리함 중 하나는 “확장성”이 좋다는 점이다. 
- 개발자가 직접 새로운 API 리소스(Custom Resource Definition)를 추가하고, 이를 자동화 시스템이 관리하여 클러스터에 배포가 가능해졌는데, 이로 인해 발생하는 문제가 있다.

- 보통, 자동화 시스템(Operator)은 강력한 권한을 가진 경우가 많다. 
- 다른 Pod를 생성하거나, Secret을 읽는 등 강력한 권한이 필요하기 때문이다. 
- 만약, 이 Operator가 공격을 받는다면, 편리함을 제공하는 Custom 확장 기능이 새로운 공격 경로로 변모할 위험이 있다.

- 즉, CRD와 Custom Controller(Operator)가 편리한 만큼, 큰 책임과 새로운 보안 요소를 만든다는 의미이다.


## Custom Resource Definition (CRD) 보안
- CRD는 쿠버네티스에 `Pod`, `Deployment`처럼 커스텀 오브젝트 타입을 정의할 수 있게 해주는 기능
- 예를 들어, `MyWebApp`, `CronJob` (과거) 같은 것들을 직접 만들 수 있다.
- **보안 관점에서의 핵심**
    - CRD 자체에는 보안 설정이 거의 없으므로, **"누가 이 새로운 커스텀 리소스를 생성(create), 조회(get), 수정(update), 삭제(delete)할 수 있는지"** 컨트롤하는 것이 중요하다.
    - 결국 **RBAC 중요성으로** 귀결된다.
    - CRD를 만들면 새로운 `apiGroup`과 `resource`가 생기는데, 이 새로운 리소스에 대한 접근 권한을 `Role`이나 `ClusterRole`로 명시적으로 정의하고 부여해야 한다.



### 실습 포인트 - 1 (CRD에 대한 RBAC 설정)
- mycompany.com 그룹에 MyWebApp이라는 CRD를 만들고, webapp-admin 역할을 가진 사용자만 이 리소스를 관리할 수 있도록 권한 부여하기

간단한 CRD 정의 및 배포

```YAML
# mywebapp-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: mywebapps.mycompany.com
spec:
  group: mycompany.com
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                image:
                  type: string
  scope: Namespaced
  names:
    plural: mywebapps
    singular: mywebapp
    kind: MyWebApp
    shortNames:
    - mwa
```



```Bash
kubectl apply -f mywebapp-crd.yaml
# 이제 kubectl get mwa 명령을 사용할 수 있습니다.
```


- 새로운 CRD를 관리할 Role 생성
- rules 섹션의 apiGroups와 resources에 방금 만든 CRD의 정보를 정확히 기입하는 것이 핵심이다.


```YAML
# mywebapp-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: mywebapp-admin
rules:
- apiGroups: ["mycompany.com"] # CRD에서 정의한 group
  resources: ["mywebapps"]     # CRD에서 정의한 plural name
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```


```Bash
kubectl apply -f mywebapp-role.yaml
```


이제 `mywebapp-admin` Role을 RoleBinding을 통해 특정 사용자나 그룹에 부여하면, 해당 사용자만 `MyWebApp` 리소스를 관리할 수 있게 된다.


---
## Custom Controller & Operator 보안
- Custom Controller(그리고 그 발전된 형태인 Operator)는 CRD 오브젝트의 상태 변화를 감시하고, 원하는 상태(Desired State)로 만들어주는 역할을 하는 프로그램이다.
- 보통 Pod 형태로 실행한다.

- **보안 관점에서의 핵심**
    - **최소 권한의 원칙 (Principle of Least Privilege):** Operator는 클러스터의 상태를 바꾸는 강력한 작업을 수행하므로, **정확히 필요한 권한만** 부여해야 한다.
    - 예를 들어, DB Operator가 `StatefulSet`과 `Secret`을 관리해야 한다면, 그 외에 `Deployment`나 `RoleBinding`을 수정할 권한을 주면 안된다.
    - **Service Account 권한 관리:** Operator는 Pod로 실행되며, 특정 `ServiceAccount`를 사용한다.
    - 이 `ServiceAccount`에 `Role` 또는 `ClusterRole`을 바인딩하여 권한을 부여한다. 이 권한 설정이 Operator 보안의 핵심이다.
    - **공격 대상이 되기 쉬움(Attack Surface):** Operator Pod 자체도 공격 대상이다. 신뢰할 수 있는 이미지를 사용하고, 취약점을 스캔하며, 불필요한 권한을 제거한 `SecurityContext`를 적용해야한다.



### 실습 포인트 - 2 (Operator의 ClusterRole 확인)

  - **시나리오:** 잘 알려진 오픈소스 Operator(예: Prometheus Operator)가 어떤 권한을 요구하는지 `ClusterRole` YAML 파일을 통해 분석하기

<!-- end list -->

1.  **Operator가 사용하는 `ClusterRole` 찾기**

      - 일반적으로 Operator를 설치하면 필요한 `ServiceAccount`, `ClusterRole`, `ClusterRoleBinding`이 함께 생성된다.

    <!-- end list -->

    ```bash
    # 설치된 ClusterRole 목록에서 operator와 관련된 이름을 찾는다.
    kubectl get clusterrole | grep prometheus
    ```

2.  **`ClusterRole`의 상세 내용(YAML) 확인**

      - `kubectl get clusterrole <role-name> -o yaml` 명령으로 어떤 API 그룹의 어떤 리소스에 대해 어떤 동작(verb)을 허용하는지 상세히 분석할 수 있다.

    <!-- end list -->

    ```yaml
    # prometheus-operator의 ClusterRole (예시)
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: prometheus-operator
    rules:
    - apiGroups:
      - monitoring.coreos.com
      resources:
      - alertmanagers
      - prometheuses
      - servicemonitors # 자신의 CRD를 관리할 권한
      verbs:
      - "*"
    - apiGroups:
      - apps
      resources:
      - statefulsets # Prometheus 서버를 StatefulSet으로 배포할 권한
      verbs:
      - "*"
    - apiGroups:
      - ""
      resources:
      - configmaps
      - secrets      # Prometheus 설정을 ConfigMap/Secret으로 만들 권한
      verbs:
      - "*"
    - apiGroups:
      - ""
      resources:
      - pods         # Pod를 관리할 권한
      verbs:
      - list
      - delete
    - apiGroups:
      - ""
      resources:
      - services     # Prometheus UI를 노출할 Service를 만들 권한
      - endpoints
      verbs:
      - get
      - create
      - update
      - delete
    # ... 이하 생략
    ```

<!-- end list -->

  - **Namespace-scoped vs Cluster-scoped Operator:**
      - 가능하면 클러스터 전체가 아닌, 특정 네임스페이스 안에서만 작동하는 **Namespace-scoped Operator**를 사용하여 `ClusterRole` 대신 `Role`을 부여하는 것이 좋다.
      - 이는 만일의 사태 발생 시 피해 범위를 해당 네임스페이스로 제한하는 효과적인 보안 전략이다.
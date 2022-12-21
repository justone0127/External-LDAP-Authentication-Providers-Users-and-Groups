## External (LDAP) Authentication Providers, Users, and Groups

### 1. Configuring External Authentication Providers

OpenShift는 다양한 인증 공급자를 지원하며 ID 공급자 구성 이해에서 전체 목록을 찾을 수 있습니다. 가장 일반적으로 사용되는 인증 공급자 중 하나는 Microsoft Active Directory 또는 기타 소스에서 제공하는 LDAP입니다.



OpenShift는 LDAP 서버에 대해 사용자 인증을 수행할 수 있으며 LDAP 그룹 구성원을 기반으로 그룹 구성원 및 특정 RBAC 속성을 구성할 수도 있습니다.



### 1.1 Background: LDAP Structure

이 환경에서는 다음 사용자 그룹에 LDAP을 제공합니다.

- `ose-user` : OpenShift 액세스 권한이 있는 사용자
  - OpenShift에 로그인할 수 있어야 하는 모든 사용자는 이 그룹의 구성원이어야 합니다.
  - 아래에 언급된 모든 사용자가 이 그룹에 속합니다.
- `ose-normal-dev` : 일반 OpenShift 사용자
  - 특별한 권한이 없는 일반 OpenShift 사용자
  - 포함된 사용자 : `normaluser1`, `teamuser1`, `teamuser2`
- `ose-fancy-dev` : Fancy OpenShift 사용자
  - 일부 특수 권한이 부여된 OpenShift 사용자
  -  포함된 사용자 : `fancyuser1`, `fancyuser2`
- `ose-taemed-app` : Teamed app 사용자
  - 동일한 OpenShift **Project**에 액세스할 수 있는 사용자 그룹
  - 포함된 사용자 : `teamuser1`, `teamuser2`

### 1.2 Examine the OAuth configuration

OpenShift 4 설치이므로 기본 OAuth 리소스가 있습니다. 다음을 사용하여 OAuth 구성을 검사할 수 있습니다.

```bash
oc get oauth cluster -o yaml
```

다음과 같은 내용이 표시됩니다.

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  annotations:
    include.release.openshift.io/ibm-cloud-managed: "true"
    include.release.openshift.io/self-managed-high-availability: "true"
    include.release.openshift.io/single-node-developer: "true"
    release.openshift.io/create-only: "true"
  creationTimestamp: "2022-12-14T09:00:12Z"
  generation: 1
  name: cluster
  ownerReferences:
  - apiVersion: config.openshift.io/v1
    kind: ClusterVersion
    name: version
    uid: 714fb6b7-5645-4b17-9905-fec1ab0b96cc
  resourceVersion: "1738"
  uid: d12229da-ee48-4e30-9ff4-b064b1a9025d
spec: {}
```

여기서 주의할 점이 몇 가지 있습니다. 첫째, 기본적으로 여기에는 아무것도 없습니다. 그러면 `kubeadmin` 사용자는 어떻게 작동합니까? OpenShift OAuth 시스템은 `kube-system` **NameSpace**에서 `kubeadmin` **Secret**을 찾는 것을 알고 있습니다.

```bash
oc get secret -n kube-system kubeadmin -o yaml
```

다음과 같은 내용이 표시됩니다.

```yaml
apiVersion: v1
data:
  kubeadmin: JDJhJDEwJENHb2QvL21mLlQ3amszMi9HNnR0amVnc2I1S3pSTzZQN3ZlakJpcmkvYVV0WU
RyOXlXR0cu
kind: Secret
metadata:
  creationTimestamp: "2022-12-14T08:59:26Z"
  name: kubeadmin
  namespace: kube-system
  resourceVersion: "389"
  uid: 7cbf51ba-5dc8-41c4-8e86-b4d066affb56
type: Opaque
```

해당 **Secret**에는 `kubeadmin` 암호의 인코딩된 해시가 포함되어 있습니다. 이 계정은 새 OAuth를 구성한 후에도 계속 작동합니다. 비활성활하려면 secret을 삭제해야 합니다.



실제 환경에서는 기존 ID 관리 솔루션과 통합하기를 원할것 입니다. 이 실습에서는 LDAP을 `identityProvider`로 구성합니다. 다음은 구성의 예입니다. 다음과 같이 `type: LDAP`인 `identityProviders`에서 요소를 찾습니다.

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - name: ldap (1)
    challenge: false
    login: true
    mappingMethod: claim (2)
    type: LDAP
    ldap:
      attributes: (3)
        id:
        - dn
        email:
        - mail
        name:
        - cn
        preferredUsername:
        - uid
      bindDN: "uid=openshiftworkshop,ou=Users,o=5e615ba46b812e7da02e93b5,dc=jumpcloud,dc=com" (4)
      bindPassword: (5)
        name: ldap-secret
      ca: (6)
        name: ca-config-map
      insecure: false
      url: "ldaps://ldap.jumpcloud.com/ou=Users,o=5e615ba46b812e7da02e93b5,dc=jumpcloud,dc=com?uid?sub?(memberOf=cn=ose-user,ou=Users,o=5e615ba46b812e7da02e93b5,dc=jumpcloud,dc=com)" (7)
  tokenConfig:
    accessTokenMaxAgeSeconds: 86400
```

`identityProviders:` 아래의 몇 가지 주목할만한 필드

1. `name` : 자격 증명 공급자의 고유 ID입니다. OpenShift 환경에 여러 인증 공급자가 있을 수 있으며 OpenShift는 이들을 구별할 수 있습니다. 
2. `mappingMethod: claim` : 이 섹션은 여러 공급자가 구성된 경우 OpenShift 클러스터 내에서 사용자 이름이 할당되는 방법과 관련이 있습니다. 자세한 내용은 [Identity provider parameters](https://docs.openshift.com/container-platform/4.11/authentication/understanding-identity-provider.html#identity-provider-parameters_understanding-identity-provider) 섹션을 참조하십시오.
3. `attributes` : 이 섹션은 반복할 LDAP 필드를 정의하고 OpenShift 사용자의 "계정"에 있는 필드에 할당합니다. 목록을 검색할 때 속성을 찾을 수 없거나 채워지지 않으면 전체 인증이 실패합니다. 이 경우 LDAP `dn`과 연결된 ID, LDAP 메일의 이메일 주소, LDAP `cn`의 이름 및 LDAP `uid`의 사용자 이름을 만듭니다.
4. `bindDN`: LDAP 검색 시 이 사용자로 서버에 바인딩합니다.
5. `bindPassword` : 검색을 위해 바인딩할 때 사용할 암호가 있는 Secret에 대한 참조입니다.
6. `ca` : 서버의 SSL 인증서 유효성을 검사하는 데 사용할 CA 인증서가 포함된 ConfigMap에 대한 참조입니다.
7. `url` : 수행할 LDAP 서버 및 검색을 식별합니다.

OpenShift에서 LDAP 인증의 특정 세부사항에 대한 자세한 정보는 [Configuring an LDAP Identity provider](https://docs.openshift.com/container-platform/4.11/authentication/identity_providers/configuring-ldap-identity-provider.html) 문서를 참조하십시오.

LDAP identity provider를 설정하려면 다음을 수행해야 합니다.

1. bind password로 `Secret`을 생성합니다.
2. CA 인증서로 `ConfigMap`을 만듭니다.
3. LDAP identity provider로 `cluster` `OAuth` object를 업데이트합니다.

`kubeadmin` 사용자로 `oc`를 사용하여 OAuth 구성을 적용합니다.

```bash
oc login -u kubeadmin -p ${PASSWORD}
```

```bash
oc create secret generic ldap-secret --from-literal=bindPassword=b1ndP^ssword -n openshift-config
wget https://ssl-ccp.godaddy.com/repository/gd-class2-root.crt -O /opt/app-root/src/support/ca.crt
oc create configmap ca-config-map --from-file=/opt/app-root/src/support/ca.crt -n openshift-config
oc apply -f /opt/app-root/src/support/oauth-cluster.yaml
```

> 기존 `OAuth` object가 있기 때문에 `apply`를 사용합니다. `create`를 사용한 경우 object가 이미 존재한다는 오류가 발생합니다. 여전히 경고가 표시되지만 괜찮습니다.

이렇게 하면 OAuth Operator의 재배포가 트리거됩니다.

```bash
oc rollout status deployment/oauth-openshift -n openshift-authentication
```

### 1.3  Syncing LDAP Groups to OpenShift Groups

OpenShift에서 그룹을 사용하여 사용자를 관리하고 한 번에 여러 사용자의 권한을 제어할 수 있습니다. 문서에 [sync groups with LDAP](https://docs.openshift.com/container-platform/4.11/authentication/ldap-syncing.html) 방법에 대한 섹션이 있습니다. 그룹 동기화에는 `cluster-admin`권한이 있는 사용자로 OpenShift에 로그인할 때 `groupsync`라는 프로그램을 실행하고 다양한 그룹에서 찾은 사용자로 수행할 작업을 OpenShift에 지시하는 구성 파일을 사용하는 것이 포함됩니다.

다음과 같은 `groupsync` 구성 파일을 제공했습니다.

구성 파일 보기

```yaml
cat /opt/app-root/src/support/groupsync.yaml
```

`groupsync` 구성 파일은 다음을 수행합니다.

- 지정된 bind user 및 password를 사용하여 LDAP을 검색
- 이름이 `ose-` 로 시작하는 모든 LDAP 그룹에 대한 쿼리
- LDAP 그룹의 `cn`에서 이름을 사용하여 OpenShift 그룹을 생성
- LDAP 그룹의 구성원을 찾은 다음 생성된 OpenShift 그룹에 넣음
- OpenShift에서 `dn` 및 `uid`를 각각 UID 및 이름 속성으로 사용

`groupsync` 실행합니다:

```bash
oc adm groups sync --sync-config=/opt/app-root/src/support/groupsync.yaml --confirm
```

다음과 같은 내용이 표시됩니다.

```bash
group/ose-fancy-dev
group/ose-user
group/ose-normal-dev
group/ose-teamed-app
```

보고 있는 것은 `groupsync` 명령으로 생성된 **Group** object입니다. `--confirm` 플래그에 대해 궁금한 경우 `oc adm groups sync -h` 로 도움말 출력을 확인하십시오.



생성된 **Groups**를 보려면 다음을 실행합니다.

```bash
oc get groups
```

다음과 같은 내용이 표시됩니다.

```bash
NAME             USERS
ose-fancy-dev    fancyuser1, fancyuser2
ose-normal-dev   normaluser1, teamuser1, teamuser2
ose-teamed-app   teamuser1, teamuser2
ose-user         fancyuser1, fancyuser2, normaluser1, teamuser1, teamuser2
```

YAML의 특정 그룹을 살펴보십시오.

```bash
oc get group ose-fancy-dev -o yaml
```

YAML은 다음과 같습니다.

```yaml
apiVersion: user.openshift.io/v1
kind: Group
metadata:
  annotations:
    openshift.io/ldap.sync-time: "2022-02-10T01:49:07Z"
    openshift.io/ldap.uid: cn=ose-fancy-dev,ou=Users,o=5e615ba46b812e7da02e93b5,dc=jumpcloud,dc=com
    openshift.io/ldap.url: ldap.jumpcloud.com:636
  creationTimestamp: "2022-02-10T01:49:07Z"
  labels:
    openshift.io/ldap.host: ldap.jumpcloud.com
  name: ose-fancy-dev
  resourceVersion: "68628"
  uid: 374c463a-bdd2-4da1-ae1a-619eca0994f6
users:
- fancyuser1
- fancyuser2
```

OpenShift는 일부 LADP 메타데이터를 **Group**과 자동으로 연결하고 그룹에 있는 사용자를 나열했습니다.

**Users**를 나열하면 어떻게 됩니까?

```bash
oc get user
```

다음과 같이 표시될 것 입니다.

```bash
No resources found.
```

**Users**를 찾을 수 없는 이유는 무엇입니까? **Group** 정의에 명확하게 나열되어 있습니다.



**Users**는 처음 로그인을 시도할 때까지 실제로 생성되지 않습니다. **Group** 정의에서 보고 있는 것은 해당 특정 ID를 가진 **Users**를 만나면 **Group**과 연결되어야 한다고 OpenShift에 알리는 표시자입니다.



### 1.4 Change Group Policy

현재 실습 환경에는 특별한 `cluster-reader` 권한이 있어야 하는 *ose-fancy-dev*라는 특별한 슈퍼 개발자 그룹이 있습니다. 사용자가 클러스터에 대한 관리 수준 정보를 볼 수 있도록 하는 역할입니다. 예를 들어, 클러스터의 모든 **Projects**의 목록을 볼 수 있습니다.



`ose-fancy-dev` **Group**에 대한 정책을 변경합니다.

```bash
oc adm policy add-cluster-role-to-group cluster-reader ose-fancy-dev
```

> OpenShift와 함께 제공되는 다양한 역할에 관심이 있는 경우 [role-based access control(RBAC)](https://docs.openshift.com/container-platform/4.11/authentication/using-rbac.html) 문서에서 이에 대해 자세히 알아볼 수 있습니다.

### 1.5 Examine `cluster-reader` policy

계속해서 일반 사용자로 로그인하십시오. (오류가 발생하면 잠시 기다렸다가 다시 시도하십시오.)

```bash
oc login -u normaluser1 -p Op#nSh1ft
```

그리고 나서 **Projects** 목록을 확인합니다.

```bash
oc get projects
```

다음과 같이 보일 것 입니다.

```bash
No resources found.
```

`ose-fancy-dev` 멤버로 로그인합니다.

```bash
oc login -u fancyuser1 -p Op#nSh1ft
```

`oc get projects` 명령을 수행합니다.

```bash
oc get projects
```

클러스터의 모든 프로젝트 목록이 표시됩니다.

```bash
NAME                                               DISPLAY NAME   STATUS
app-management                                                    Active
default                                                           Active
default-broker                                                    Active
hive                                                              Active
kube-node-lease                                                   Active
kube-public                                                       Active
kube-system                                                       Active
lab-ocp-cns                                                       Active
local-cluster                                                     Active
multicluster-engine                                               Active
open-cluster-management                                           Active
open-cluster-management-agent                                     Active
open-cluster-management-agent-addon                               Active
open-cluster-management-hub                                       Active
--- 생략 ---
```

이제 OpenShift Container Platform의 RBAC이 작동하는 방식을 이해하기 시작해야 합니다.

### 1.6 Create Projects for Collaboration

클러스터 관리자로 로그인했는지 확인합니다.

```bash
oc login -u kubeadmin -p 6iZvh-3Jhd6-ZKTiW-MISP8
```

그런 다음 사람들이 협업할 여러 **Project**를 만듭니다.

```bash
oc adm new-project app-dev --display-name="Application Development"
oc adm new-project app-test --display-name="Application Testing"
oc adm new-project app-prod --display-name="Application Production"
```

이제 일반적인 소프트웨어 개발 수명 주기 설정을 나타내는 여러 **ProjectS**를 생성했습니다.  다음으로 이러한 프로젝트에 대한 협업 액세스 권한을 부여하도록 **Groups**을 구성합니다.

> `oc adm new-project`로 프로젝트를 생성하면 프로젝트 요청 프로세스 또는 프로젝트 요청 템플릿을 사용하지 않습니다. 이러한 프로젝트에는 기본적으로 적용되는 할당량 또는 제한 범위가 없습니다. 클러스터 관리자는 다른 사용자를 "가장(속임)" 할 수 있으므로 이러한 프로젝트가 할당량/제한 범위를 가져오도록 하려면 몇 가지 옵션이 있습니다.
>
> 1. `--as`를 사용하여 `oc new-projec`로 일반 사용자로 가장하도록 지정
> 2. `oc process`를 사용하고 프로젝트 요청 템플릿에 대한 값을 제공하여 만들기로 파이핑합니다. (eg: `oc process ... | oc create -f -`). 이렇게하면 프로젝트 요청 템플릿에 할당량 및 제한 범위를 포함하는 모든 objects가 생성됩니다.
> 3. 프로젝트 생성 후 할당량 및 제한 범위를 수동으로 생성/정의합니다.
>
> 이러한 연습의 경우 이러한 프로젝트에 대한 할당량이나 제한 범위가 있는 것은 중요하지 않습니다.

### 1.7 Map Groups to Projects

앞에서 본 것처럼 OpenShift에는 미리 구성된 여러 역할이 있습니다. **Projects**의 경우 마찬가지로 보기, 편집 또는 관리 액세스 권한을 부여할 수 있습니다. `ose-teamed-app` 사용자에게 개발 및 테스트 프로젝트를 편집할 수 있는 액세스 권한을 부여해보겠습니다.

```bash
oc adm policy add-role-to-group edit ose-teamed-app -n app-dev
oc adm policy add-role-to-group edit ose-teamed-app -n app-test
```

그런 다음 프로덕션을 볼 수 있는 액세스 권한을 부여합니다.

```bash
oc adm policy add-role-to-group view ose-teamed-app -n app-prod
```

`ose-fancy-dev` 그룹에 프로덕션 프로젝트에 대한 수정 액세스 권한을 부여합니다.

```bash
oc adm policy add-role-to-group edit ose-fancy-dev -n app-prod
```

### 1.8 Examine Group Access

`normaluser1`로 로그인하고 볼 수 있는 **Projects**를 확인합니다.

```bash
oc login -u normaluser1 -p Op#nSh1ft
oc get projects
```

다음을 확인 할 수 있습니다.

```bash
No resources found.
```

`ose-teamed-app` 그룹에서 `teamuser1`로 다시 시도합니다.

```bash
oc login -u teamuser1 -p Op#nSh1ft
oc get projects
```

다음을 확인 할  수 있습니다.

```bash
NAME       DISPLAY NAME              STATUS
app-dev    Application Development   Active
app-prod   Application Production    Active
app-test   Application Testing       Active
```

팀 사용자에게 프로덕션 프로젝트에 대한 편집 엑세스 권한을 부여하지 않았습니다. 계속해서 프로덕션 프로젝트에서 `teamuser1`로 무언가를 생성해 보십시오.

```bash
oc project app-prod
oc new-app docker.io/siamaksade/mapit
```

제대로 동작하지 않는 것을 볼 수 있습니다.

```bash
error: can't lookup images: imagestreamimports.image.openshift.io is forbidden: Use
r "teamuser1" cannot create resource "imagestreamimports" in API group "image.opens
hift.io" in the namespace "app-prod"
error:  local file access failed with: stat docker.io/siamaksade/mapit: no such fil
e or directory
error: unable to locate any images in image streams, templates loaded in accessible
 projects, template files, local docker images with name "docker.io/siamaksade/mapi
t"

Argument 'docker.io/siamaksade/mapit' was classified as an image, image~source, or
loaded template reference.

The 'oc new-app' command will match arguments to the following types:

  1. Images tagged into image streams in the current project or the 'openshift' pro
ject
     - if you don't specify a tag, we'll add ':latest'
  2. Images in the container storage, on remote registries, or on the local contain
er engine
  3. Templates in the current project or the 'openshift' project
  4. Git repository URLs or local paths that point to Git repositories

--allow-missing-images can be used to point to an image that does not exist yet.

See 'oc new-app -h' for examples.
```

### 1.9 Prometheus

이제 `cluster-reader` 권한 (클러스터의 많은 관리 측면을 볼 수 있는 권한)이 있는 사용자가 있으므로 Prometheus에 로그인을 시도할 수 있습니다. 

`cluster-reader` 권한이 있는 사용자로 로그인합니다:

```bash
oc login -u fancyuser1 -p Op#nSh1ft
```

다음 명령을 사용하여 `prometheus` `Route`를 찾습니다.

```bash
oc get route prometheus-k8s -n openshift-monitoring -o jsonpath='{.spec.host}{"\n"}'
```

다음과 같은 내용이 표시됩니다.

```bash
prometheus-k8s-openshift-monitoring.apps.cluster-qf4cl.qf4cl.sandbox2725.opentlc.co
m
```

> 계속하기 전에 OpenShift 웹 콘솔 (내장 콘솔 사용 안 함)로 이동하고 오른쪽 상단에 있는 `kube:admin` 드롭다운 메뉴를 사용하여 로그아웃 해야 합니다. 그렇지 않으면 Prometheus는 `kubeadmin` 사용자를 사용하여 인증을 통과하려고 합니다. 작동하는 동안 `cluster-reader` 역할을 보여주지는 않습니다.

설치 프로그램은 기본적으로 Prometheus에 대한 `Route`를 구성했습니다. 계속해서 Prometheus의 `Route`정보를 통해 새로운 브라우저에서 접속합니다. **Log in with OpenShift** 버튼을 클릭한 다음 `ldap` 인증 매커니즘을 선택하고 이전에 `cluster-reader` 권한을 부여한 `fancyuser1` 사용자를 사용합니다.

![01_ldap](https://github.com/justone0127/External-LDAP-Authentication-Providers-Users-and-Groups/blob/main/images/01_ldap.png)

보다 구체적으로 `ose-fancy-dev` 그룹에는 `cluster-reader` 권한이 있으며, `fancyuser1`은 구성원입니다.

이러한 모든 사용자의 password는 다음과 같습니다.

```bash
Op#nSh1ft
```

> 자체 서명된 인증서로 인해 오류가 발생할 수 있습니다. (클러스터 설치 방법에 따라 다름)

로그인 후 처음으로 인증 프록시 권한 승인이 표시됩니다.

![05_authorize_access](https://github.com/justone0127/External-LDAP-Authentication-Providers-Users-and-Groups/blob/main/images/02_authorize_access.png)

실제로 사용자와 Prometheus container 사이의 흐름에 있는 OAuth 프록시가 있습니다. 이 프록시는 AuthenticationN(AuthN)의 유효성을 검사하고 허용된 작업을 승인(AuthZ)하는 데 사용됩니다. 여기에서 Prometheus 액세스의 일부로 사용할 `fancyuser1` 계정의 권한을 *명시적으로 승인*합니다. 



이 시점에서 Prometheus를 보고 있습니다. 구성된 알림이 없습니다. `Status`와 `Targets`를 보면 현재 상태에 대한 몇 가지 흥미로운 정보를 볼 수 있습니다.



작업을 마친 후 관리 사용자로 다시 로그인해야 합니다.

```bash
oc login -u kubeadmin -p ${PASSWORD}
```


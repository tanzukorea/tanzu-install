
## Tanzu Application Platform (TAP) 인터넷 차단 환경 설치
본 가이드는 인터넷 차단 환경에서 TAP를 설치하는 방법에 대해 설명합니다.

### 0. 설치 환경 구성
설치 환경은 다음과 같습니다. 단, TKG가 이미 설치되어 있고 사설 레지스트리가 제공되는 상태에서 TAP 만을 AirGapped로 테스트 하였기 때문에, 네트워크를 차단하는 방식으로 환경을 구성합니다.
- AWS
- TKG 1.5.4
- TAP 1.1.1

#### 점프박스 VM 준비
TAP를 설치하기 위해 점프박스 VM을 준비하고 다음 소프트웨어를 설치합니다. 본 가이드에 사용된 Jumpbox는 Ubuntu 22.04 버전을 사용하였습니다.
- Tanzu CLI 및 플러그인 설치
  ```
  tanzu plugin install --local cli all
  ```

- imgpkg CLI: Tanzu CLI 설치 번들에 포함되어 있음
- kubectl CLI
- Docker CE

### 1. 설치 환경 구성
설치를 위한 환경 변수를 다음과 같이 설정합니다.
```
export INSTALL_REGISTRY_USERNAME=사설 레지스트리 접속 ID
export INSTALL_REGISTRY_PASSWORD=사설 레지스트리 접속 비밀번호
export INSTALL_REGISTRY_HOSTNAME=사설 레지스트리 FQDN
export TAP_VERSION=1.1.1
```

### 2. 레지스트리 로그인
점프박스 VM에서 다음의 레지스트리에 로그인을 합니다. 

TAP 설치 이미지가 저장되어 있는 VMware 공개 레지스트리
```
docker login registry.tanzu.vmware.com
```

TAP 설치 이미지를 저장할 사설 레지스트리
```
docker login $INSTALL_REGISTRY_HOSTNAME
```

### 3. 설치 이미지 리로케이션
사설 레지스트리로 이미지를 푸쉬하기 위해 공개 레지스트리에 저장되어 있는 이미지를 파일로 저장합니다.
```
imgpkg copy -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:${TAP_VERSION} --to-tar tap-packages-${TAP_VERSION}.tar --include-non-distributable-layers
```

> 만일 점프박스 VM에 Harbor 등의 사설 레지스트리를 설치하여 구성하고 이 VM 이미지를 인터넷 차단 환경으로 반입할 예정이라면, 다음 명령어를 이용하여 사설 레지스트리에 직접 이미지를 저장할 수도 있습니다.
```
imgpkg copy -b registry.tanzu.vmware.com/tanzu-application-platform/tap-packages:${TAP_VERSION} --to-repo ${INSTALL_REGISTRY_HOSTNAME}/tanzu-application-platform/tap-packages
```

### 4. 네트워크 차단
Jumpbox가 구성되었으면 이제 인터넷 차단 환경을 구성합니다.

#### 1. TKG 클러스터 아웃바운드
TKG가 생성한 "클러스터명-node" 보안그룹 명의 설정을 아래와 같이 추가/수정합니다.
|Protocol|Port|Target|비고|
|------|---|---|---|
|TCP|443|사설 레지스트리 IP|추가|
|All|All|클러스터에서 사용하는 Subnet CIDR|0.0.0.0/0에서 변경|

#### 2. 점프박스 VM 아웃바운드
|Protocol|Port|Target|비고|
|------|---|---|---|
|TCP|443|사설 레지스트리 IP|추가|
|TCP|6443|0.0.0.0/0|ELB 접속을 위해 오픈함. IP 목록을 정확히 알기 어려워서 전체 오픈|
|All|All|클러스터에서 사용하는 Subnet CIDR|추가|

#### 3. 점프박스 VM 인바운드
|Protocol|Port|Target|
|------|---|---|
|TCP|22|점프박스 VM을 접속할 PC의 IP|

### 5. 사설 레지스트리에 이미지 복사
다음 명령어를 사용하여 점프박스 VM에 파일로 저장된 이미지를 사설 레지스트리로 복사합니다. 
```
imgpkg copy --tar tap-packages-${TAP_VERSION}.tar --to-repo ${INSTALL_REGISTRY_HOSTNAME}/tanzu-application-platform/tap-packages --include-non-distributable-layers --registry-verify-certs=false
```
> 사설 인증서로 인증된 사설 레지스트리를 사용하는 경우 registry-verify-certs 옵션을 사용하여 레지스트리 인증을 생략할 수도 있습니다.

### 6. TAP 설치

#### 1) 네임스페이스 생성
TAP 설치를 위해 클러스터에 접속 후 다음 명령어를 사용하여 네임스페이스를 생성합니다.
```
kubectl create ns tap-install
```

#### 2) 레지스트리 인증 정보 추가
레지스트리 시크릿 추가
```
tanzu secret registry add tap-registry --username ${INSTALL_REGISTRY_USERNAME} --password ${INSTALL_REGISTRY_PASSWORD} --server ${INSTALL_REGISTRY_HOSTNAME} --export-to-all-namespaces --yes --namespace tap-install 
```

패키지 레지스트리 추가
```
tanzu package repository add tanzu-tap-repository --url ${INSTALL_REGISTRY_HOSTNAME}/tanzu-application-platform/tap-packages:${TAP_VERSION} --namespace tap-install
```





확인
```
tanzu package repository get tanzu-tap-repository --namespace tap-install
```

목록 확인
```
tanzu package available list --namespace tap-install 
```

버전 확인
```
tanzu package available get tap.tanzu.vmware.com/${TAP_VERSION} --values-schema --namespace tap-install 
```


#### 3) TAP 설치를 위한 YAML 파일 생성
TAP 을 구성할 프로파일은 Full을 기준으로 작성하였으며, 아래 템플릿을 참고하여 YAML 파일을 생성합니다.
* YAML 파일 템플릿 참조: https://raw.githubusercontent.com/tanzukorea/tanzu-install/main/airgapped/tap/tap-values.yaml
* 템플릿에 사용된 관련 변수 정보
  |변수명|설명|
  |------|---|
  |INSTALL_REGISTRY_HOSTNAME|사설 레지스트리 접속 FQDN|
  |INSTALL_REGISTRY_USERNAME|사설 레지스트리 접속 ID|
  |INSTALL_REGISTRY_PASSWORD|사설 레지스트리 접속 비밀번호|
  |TANZUNET-USERNAME|registry.tanzu.vmware.com 접속 ID|
  |TANZUNET-PASSWORD|registry.tanzu.vmware.com 접속 비밀번호|
  |INGRESS-DOMAIN|TAP 구성에 사용할 인그레스 도메인|
  |GIT-CATALOG-URL|TAP 카탈로그를 구성할 YAML 파일이 저장되어 있는 GIT의 주소|

#### 4) TAP 설치
다음 명령어를 사용하여 TAP 패키지를 설치합니다. tap-values.yaml 파일은 위의 3)번 단계에서 생성한 파일입니다.
```
tanzu package install tap -p tap.tanzu.vmware.com -v $TAP_VERSION --values-file tap-values.yaml -n tap-install
```

만일 TAP 패키지의 수정이 필요하면 다음 명령어를 사용하여 패키지를 수정합니다.
```
tanzu package installed update tap -p tap.tanzu.vmware.com -v $TAP_VERSION --values-file tap-values.yaml -n tap-install
```

#### 5) TAP 설치 확인
다음 명령어를 사용하여 TAP 패키지 설치를 확인합니다.
```
tanzu package installed list -n tap-install
```

---
title: "[OKD] 외부 nexus에서의 image pull"
excerpt: "외부 nexus에서의 image pull 하면서 겪을 시행착오"

categories:
  - DevOps
tags:
  - [okd, nexus, devops]

permalink: /devops/okd-nexus-image-pull/

toc: true
toc_sticky: true

date: 2024-04-10
last_modified_at: 2024-04-10
---

# 🦥 okd nexus 이미지 풀 다운로드

## 1. docker secret 등록

```java
oc create secret docker-registry nexus \
--docker-server={server.url} \
--docker-username={username} \
--docker-password={password} --docker-email={email}
```

## 2. deployment 등록

oc apply -f `.yaml`

```java
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echoserver-deployment
  namespace: test
  labels:
    app: echoserver
spec:
  replicas: 3
  selector:
    matchLabels:
      app: echoserver
  template:
    metadata:
      labels:
        app: echoserver
    spec:
      imagePullSecrets:
      - name: nexus
      containers:
      - name: gateway
        image:{nexus.image.url}:{version}
        ports:
        - containerPort: 58080
        env:
        - name: eureka.client.fetch-registry
          value: "false"
        - name: eureka.client.register-with-eureka
          value: "false"
```

okd에 deployment를 띄우면서, nexus server에서 이미지를 pull하도록 등록했다.

등록해 준 Secret name을 yaml 파일에 작성해서 apply  한다.

`imagePullSecrets:`  
`- name: nexus`

## 결과

이미지를 다운받느라 시간이 조금 소요된다. 배포 된 다음 잘 동작한다.

# 그 외의 시도와 실패 내역

- 실패 사례이므로 따라하지 말 것

## 1. `oc edit image.config.openshift.io/cluster`

. nexus.synology의 server주소를 `192.168.x.xx:5001` 로 지정해서 시도했더니 아래와 같은 오류가 발생했다.

`Failed to pull image "192.168.x.xx:5001/test/test:latest": rpc error: code = Unknown desc = pinging container registry 192.168.x.xx:5001: Get "https://192.168.x.xx:5001/v2/": http: server gave HTTP response to HTTPS client`

```java
Events:
  Type     Reason          Age                   From               Message

  Normal   Scheduled       146m                  default-scheduler  Successfully assigned test/gateway-deployment-5b7fb45f68-z4sbk to okd-r4fbd-worker-0-7zx2m
  Normal   AddedInterface  146m                  multus             Add eth0 [10.129.2.24/23] from ovn-kubernetes
  Warning  Failed          144m (x4 over 146m)   kubelet            Failed to pull image "192.168.x.xx:5001/test/test:latest": rpc error: code = Unknown desc = pinging container registry 192.168.x.xx:5001: Get "https://192.168.x.xx:5001/v2/": http: server gave HTTP response to HTTPS client
  Warning  Failed          144m (x4 over 146m)   kubelet            Error: ErrImagePull
  Warning  Failed          144m (x6 over 146m)   kubelet            Error: ImagePullBackOff
  Normal   Pulling         10m (x31 over 146m)   kubelet            Pulling image "192.168.x.xx:5001/test/test:latest"
  Normal   BackOff         62s (x630 over 146m)  kubelet            Back-off pulling image "192.168.x.xx:5001/test/test:latest"
```

이 오류는 jenkins pipeline 작성 했을 때에도 본 적이 있는 오류였다.

당시에 이 오류를 docker 설정에서 `insecure-registries` 로 해당 아이피를 설정했어야 했기 때문에,

```java
$ vi /etc/docker/daemon.json

{

    "insecure-registries": ["{ip}:{port}"]

}
```

okd machine-config 에 `oc edit image.config.openshift.io/cluster`

로`image.config.openshift.io/cluster` CR: 을 수정해서 insecureRegistries를 작성해줬다.

```
spec:
  registrySources:
    insecureRegistries:
    - "192.168.x.xx:5001"
    allowedRegistries:
    - quay.io
    - registry.redhat.io
    - image-registry.openshift-image-registry.svc:5000
```

CR을 수정하면, machine config pool 이 동작하는 모습을 볼 수 있고

```java
$ oc get mcp
        
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-7c62a88d655ff8f5e4134958d4ca6d63   True      False      False      3              3                   3                     0                      2d22h
worker   rendered-worker-537c6d7d873dad6a453ee412d5a1d647   True      False      False      3              3                   3                     0                      2d22h
```

각 master 와 worker 에 수정한 machine config 값이 반영되어야 하는게 맞다.

/etc/config/containers/policy.json 의 기본값

```java
{
"default": [
{
"type": "insecureAcceptAnything"
}
],
"transports":
{
"docker-daemon":
{
"": [{"type":"insecureAcceptAnything"}]
}
}
}
```

이미지 정책상 default 타입이 “insecureAcceptAnything” 이다. 그런데 해당 내용이 업데이트 되고 나면, 지정한 이미지 설정을 제외한 부분을 “reject”로 변경한다.

```java
{
  "default": [
    {
      "type": "reject"
    }
  ],
  "transports": {
    "atomic": {
      "192.168.x.xx:5001": [
        {
          "type": "insecureAcceptAnything"
        }
      ],
      "image-registry.openshift-image-registry.svc:5000": [
        {
          "type": "insecureAcceptAnything"
        }
      ]
    },
    "docker": {
      "192.168.x.xx:5001": [
        {
          "type": "insecureAcceptAnything"
        }
      ],
      "image-registry.openshift-image-registry.svc:5000": [
        {
          "type": "insecureAcceptAnything"
        }
      ]
    },
    "docker-daemon": {
      "": [
        {
          "type": "insecureAcceptAnything"
        }
      ]
    }
  }
}
```

문제는 machine config 값 반영이 master에만 되거나 worker에만 반영되어 오류를 발생하고, 해당 건을 파일 경로에서 문서를 수정하는 방식으로 작성하면 mismatch 에러가 발생한다.

```java
Message:               Node okd-26qs9-worker-0-cx5zs is reporting: "failed to drain node: okd-26qs9-worker-0-cx5zs after 1 hour. Please see machine-config-controller logs for more information", Node okd-26qs9-worker-0-j4sr6 is reporting: "unexpected on-disk state validating against rendered-worker-75b63269977b730732df5db05c4da3b7: content mismatch for file \"/etc/containers/policy.json\""
```

### policy.json의 설정 값 설명

https://github.com/containers/image/blob/main/docs/containers-policy.json.5.md#insecureacceptanything

## 2. machineconfig 추가

```java
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 99-worker-unqualified-search-registries
spec:
  config:
    ignition:
      version: 3.1.0
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,dW5xdWFsaWZpZWQtc2VhcmNoLXJlZ2lzdHJpZXMgPSBbJ3JlZ2lzdHJ5LmFjY2Vzcy5yZWRoYXQuY29tJywgJ2RvY2tlci5pbycsICcxOTIuMTY4LjIuMTE6NTAwMSdd
        filesystem: root
        mode: 0644
        path: /etc/containers/registries.conf.d/99-worker-unqualified-search-registries.conf

```

```java
unqualified-search-registries = ['registry.access.redhat.com', 'docker.io', '192.168.2.11:5001']
```

## 3. /etc/containers/registries.conf.d 에 myregistry.conf 파일을 추가해줬다.

```java
[core@okd-r4fbd-worker-0-6t4rm ~]$ cd etc/containers/registries.conf.d
-bash: cd: etc/containers/registries.conf.d: No such file or directory
[core@okd-r4fbd-worker-0-6t4rm ~]$ cd /etc/containers/registries.conf.d
[core@okd-r4fbd-worker-0-6t4rm registries.conf.d]$ ll
total 12
-rw-r--r--. 1 root root 5828 Jan  9 01:40 000-shortnames.conf
-rw-r--r--. 1 root root   60 Jan 10 06:04 myregistry.conf
[core@okd-r4fbd-worker-0-6t4rm registries.conf.d]$ cat myregistry.conf 
[[registry]]
location = "192.168.2.11:5001"
insecure = true

```

영향을 주지는 않았다.

- 설정 값 변경을 할 때마다 `oc get co`로 확인해보면, 반영되는 동안 오류가 발생한다. 그 과정에서 콘솔 대시보드가 동작하지 않거나, api server가 접속이 안될 수 있다.

## 해결

ip 로 https  요청이 불가 하기 때문에, 도메인으로 지정하고, 문제가 해결되었다. mc, mcp 등의 설정을 변경하지 않아도 된다.

## 참고 자료

https://github.com/containers/image/blob/main/docs/containers-policy.json.5.md#insecureacceptanything

https://docs.openshift.com/container-platform/4.9/openshift_images/image-configuration.html#images-configuration-insecure_image-configuration

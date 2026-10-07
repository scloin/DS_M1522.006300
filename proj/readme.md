## 과제용 A100 노드 사용 가이드

### 1. SSH Tunnel
A100 노드에 직접 SSH 접속하지 않고, SSH tunnel을 통해 Kubernetes API 서버 접근

```bash
ssh -N \
  -L 16443:127.0.0.1:6443 \
  -p 53036 \
  k8s-tunnel@147.47.206.149
```

* ***PW = ETL 별도 공지*** 
* 별도 출력이 없는 것이 정상
* 해당 터미널을 유지한 상태에서 다른 터미널로 이동 후 작업 진행

### 2. kubeconfig

> `kubectl` 설치 필요: [kubectl 설치 가이드](https://kubernetes.io/docs/tasks/tools) \
> 조장을 통해 팀별 kubeconfig 파일 개별 제공 예정

```bash
export KUBECONFIG=$HOME/path/to/teamXX.kubeconfig
kubectl auth can-i create jobs
```

```text
yes → 사용 가능
no  → 사용 불가
```

### 3. 사용 방식
#### 3-1. Interactive 사용

`Job`을 생성한 뒤, 생성된 Pod 내부 shell에 접속하여 코드 작성 및 테스트 \
예시 파일 : <a href="./dev.yaml" download>dev.yaml</a>

```bash
kubectl apply -f dev.yaml
kubectl get pods
kubectl exec -it <pod-name> -- bash
```

#### 3-2. 실험 실행
필요한 GPU 수만큼 Pod를 생성하여 분산 학습 실행
예: 8 GPU 사용 시 8개 Pod 생성 \
예시 파일 : <a href="./experiment.yaml" download>experiment.yaml</a>
```bash
kubectl apply -f experiment.yaml
kubectl get pods -o wide
```

Pod별 독립 IP 기반 NCCL 통신 및 pairwise `tc` 설정 가능 \
8-rank GPU/NCCL/TC 학습 동작 확인 완료

#### 주요 경로

```text
/workspace  팀별 RW (코드, 이미지, 학습모델, 결과 등)
/shared     공용 RO (모델, 데이터셋 등)
```

### 4. Network Emulation
- Heterogeneous network 환경 재현 용도
- Linux Traffic Control (`tc`) 사용
- 실험용 Pod에는 `NET_ADMIN` capability 필요
- 참고 : [tc-netem](https://man7.org/linux/man-pages/man8/tc-netem.8.html), [tc-htb](https://man7.org/linux/man-pages/man8/tc-htb.8.html), [tc-u32](https://man7.org/linux/man-pages/man8/tc-u32.8.html)
```yaml
# experiment.yaml 참고
securityContext:
  capabilities:
    add:
    - NET_ADMIN
```

예: 전체 outbound traffic에 50 ms latency 추가

```bash
tc qdisc add dev eth0 root netem delay 50ms
```

설정 확인 / 제거

```bash
tc qdisc show dev eth0
tc qdisc del dev eth0 root
```

### 5. 확인 / 종료

```bash
kubectl get jobs
kubectl get pods
kubectl logs <pod-name>

kubectl delete job <job-name>
```

### 6. 사용 시간
- 팀별 A100 사용 시간: [ETL에 공지](https://docs.google.com/spreadsheets/d/1DCHBkrCIh-42IE9DUCYJjBmGgAZQ7fey_TPYcSjufmY/edit?gid=1373773841#gid=1373773841)
- 할당 시간 종료 시:
  - Kubernetes 접근 권한 자동 회수
  - 실행 중인 Job / Pod 자동 삭제
  - /workspace의 작업 파일 유지
# deployment-collection

Kubernetes 플랫폼과 애플리케이션 워크로드를 관리하는 `dev_infra.deployment` Ansible
Collection입니다.

## 구조

- `k8s_deployment`: Kubernetes/Helm/RKE2 image import 공통 동작
- 플랫폼 Role: storage, cert-manager, ACME/TLS, PKI, APISIX, ExternalDNS, Reloader
- 워크로드 Role: Vault, etcd, OpenSearch, OpenSearch Dashboard
- `playbooks/`: `init`, `configure`, `kickstart`, `pull`, `restart`, `uninstall`

서비스 Role은 public Playbook에서 직접 호출되며 전달만 하는 wrapper Role은 사용하지 않습니다. 비밀
scalar와 인벤토리는 소비 저장소가 inline Ansible Vault 변수로 제공하고, control-node kubeconfig는
whole-file Vault 경로로 전달할 수 있습니다. 공식 Interface는 public Playbook이며 Role의 `tasks_from`은
그 구현에 사용합니다.

`restart`는 local-path, cert-manager, APISIX, ExternalDNS, Reloader, Vault와 애플리케이션 workload를
함께 재시작합니다. `uninstall`은 실행 자체를 승인으로 간주하며 Collection이 만든 Kubernetes platform,
workload, namespace, PVC/PV, CRD와 소유가 확인된 Cloudflare DNS record를 모두 제거합니다. RKE2와 host
설정은 소비 저장소 소유이므로 제거하지 않습니다. `--check`를 명시하면 외부 소유권과 삭제 대상을
검증하지만 Cloudflare DELETE를 수행하지 않습니다.

관리 namespace와 cluster-scoped resource에는 exclusive ownership label을 기록합니다. 기존 설치는 새
Collection의 `configure`를 한 번 실행해 ownership을 반영한 뒤 `uninstall`해야 합니다. 다른 namespace의
CR이나 local-path PVC처럼 foreign consumer가 발견되면 전체 제거를 시작하기 전에 중단합니다.

## Ansible control node

Kubernetes와 Helm API 작업은 `dev-infra`에서 `ansible-playbook`을 실행하는 Ansible control
node에서 수행합니다. Kubernetes helper는 `delegate_to: localhost`, `become: false`, `run_once:
true`를 사용하고 `ansible_playbook_python`으로 `kubernetes.core` module을 실행합니다. RKE2 managed
node에는 Collection 전용 Helm, kubectl, Python venv를 설치하지 않습니다. `pull`의 RKE2 image
import와 `crictl` 검증만 managed node에서 실행합니다.

`kubernetes.core` local action plugin이 Vault 암호문 kubeconfig를 실행 중에 임시 복호화하고 종료 시
정리합니다. 이 동작은 local transport에만 적용되므로 Kubernetes/Helm 작업은 control node 위임을
유지해야 합니다.

Control node에는 Ansible, Helm, Git, jq와 Python `kubernetes`, `yaml`, `jsonpatch` module이 미리
설치되어 있어야 합니다. Collection은 이를 설치하지 않고 `init`에서 검증합니다. kubectl은 필수
의존성이 아닙니다.

Vault fresh install은 자동 초기화하지 않습니다. `configure`가 uninitialized 상태에서 중단하면
보호된 운영 세션에서 `vault operator init`을 수행하고 생성 값을 inline Ansible Vault 변수로 갱신한
뒤 `configure`를 끝까지 다시 실행하고 `kickstart`를 실행합니다. root token과 unseal key를 평문 파일,
로그 또는 shell history에 남기지 않습니다.

## 검증과 0.0.1 릴리스

```bash
task check
task release:status
```

초기 개발 중 `0.0.1`을 갱신할 때는 변경을 commit하고 `main`에 push한 뒤 다음을 실행합니다.

```bash
task release:retag
```

`release:retag`은 clean worktree와 `HEAD == origin/main`을 확인하고 build, Collection 무결성 검증,
6개 public Playbook syntax check를 통과한 HEAD로만 태그를 이동합니다. 원격 태그는
`--force-with-lease`로 갱신해 검증 중 다른 작업자가 이동한 태그를 덮어쓰지 않습니다.

소비 저장소에서는 태그 이동 후 Collection을 재설치하고 설치 receipt SHA를 확인합니다.

```bash
task collection:reinstall
task collection:status
```

안정화 후에는 기존 태그를 이동하지 않고 새 버전을 발행합니다.

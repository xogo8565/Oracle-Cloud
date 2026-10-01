# OCI A1 Always Free 자동 생성

이 저장소의 GitHub Actions 워크플로는 10분마다 `ap-osaka-1`의 지정 컴파트먼트에서 `oci-a1-always-free` 인스턴스가 이미 존재하는지 확인합니다. 인스턴스가 없으면 Canonical Ubuntu 24.04 Minimal aarch64 이미지를 조회해 A1.Flex 인스턴스 생성을 시도합니다. 용량 부족 등 생성 실패는 해당 실행을 실패 처리하므로 다음 예약 실행에서 다시 시도합니다.

동일한 GitHub Actions 동시성 그룹을 사용하고, 조회 시 `TERMINATED` 이외의 모든 상태를 기존 인스턴스로 취급합니다. 실행 중인 인스턴스가 확인되면 새로 만들지 않습니다. 인스턴스 이름은 `oci-a1-always-free`이며, 설정은 1 OCPU, 6 GB 메모리, 50 GB 부트 볼륨입니다.

## GitHub Actions 설정

저장소의 **Settings → Secrets and variables → Actions**에서 다음 값을 등록하세요.

### Repository secrets

| 이름 | 값 |
| --- | --- |
| `OCI_USER_OCID` | OCI API 서명 키를 소유한 사용자 OCID |
| `OCI_FINGERPRINT` | OCI에 등록한 API 서명 공개 키의 fingerprint |
| `OCI_API_PRIVATE_KEY_PEM` | 해당 API 서명 키의 PEM 개인 키 전체 내용 |
| `OCI_API_KEY_PASSPHRASE` | 개인 키가 암호화된 경우에만 설정 |

### Repository variables

| 이름 | 값 |
| --- | --- |
| `OCI_TENANCY_OCID` | OCI tenancy OCID |
| `OCI_COMPARTMENT_OCID` | 인스턴스를 생성하고 검색할 컴파트먼트 OCID |
| `OCI_REGION` | `ap-osaka-1` |
| `OCI_AVAILABILITY_DOMAIN` | `ZgAv:AP-OSAKA-1-AD-1` |
| `OCI_SUBNET_OCID` | 사용할 VCN 서브넷 OCID |
| `OCI_SSH_PUBLIC_KEY` | 인스턴스의 `authorized_keys`에 넣을 공개 키 한 줄 |

사용자가 전달한 `ocid1.tenancy...` 값은 tenancy OCID입니다. tenancy 루트 컴파트먼트를 대상으로 하는 경우 이 값을 `OCI_TENANCY_OCID`와 `OCI_COMPARTMENT_OCID` 양쪽에 입력할 수 있습니다. 하위 컴파트먼트를 사용한다면 `OCI_COMPARTMENT_OCID`에는 해당 하위 컴파트먼트 OCID를 입력하세요.

개인 키를 GitHub Actions secret에 저장하고, 저장소 파일이나 로그에 기록하지 마세요. 제공한 인스턴스 로그인 공개 키는 `OCI_SSH_PUBLIC_KEY` 변수에 저장합니다. GitHub 저장소 인증 키와 OCI 인스턴스 로그인 키의 용도를 혼동하지 마세요.

## OCI 사용자 및 IAM 정책

OCI에서 전용 그룹과 API 서명 키를 사용하는 사용자를 준비하고, 사용자를 해당 그룹에 추가하세요. 사용자의 API 서명 공개 키를 OCI 사용자 프로필에 등록한 뒤 fingerprint를 `OCI_FINGERPRINT` secret에 저장합니다. 개인 키는 `OCI_API_PRIVATE_KEY_PEM` secret에 저장합니다.

정책은 tenancy 정책에 만들고, 인스턴스 대상 컴파트먼트와 서브넷이 위치한 컴파트먼트를 각각 지정합니다. `<group-name>`, `<compute-compartment>` 및 `<network-compartment>`를 실제 이름으로 바꾸세요.

```text
Allow group <group-name> to manage instance-family in compartment <compute-compartment>
Allow group <group-name> to read app-catalog-listing in tenancy
Allow group <group-name> to use virtual-network-family in compartment <network-compartment>
```

이 권한은 인스턴스 조회/생성, 이미지 조회, 서브넷 사용 및 VNIC 연결을 위한 것입니다. 워크플로는 VCN이나 서브넷을 생성하지 않습니다.

## 배포 전 확인

- `OCI_COMPARTMENT_OCID`가 실제 생성 대상 컴파트먼트인지 확인합니다. tenancy OCID는 루트 컴파트먼트에 사용할 수 있습니다.
- `OCI_SUBNET_OCID`가 Osaka 리전의 대상 가용성 도메인에서 인스턴스 생성이 가능한 서브넷인지 확인합니다.
- 제공한 A1 구성은 1 OCPU / 6 GB입니다. Always Free A1 한도는 tenancy 전체 합계 2 OCPU / 12 GB이므로, 다른 A1 인스턴스 사용량도 합산해야 합니다.
- OCI Always Free 컴퓨트는 tenancy의 홈 리전에서 생성해야 합니다. `ap-osaka-1`이 홈 리전인지 확인하세요.
- 공개 IP는 서브넷 설정을 따릅니다. SSH 접근이 필요하면 공용 서브넷/인터넷 경로 및 보안 목록 또는 NSG의 SSH 규칙을 별도로 확인하세요.
- GitHub Actions 예약 워크플로는 기본 브랜치에서만 실행됩니다. 브랜치 검토 후 기본 브랜치에 반영하고 필요한 secrets/variables와 IAM 정책을 설정한 다음 `workflow_dispatch`로 먼저 실행하세요.

## 공식 문서

- [OCI Compute instance launch CLI](https://docs.oracle.com/en-us/iaas/tools/oci-cli/latest/oci_cli_docs/cmdref/compute/instance/launch.html)
- [OCI Compute image list CLI](https://docs.oracle.com/en-us/iaas/tools/oci-cli/latest/oci_cli_docs/cmdref/compute/image/list.html)
- [OCI Always Free 리소스와 한도](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm)
- [OCI 인스턴스 생성 IAM 정책 예시](https://docs.oracle.com/en-us/iaas/Content/Identity/policiescommon/commonpolicies.htm)
- [GitHub Actions workflow 문법 및 concurrency](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)
- [GitHub Actions secrets](https://docs.github.com/en/actions/concepts/security/secrets)

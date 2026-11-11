# CourtSide

Workshop repository for learning GitLab CI/CD, Podman and Kubernetes.

Current contents are **infrastructure verification only**. The React frontend,
FastAPI API and simulator have not been implemented.

The temporary instructor pipeline verifies the test database, builds a disposable
image with rootless Podman, pushes it to the GitLab Container Registry, deploys it
to the `courtside-demo` namespace, checks the rollout and removes the workload.
The participant starter pipeline will be defined separately when the application
is built; this is not the participant starting point.

Required project configuration:

- Protected default branch `main`.
- Dedicated protected runner, tag `courtside`, untagged jobs disabled.
- `TEST_DATABASE_URL`: masked/hidden protected variable, environment scope `test`.
- `KUBE_CONFIG`: masked/hidden protected **File** variable, environment scope `demo`.
- Variable expansion disabled for both secrets.
- Public Container Registry read access for the disposable image.

The deployer can manage Deployments and ConfigMaps only in the demonstration
namespace and read rollout/log information. It cannot read Secrets through the
API or administer the cluster. It is nevertheless trusted with demonstration
data: anyone who can change a workload can make that workload consume its
namespace's Secrets. No development, test or administrator credentials belong in
that namespace.

No host socket or permanent credential mount is exposed to pipeline containers.
Kubernetes uses containerd; Podman runs PostgreSQL and the CI containers.
The cluster and database endpoints are private and accessed through SSH tunnels
from the instructor's Mac. Credentials and kubeconfigs must never be committed.

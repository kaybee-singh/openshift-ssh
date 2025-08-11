```bash
oc project ssh-test
```
```bash
oc apply -f - <<'YAML'
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: sshd-anyuid-syschroot
allowPrivilegedContainer: false
allowedCapabilities: ["SYS_CHROOT","NET_BIND_SERVICE"]
defaultAddCapabilities: []
requiredDropCapabilities: []
allowHostNetwork: false
allowHostPID: false
allowHostIPC: false
readOnlyRootFilesystem: false
fsGroup: { type: RunAsAny }
runAsUser: { type: RunAsAny }    # keeps anyuid behavior (root allowed)
seLinuxContext: { type: MustRunAs }
supplementalGroups: { type: RunAsAny }
users:
- system:serviceaccount:ssh-test:default
YAML
```
```bash
oc delete pod ssh-pod --ignore-not-found
```
```bash
oc apply -f - <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: ssh-pod
  namespace: ssh-test
spec:
  serviceAccountName: default
  securityContext:
    runAsUser: 0
    runAsNonRoot: false
  containers:
  - name: ubuntu
    image: ubuntu:22.04
    command: ["bash","-lc","sleep infinity"]
    securityContext:
      runAsUser: 0
      allowPrivilegeEscalation: true
      capabilities:
        add: ["SYS_CHROOT","NET_BIND_SERVICE"]
    ports:
    - containerPort: 22
YAML
```
# verify the SCC in use (must print: sshd-anyuid-syschroot)
```bash
oc get pod ssh-pod -o jsonpath='{.metadata.annotations.openshift\.io/scc}{"\n"}'
```
```bash
oc rsh ssh-pod
```
```bash
export DEBIAN_FRONTEND=noninteractive
apt-get update &&
apt-get install -y openssh-client openssh-server sudo procps &&
mkdir -p /run/sshd && ssh-keygen -A &&
mkdir -p /etc/ssh/sshd_config.d &&
printf "PasswordAuthentication yes\nPermitRootLogin no\nUsePAM no\nListenAddress 0.0.0.0\n" > /etc/ssh/sshd_config.d/60-custom.conf &&
id ocpadmin 2>/dev/null || (useradd -m -s /bin/bash ocpadmin && echo "ocpadmin:redhat" | chpasswd && adduser ocpadmin sudo) &&
/usr/sbin/sshd -t &&
exec /usr/sbin/sshd -D -e
```
Now try to SSH the pod from the OCP node
```bash
oc get pods -o wide
```
```bash
oc debug node/node-name
chroot /host
ssh ocpadmin@10.0.11.12
```

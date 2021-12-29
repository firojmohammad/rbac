# rbac
creating roles and rolebinding using rbac in kubuernetes
Setting up RBAC in GKE

This document explains how to setup role based user accounts in GKE Cluster assuming  that you already have access to the cluster.

Create a test namespace
$ kubectl create ns test
 Download cfssl  and cfssljson in order to generate new CSR.
$ sudo apt install golang-cfssl 
Generate CSR and private key
$ cat <<EOF | cfssl genkey - | cfssljson -bare server
{
  "hosts": [
    "office"
      ],
  "CN": "office",
  "key": {
    "algo": "ecdsa",
    "size": 256
  }
}
EOF


Create a CSR object to send  it to Kubernetes API
$ cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRe
metadata:
  name: office-csr
spec:
  request: $(cat server.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF

Verify if CSR is created and is in pending state.
$ kubectl get csr


Sign the CSR and Save the issued certificate in server.crt
$ kubectl certificate approve test-csr 
$ kubectl get csr test-csr -o jsonpath='{.status.certificate}'| base64 --decode > server.crt


Define and apply a  role
$ vim role.yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: test
  name: test
rules:
- apiGroups: ["", "extensions", "apps"]
  resources: ["deployments", "replicasets", "pods"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
$ kubectl apply -f role.yaml
 
Bind the above role to user “test” using rolebinding
$ vim rolebinding.yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: test
  namespace: test
subjects:
- kind: User
  name: test
  apiGroup: ""
roleRef:
  kind: Role
  name: test
  apiGroup: ""
Copy the kubeconfig entry below , edit necessary parameters and  save it to a file “config”
$ vim config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: YOUR_BASE64_CA_CERTIFICATE_HERE             
    server: https://ENDPOINT_IP
  name: CLUSTER_NAME
contexts:
- context:
    cluster: CLUSTER_NAME
    namespace: test
    user: test
  name: test-context
current-context: test-context
kind: Config
preferences: {}
users:
- name: test
  user:
    client-certificate: server.crt
    client-key: server-key.pem
Get the CA-CERTIFICATE , ENDPOINT_IP and CLUSTER_NAME from the GCP Web Portal. Reference is provided in below snapshot.

The role attached to the  above user has get, list, watch ,create, update and  patch permissions on resources deployments , pods and replicasets. 
 
$ kubectl $verb $resource
Where, verb = get, list, watch ,create, update and  patch
       Resource = deployments, replicasets and pods
The  above command should be able to execute on providing the verb and resource parameters provided  above in namespace test. It should give forbidden error when verbs and resources  not listed  above are provided.
Copy the kubeconfig entry below , edit necessary parameters and  save it to a file “config”.


 




Create Service Account, Role, ClusterRole & Assign that role, And create a secret for Service Account and genrate a Token
Creating Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: webapps
**Create Role**
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: webapps
rules:
  - apiGroups:
        - ""
        - apps
        - autoscaling
        - batch
        - extensions
        - policy
        - rbac.authorization.k8s.io
    resources:
      - pods
      - secrets
      - componentstatuses
      - configmaps
      - daemonsets
      - deployments
      - events
      - endpoints
      - horizontalpodautoscalers
      - ingress
      - jobs
      - limitranges
      - namespaces
      - nodes
      - pods
      - persistentvolumes
      - persistentvolumeclaims
      - resourcequotas
      - replicasets
      - replicationcontrollers
      - serviceaccounts
      - services
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
**Bind the role to service account**
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: webapps 
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-role 
subjects:
- namespace: webapps 
  kind: ServiceAccount
  name: jenkins 
**Create a ClusterRole for PV access**
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: persistent-volume-access
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
**Bind the clusterRole to Jenkins Service Account**
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-persistent-volume-access
subjects:
  - kind: ServiceAccount
    name: jenkins
    namespace: webapps
roleRef:
  kind: ClusterRole
  name: persistent-volume-access
  apiGroup: rbac.authorization.k8s.io
Generate token using service account in the namespace
Create Token

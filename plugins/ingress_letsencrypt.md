# kubernetes_ingress_letsencrypt

![](https://i.imgur.com/bbCz37T.png)
## ��?Domain Name
�����N�O��?�@?�A�n���I��, ??�I���W?���ܦh���i�H�d�@�U��?�I��өάO�@�Ǭ�?����?,??�ڴN�����h�h����?�F,�峹��?�Hsam.nctu.me?�@�S��



## ��[Letsencrypt](https://letsencrypt.org/)?????
??��?�Τ�?���⥦��?�U?,�W[Letsencrypt](https://letsencrypt.org/)�h�w?certbot, ��??���覡�]�i�H?�ҧڪ��W�@�g[?letsencrpyt??](https://www.nctusam.com/2017/08/30/ubuntumint-nginx-%E7%B0%BDletsencrpyt%E6%86%91%E8%AD%89/) �峹,?�J�H�U���O????


```shell
$ sudo certbot certonly --standalone sam.nctu.me
```
?�G?�p�U��?��
```
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/sam.nctu.me/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/sam.nctu.me/privkey.pem
   Your cert will expire on 2017-12-15. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le

```


?�n�Z?��?��b

```
/etc/letsencrypt/live/sam.nctu.me/
```
�i�H�ݨ�̭���??,?����?�Oroot�~��ݨ�???��?��?�e
```shell
# ls
cert.pem  chain.pem  fullchain.pem  privkey.pem  README
```

????�N?���F

## ?�mKubernetes Secret

?fullchain.pem(�]?�ϥ�nginx controller) �O privkey.pem?�إ�secret,?���Z?�t�m?ingress�ϥ�

```shell
$ kubectl create secret tls tls-certs --cert fullchain.pem --key privkey.pem 
```
�d��secret
```shell
$ kubectl get secret
NAME                              TYPE                                  DATA      AGE
default-token-rtlbl               kubernetes.io/service-account-token   3         6d
tls-certs                         kubernetes.io/tls                     2         55m
```
�H�إߧ���


## �إ�Ingress
��?���h�إߤ@?ingress��yaml?,??�U��?create ingress�}�[�Jtls��?�w
ingress.yaml
```yaml=
apiVersion: extensions/v1beta1                                                                                
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  tls:
    - hosts:
      - sam.nctu.me
      secretName: tls-certs
  rules:
  - host: sam.nctu.me
    http:
      paths:
        - backend:
            serviceName: phpmyadmin
            servicePort: 80
```
�M�Z�إߦ�Ingress

```shell
$ kubectl create -f ingress.yaml
```
�}�d��
```shell
$ kubectl get ingress 
NAME            HOSTS            ADDRESS   PORTS     AGE
nginx-ingress   sam.nctu.me             80, 443   1h
```

## �إ�default-backend
���إߤ@?delfault,�p�G??��ip����request�άO?��?�w?��domain name,?�^?404

default-backend.yaml
```yaml=
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: default-http-backend
  labels:
    k8s-app: default-http-backend
  namespace: kube-system
spec:
  replicas: 1
  template:
    metadata:
      labels:
        k8s-app: default-http-backend
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: default-http-backend
        # Any image is permissable as long as:
        # 1. It serves a 404 page at /
        # 2. It serves 200 on a /healthz endpoint
        image: gcr.io/google_containers/defaultbackend:1.0
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        ports:
        - containerPort: 8080
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
          requests:
            cpu: 10m
            memory: 20Mi
---
apiVersion: v1
kind: Service
metadata:
  name: default-http-backend
  namespace: kube-system
  labels:
    k8s-app: default-http-backend
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    k8s-app: default-http-backend
```

```shell
$ kubectl create -f default-backend.yaml
```
## �إ�Ingress-Nginx-Controller
??��?�p�|,�b�إߤ��e�n��?�mRBAC�~??�Q�ذ_?,��F???�~??,�ҥH��?�n���إ߬�?��RBAC
ingress-controller-rbac.yaml
```yaml=
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: ingress
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  - secrets
  - services
  - endpoints
  - ingresses
  - nodes
  - pods
  verbs:
  - list
  - watch
- apiGroups:
  - "extensions"
  resources:
  - ingresses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - events
  - services
  verbs:
  - create
  - list
  - update
  - get
  - patch
- apiGroups:
  - "extensions"
  resources:
  - ingresses/status
  - ingresses
  verbs:
  - update
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: Role
metadata:
  name: ingress-ns
  namespace: kube-system
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - list
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - endpoints
  verbs:
  - get
  - create
  - update
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: ingress-ns-binding
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: ingress-ns
subjects:
  - kind: ServiceAccount
    name: ingress
    namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: ingress-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: ingress
subjects:
  - kind: ServiceAccount
    name: ingress
    namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ingress
  namespace: kube-system
```

```shell
$ kubectl create -f ingress-controller-rbac.yaml
```

���Z�N�i�H�إ�Ingress-controller
nginx-ingress-daemonset.yaml
```yaml=
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: nginx-ingress-controller
  labels:
    k8s-app: nginx-ingress-controller
  namespace: kube-system
spec:
  template:
    metadata:
      labels:
        k8s-app: nginx-ingress-controller
    spec:
      # hostNetwork makes it possible to use ipv6 and to preserve the source IP correctly regardless of docker configuration
      # however, it is not a hard dependency of the nginx-ingress-controller itself and it may cause issues if port 10254 already is taken on the host
      # that said, since hostPort is broken on CNI (https://github.com/kubernetes/kubernetes/issues/31307) we have to use hostNetwork where CNI is used
      # like with kubeadm
      hostNetwork: true
      terminationGracePeriodSeconds: 60
      serviceAccountName: ingress
      containers:
      - image: gcr.io/google_containers/nginx-ingress-controller:0.9.0-beta.3
        name: nginx-ingress-controller
        readinessProbe:
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
        livenessProbe:
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          timeoutSeconds: 1
        ports:
        - containerPort: 80
          hostPort: 80
        - containerPort: 443
          hostPort: 443
        env:
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
        args:
        - /nginx-ingress-controller
        - --default-backend-service=$(POD_NAMESPACE)/default-http-backend
```
```shell
$ kubectl create -f nginx-ingress-daemonset.yaml
```
�d�ݬO�_���إߧ���
```shell
$ kubectl get pod -n kube-system
nginx-ingress-controller-3q2b0           1/1       Running   0          1h
nginx-ingress-controller-j28sz           1/1       Running   0          1h
nginx-ingress-controller-lrn34           1/1       Running   0          1h

```
�M�Z�A?? ����?�J https://sam.nctu.tw �N��F, ?�M??�u�O??,?�o?���A?�w��domain name

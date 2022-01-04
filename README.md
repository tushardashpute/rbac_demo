# rbac_demo

**Assumption** : EKS cluster is ready and helm is installed

# Steps:

1.	Install springboot application
2.	Create user rbac-user
3.	MAP user to K8S
4.	Test new user
5.	Create role and binding
6.	Verify role and binding

# 1.Install Springboob application.

# helm install my-release helm-chart-spring

# helm ls 
    NAME      	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART       	APP VERSION         
    my-release	default  	1       	2022-01-04 14:59:11.020727812 +0000 UTC	deployed	spring-0.0.6	2.1.0.BUILD-SNAPSHOT


# k get all

    NAME                                     READY   STATUS    RESTARTS   AGE
    pod/my-release-spring-5b8f9bb6f5-lfxjw   1/1     Running   0          21s

    NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
    service/kubernetes          ClusterIP   10.100.0.1       <none>        443/TCP   94m
    service/my-release-spring   ClusterIP   10.100.116.236   <none>        80/TCP    21s

    NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/my-release-spring   1/1     1            1           21s

    NAME                                           DESIRED   CURRENT   READY   AGE
    replicaset.apps/my-release-spring-5b8f9bb6f5   1         1         1       21s


**2.Create user rbac-user**

aws iam create-user --user-name rbac-user
aws iam create-access-key --user-name rbac-user | tee /tmp/create_output.json

# aws iam create-user --user-name rbac-user
        {
            "User": {
                "UserName": "rbac-user", 
                "Path": "/", 
                "CreateDate": "2022-01-04T15:17:31Z", 
                "UserId": "AIDAYDLH#############XAIOP", 
                "Arn": "arn:aws:iam::556952635478:user/rbac-user"
            }
        }
# aws iam create-access-key --user-name rbac-user | tee /tmp/create_output.json
      {
          "AccessKey": {
              "UserName": "rbac-user", 
              "Status": "Active", 
              "CreateDate": "2022-01-04T15:17:31Z", 
              "SecretAccessKey": "oPPiIdSfA888b#################TTXtnoPPyZQTU", 
              "AccessKeyId": "AKIAYDLHWZBLPP#########"
          }
      }

3.	MAP user to K8S

# cat << EoF > rbacuser_creds.sh
      > export AWS_SECRET_ACCESS_KEY=$(jq -r .AccessKey.SecretAccessKey /tmp/create_output.json)
      > export AWS_ACCESS_KEY_ID=$(jq -r .AccessKey.AccessKeyId /tmp/create_output.json)
      > EoF

#kubectl get configmap -n kube-system aws-auth -o yaml | grep -v "creationTimestamp\|resourceVersion\|selfLink\|uid" | sed '/^  annotations:/,+2 d' > aws-auth.yaml

# cat aws-auth.yaml 
    apiVersion: v1
    data:
      mapRoles: |
        - groups:
          - system:bootstrappers
          - system:nodes
          rolearn: arn:aws:iam::556952635478:role/eksctl-eksdemo-nodegroup-eksdemo-NodeInstanceRole-349U9QCL9VHG
          username: system:node:{{EC2PrivateDNSName}}
    kind: ConfigMap
    metadata:
      name: aws-auth
      namespace: kube-system

# . rbacuser_creds.sh

# aws sts get-caller-identity

  {
      "Account": "556952635478", 
      "UserId": "AIDAYDLHW#######XXAIOP", 
      "Arn": "arn:aws:iam::556952635478:user/rbac-user"
  }

# kubectl get pods -n rbac-test

  Error from server (Forbidden): pods is forbidden: User "rbac-user" cannot list resource "pods" in API group "" in the namespace "default"

# unset AWS_SECRET_ACCESS_KEY
# unset AWS_ACCESS_KEY_ID

# aws sts get-caller-identity
  {
      "Account": "556952635478", 
      "UserId": "AROAYDLHW#######RLKU6KG:i-09ec02a03bfd55fb3", 
      "Arn": "arn:aws:sts::556952635478:assumed-role/ec2-role-for-creating-ekscluster/i-09ec02a03bfd55fb3"
  }


# cat << EoF > rbacuser-role.yaml
    > kind: Role
    > apiVersion: rbac.authorization.k8s.io/v1
    > metadata:
    >   namespace: default
    >   name: pod-reader
    > rules:
    > - apiGroups: [""] # "" indicates the core API group
    >   resources: ["pods"]
    >   verbs: ["list","get","watch"]
    > - apiGroups: ["extensions","apps"]
    >   resources: ["deployments"]
    >   verbs: ["get", "list", "watch"]
    > EoF

# cat << EoF > rbacuser-role-binding.yaml
    > kind: RoleBinding
    > apiVersion: rbac.authorization.k8s.io/v1
    > metadata:
    >   name: read-pods
    >   namespace: default
    > subjects:
    > - kind: User
    >   name: rbac-user
    >   apiGroup: rbac.authorization.k8s.io
    > roleRef:
    >   kind: Role
    >   name: pod-reader
    >   apiGroup: rbac.authorization.k8s.io
    > EoF

# kubectl apply -f rbacuser-role.yaml
  role.rbac.authorization.k8s.io/pod-reader created
  [root@ip-172-31-28-184 opt]# kubectl apply -f rbacuser-role-binding.yaml
  rolebinding.rbac.authorization.k8s.io/read-pods created

# . rbacuser_creds.sh; aws sts get-caller-identity
  {
      "Account": "556952635478", 
      "UserId": "AIDAY#########PLXXAIOP", 
      "Arn": "arn:aws:iam::556952635478:user/rbac-user"
  }


# k get pods
  NAME                                 READY   STATUS    RESTARTS   AGE
  my-release-spring-5b8f9bb6f5-lfxjw   1/1     Running   0          43m


[root@ip-172-31-28-184 opt]# kubectl get pods -n kube-system
  Error from server (Forbidden): pods is forbidden: User "rbac-user" cannot list resource "pods" in API group "" in the namespace "kube-system"



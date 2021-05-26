# Deploy Application CI/CD pipeline in AWS-EKS

Wordpress, MySQL을 이용해 어플리케이션을 만들고, CI/CD 파이프라인으로 AWS(EKS)에 배포합니다.

- Application
  - Wordpress
  - MySQL

- EKS
  - Deployment
    - Service(NodePort)
    - configmap
    - PVC
  - Statefulset
    - Service
    - configmap
    - PVC
  - Ingress
- CI
  - Git Registry: Git hub
  - Docker Registry : Docker hub
  - Docker Build : DockerHub Automated Build

- CD
  - Argo CD
- Monitoring 
  - Prometheus (Grafana)

## Application

Wordpress, MySQL을 이용해 어플리케이션을 만듭니다.

각각의 어플리케이션 이미지는 Dockerfile로 작성하였고, Git을 통해 Dockerhub로 이미지를 배포합니다.

> 도커 허브 레지스트리
>
> https://hub.docker.com/repository/docker/pogbia29/mywordpress
>
> https://hub.docker.com/repository/docker/pogbia29/mydb_mysql



![image-20210526054919252](C:\Users\pogbi\AppData\Roaming\Typora\typora-user-images\image-20210526054919252.png)

## EKS

#### EKS 클러스터 구성

#### 사전 요구사항

- aws cli 설치
- aws configure 설정
- kubectl 설치
- eksctl 설치



#### EKS 클러스터 생성

[testeks.yaml](https://github.com/pogbia/CICDwithEKS/blob/master/testeks.yaml)

```
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: cicd-eks
  region: ap-northeast-2
  version: "1.18"

# AZ
availabilityZones: 
  - ap-northeast-2a
  - ap-northeast-2b
  - ap-northeast-2c
  - ap-northeast-2d
...
```



#### 클러스터 구성

![image-20210526203626736](C:\Users\pogbi\AppData\Roaming\Typora\typora-user-images\image-20210526203626736.png)

- [Deployment](Deployment https://github.com/pogbia/CICDwithEKS/blob/master/manifest/deploy-wordpress.yaml )
  - [Service(NodePort)](https://github.com/pogbia/CICDwithEKS/blob/master/manifest/deploy-svc.yaml)
  - [configmap](https://github.com/pogbia/CICDwithEKS/blob/master/manifest/wordpress-configmap.yaml)
  - [PVC](https://github.com/pogbia/CICDwithEKS/blob/master/manifest/wordpress-pvc.yaml)
- [Statefulset](https://github.com/pogbia/CICDwithEKS/blob/master/manifest/sts-mysql.yaml)
  - [Service](https://github.com/pogbia/CICDwithEKS/blob/master/manifest/sts-svc.yaml)
  - [configmap](https://github.com/pogbia/CICDwithEKS/blob/master/manifest/mysql-configmap.yaml)
  - [PVC](https://github.com/pogbia/CICDwithEKS/blob/master/manifest/mysql-pvc.yaml)
- [Ingress](https://github.com/pogbia/CICDwithEKS/blob/master/manifest/deploy-ing.yaml)



#### Ingress Controller

[Document](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/aws-load-balancer-controller.html)

```
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
```

```
helm repo add eks https://aws.github.io/eks-charts
```

```
helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
  --set clusterName=[cluster-name] \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set image.tag=[version] \
  -n kube-system
```



#### EBS CSI

[Document](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html)

```
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update
```

```
helm upgrade -install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set enableVolumeResizing=true \
  --set enableVolumeSnapshot=true \
  --set serviceAccount.controller.create=false \
  --set serviceAccount.controller.name=ebs-csi-controller-sa
```



example ebs-sc.yaml

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
```

```
kubectl create -f ebs-sc.yaml
```





## CI/CD

### CI

1. Git Registry: Git hub 

2. Docker Registry : Docker hub

3. Docker Build : DockerHub Automated Build



### CD

- Argo CD

> https://argo-cd.readthedocs.io/en/stable/#getting-started



Argo CD 설치

```
kubectl create namespace argocd
```

```
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```



LoadBalancer 설정

```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}' 
```



로그인 정보

- ID : admin
- PW : `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`



Argo cd 접속

```
kubectl get svc argocd-server -n argocd

NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                                                                                   
argocd-server   LoadBalancer   10.100.67.150   a919076e78fa745508a332a5348de349-477343606.ap-northeast-2.elb.amazonaws.com
```



Argo CD 설정



**General**

`Application Name` : <App-name>

`Project` : <defalt>

`SYNC POLICY` : <Automatic>



**SOURCE**

`Repository URL` : <https://github.com/pogbia/CICDwithEKS.git>

`Revision` : <master>

`Path` : <manifest/>



**DESTINATION**
`Cluster URL` : <https://kubernetes.default.svc>

`Namespace` : <defualt>



Argo CD 배포

![image-20210526215004746](C:\Users\pogbi\AppData\Roaming\Typora\typora-user-images\image-20210526215004746.png)



---

#### wordpress 접속

```
kubectl get ingress wordpress-ing
```

```
NAME            CLASS    HOSTS   ADDRESS          
wordpress-ing   <none>   *     k8s-default-wordpres-6ea970997d-2047022698.ap-northeast-2.elb.amazonaws.com
```



*http://k8s-default-wordpres-6ea970997d-2047022698.ap-northeast-2.elb.amazonaws.com/*

![image-20210526221154634](C:\Users\pogbi\AppData\Roaming\Typora\typora-user-images\image-20210526221154634.png)



접속 확인

![image-20210526221442963](C:\Users\pogbi\AppData\Roaming\Typora\typora-user-images\image-20210526221442963.png)





## Monitoring

- Prometheus
- Grafana

[Document](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)



Prometheus 설치

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

```
helm install [RELEASE_NAME] prometheus-community/kube-prometheus-stack --namespace prometheus --create-namespace
```



로그인 정보

- ID: admin
- PW: prom-operator



![image-20210526220227363](C:\Users\pogbi\AppData\Roaming\Typora\typora-user-images\image-20210526220227363.png)





![image-20210526220327287](C:\Users\pogbi\AppData\Roaming\Typora\typora-user-images\image-20210526220327287.png)
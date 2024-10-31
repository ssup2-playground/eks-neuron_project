# eks-neuron Project

eks-neuron is prototyping project for testing inferentia/trainium instances based on EKS. eks-neuron project consists of the following git repositories.

* [aws-terraform](https://github.com/ssup2-playground/eks-neuron_aws-terraform) : Terraform for EKS cluster and inferentia/trainium instances.
* [serving-inf1-app](https://github.com/ssup2-playground/eks-neuron_serving-inf1-app) : ML model serving application based on inferentia 1 and FastAPI.

## Architecture

<img src="/images/architecture.png" width="800"/>

### "core" Managed Node Group

* Running CoreDNS and Karpenter

### "add-on" Karpenter NodePool

* Running prometheus server, my-scheduler and neuron-scheduler extender.
* When multiple inferentia cores are assigned to one pod, **my-scheduler** and **neuron-scheduler** extender plays the role of allocating cores consecutively.
* my-scheduler operates in active-standby mode and ensures high availability.
* neuron-scheduler operates in single-active mode and ensure high availability with recreation.
  * neuron-scheduler doesn't support active-active or active-standby mode.

### "inf" Karpenter NodePool

* Running serving app, neuron device plugin, neuron monitor and node problem detector.
* **Neuron device plugin** makes the kubelet aware of the inferentia core.
* **Neuron monitor** provides inferentia/trainium metrics to prometheus server. 
* **Node problem detector** detects failure inferentia/trainium cores.

## Serving Inf1 App

* based on inferentia 1 and FastAPI
* serving app uses **my-scheduler** and **neuron-scheduler** to allocate mutiple inferentia cores sequentially

* Get serving API endpoints
```shell
$ ENDPOINT_INF1=$(echo http://$(kubectl -n app get service serving-inf1 --output jsonpath='{.status.loadBalancer.ingress[0].hostname}'))
$ echo $ENDPOINT_INF1
http://k8s-app-servingi-6f94fb09a3-32c0217cb413f5b4.elb.ap-northeast-2.amazonaws.com
```

* API Examples
```shell
# Testing ResNet50 model
$ curl https://raw.githubusercontent.com/ssup2-playground/eks-neuron_serving-inf1-app/refs/heads/master/images/kitten.jpg -o kitten.jpg
$ curl https://raw.githubusercontent.com/ssup2-playground/eks-neuron_serving-inf1-app/refs/heads/master/images/tiger.jpg -o tiger.jpg
$ curl https://raw.githubusercontent.com/ssup2-playground/eks-neuron_serving-inf1-app/refs/heads/master/images/strawberry.jpg -o strawberry.jpg

# "/resnet50" API
# Processing a image on a inferentia core
$ curl -F "file=@kitten.jpg" $(echo $ENDPOINT_INF1/resnet50)
{"tabby":"0.5812537670135498","Egyptian_cat":"0.22762224078178406","tiger_cat":"0.10100676119327545","lynx":"0.07389812916517258","tiger":"0.010001023299992085"}
$ curl -F "file=@tiger.jpg" $(echo $ENDPOINT_INF1/resnet50)
{"tiger":"0.9340131282806396","tiger_cat":"0.05970945954322815","jaguar":"0.0014042318798601627","zebra":"0.0005853709881193936","tabby":"0.0003550454566720873"}%
$ curl -F "file=@strawberry.jpg" $(echo $ENDPOINT_INF1/resnet50)
{"strawberry":"0.9997598528862","banana":"5.1432507461868227e-05","pineapple":"3.762882261071354e-05","lemon":"2.144025893358048e-05","trifle":"1.3842871339875273e-05"}%

# "/resnet50_batch" API
# Processing mutiple images on multiple inferentia cores
# In order to use all inferentia cores uniformly, the number of images must be requested as a multiple of the number of inferentia cores assigned to the pod. 
$ curl -F "files=@kitten.jpg" -F "files=@tiger.jpg" -F "files=@strawberry.jpg" $(echo $ENDPOINT_INF1/resnet50_batch)
[{"tabby":"0.5812537670135498","Egyptian_cat":"0.22762224078178406","tiger_cat":"0.10100676119327545","lynx":"0.07389812916517258","tiger":"0.010001023299992085"},{"tiger":"0.9340131282806396","tiger_cat":"0.05970945954322815","jaguar":"0.0014042318798601627","zebra":"0.0005853709881193936","tabby":"0.0003550454566720873"},{"strawberry":"0.9997598528862","banana":"5.1432507461868227e-05","pineapple":"3.762882261071354e-05","lemon":"2.144025893358048e-05","trifle":"1.3842871339875273e-05"}]%
```

## Monitoring

### Login Grafana

* Set grafana NLB security Group
```
$ MY_IP=$(curl -s https://checkip.amazonaws.com/)
$ SG_ID=$(aws ec2 describe-security-groups --filters Name=tag:Name,Values=eks-neuron-grafana-sg --query "SecurityGroups[*].GroupId" --output text)
$ aws ec2 authorize-security-group-ingress --group-id "$SG_ID" --protocol tcp --port 80 --cidr "$MY_IP/32"
```

* Get grafana `admin` user password
```
$ kubectl -n observability get secrets grafana -o jsonpath='{.data.admin-password}' | base64 --decode
ugnQJC5Sgg3WkuHi7k8le4U3oB1f9EKhj2G4uS48
```

* Get grafana endpoint
```shell
$ echo http://$(kubectl -n observability get service grafana --output jsonpath='{.status.loadBalancer.ingress[0].hostname}')
http://k8s-observab-grafana-e4e76fb41d-a64e31df8c64616d.elb.ap-northeast-2.amazonaws.com
```

* Login Grafana

  <img src="/images/grafana-login.png" width="400"/>

  * ID : `admin`
  * Password : `kubectl -n observability get secrets grafana -o jsonpath='{.data.admin-password}' | base64 --decode ugnQJC5Sgg3WkuHi7k8le4U3oB1f9EKhj2G4uS48`

### Metric List

* Neuron metric example : [example](neuron-metric-example.txt) 
* Reference : https://awsdocs-neuron.readthedocs-hosted.com/en/latest/tools/neuron-sys-tools/neuron-monitor-user-guide.html

### Neuron Top

* Access inferentia instance via SSH or SSM
* Install and run neuron-top

```shell
$ yum install aws-neuronx-tools
$ neuron-top
```

<img src="/images/neuron-top.png" width="800"/>

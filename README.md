# Atolye15 Demo

Bu ekrana sadece Hello world yazdiran basit bir Nest.JS uygulamasi. Senden asagidaki sekilde bir pipeline olusturmani bekliyoruz;

Git repomuzda `master` ve `develop` branch'leri bulunuyor. Insanlar `develop` branch'ine feature branch'ler uzerinden yeni ozellikler ekleyebilirler. Bunun icin de soyle bir pipeline planliyoruz;

Kisi gerekli commit'leri attiktan sonra CI (Biz genelde CircleCI kullaniyoruz ama sen istedigini kullanabilirsin) aracinda sirasiyla asagidaki kontroller calisir;

- Lint kurallari geciyor mu: `yarn lint`
- Formatlama kurallarina uyuyor mu: `yarn format:check`
- Unit testler geciyor mu: `yarn test`
- Coverage threshold'una uyulmus mu?: `yarn test:cov`
- E2E testler geciyor mu: `yarn test:e2e`

Tum bu kontroller pass olduktan sonra PR `develop` ile birlestigi zaman senin yazmis oldugun bu projeye dahil edecegin `Dockerfile`'daki stepleri takip eden herhangi bir builder'da image build alip onu herhangi bir private container registy'e yollamani bekliyoruz. GCloud'ta oldugunu varsayarsak bu araclar cloud build ve GCR olacaktir. Sen istedigin cozumu kullanabilirsin.

Image registry'e gittikten sonra latest tag'li bu image'in Kubernetes tarafinda senin yazdigin manifestolara uygun olarak `stage` namespace'inde yayina girmesini istiyoruz. Bu asamada ilgili kisiye mail gidebilir. Daha sonra Git tarafinda `develop`'tan `master`'a PR acildiginda tum surec tekrar yukaridaki gibi isleyip en sonunda `production` namespace'inde Kubernetes uzerinde yayinda olmasini bekliyoruz.

Kubernetes tarafinda da Let's encrypt uzerinden auto provision ile SSL ayarlarsan da super olur.

Pipeline'in istedigin kismini es gecebilir veya kendince daha dogru oldugunu dusundugun bir hale getirebilirsin.

NOT: Dependency'lerin kurulmasi icin proje dizininde `yarn` komutunun calistirilmasi gerekiyor.

NOT: Uygulama `yarn start:prod` komutu ile ayaga 3000 portunda ayaga kalkiyor.


---------------------------

#### Set Default Region
```bash
gcloud config set compute/zone us-central1-c
```

#### Configure default project
```bash
gcloud config get-value core/project
export PROJECT_ID=<PROJECT_ID>
gcloud config set project $PROJECT_ID
```
#### Enable kuberntes APIs
```bash
gcloud services enable compute.googleapis.com container.googleapis.com
```
#### Create kubernetes clusters
```bash
gcloud container clusters create tools-cluster \
      --cluster-version=1.10 \
      --num-nodes 3 --machine-type n1-standard-2 --scopes cloud-platform

gcloud container clusters create staging-cluster \
      --cluster-version=1.10 \
      --num-nodes 3 --machine-type n1-standard-2 --scopes cloud-platform
gcloud container clusters list
```

```bash
NAME             LOCATION       MASTER_VERSION  MASTER_IP      MACHINE_TYPE   NODE_VERSION   NUM_NODES  STATUS
staging-cluster  us-central1-c  1.10.12-gke.1   35.224.225.27  n1-standard-2  1.10.12-gke.1  3          RUNNING
tools-cluster    us-central1-c  1.10.12-gke.1   35.202.81.130  n1-standard-2  1.10.12-gke.1  3          RUNNING
```

```bash
gcloud container clusters get-credentials tools-cluster --zone us-central1-c --project inspired-bus-194216
```

#### Install Helm
```bash
kubectl create clusterrolebinding user-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value account)
kubectl create serviceaccount tiller --namespace kube-system
kubectl create clusterrolebinding tiller-admin-binding --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account=tiller
helm update
```
#### Install Jenkins
```bash
helm install --name jenkins stable/jenkins --namespace omer -f values.yaml
#print password
 printf $(kubectl get secret --namespace omer jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
#connect jenkins
kubectl port-forward -n jenkins \
     $(kubectl get pods --namespace omer -l \
     "component=jenkins-jenkins-master" -o jsonpath="{.items[0].metadata.name}") \
     8080:8080
```
* Login to Jenkins from http://localhost:8080
* Create Pipeline
* Copy Jenkinsfile content to pipeline script panel


#### Cretae Service Account for managing kubernetes clusters
```bash
export PROJECT_ID=$(gcloud config get-value core/project)
 gcloud alpha iam service-accounts create jenkins-sa --display-name jenkins-sa
 gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member serviceAccount:jenkins-sa@$PROJECT_ID.iam.gserviceaccount.com --role roles/container.developer

gcloud iam service-accounts keys create ~/key.json \
  --iam-account jenkins-sa@$PROJECT_ID.iam.gserviceaccount.com

kubectl create secret generic k8s-developer-sa --from-file=$HOME/key.json  -n jenkins
```

* Build the jenkins pipeline


# Using Spinnaker to Deploy Applications to GKE 

This is a step by step guide about how to deploy applications to Kubernetes environments using Spinnaker as CD tool.

### Prerequisities

- Make sure GKE cluster is created.


### Spinnaker Installation Steps

Install Halyard on Ubuntu:

```
curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh
sudo bash InstallHalyard.sh
hal -v
```

Configuring Provider:

```
CONTEXT=$(kubectl config current-context)

kubectl apply --context $CONTEXT \
    -f https://spinnaker.io/downloads/kubernetes/service-account.yml

TOKEN=$(kubectl get secret --context $CONTEXT \
   $(kubectl get serviceaccount spinnaker-service-account \
       --context $CONTEXT \
       -n spinnaker \
       -o jsonpath='{.secrets[0].name}') \
   -n spinnaker \
   -o jsonpath='{.data.token}' | base64 --decode)

cp ~/.kube/config ~/.kube/config-backup

kubectl config set-credentials ${CONTEXT}-token-user --token $TOKEN

kubectl config set-context $CONTEXT --user ${CONTEXT}-token-user

mv ~/.kube/config ~/.kube/config-spinnaker

mv ~/.kube/config-backup ~/.kube/config

hal config provider kubernetes enable

CONTEXT=$(kubectl config --kubeconfig=/home/ubuntu/.kube/config-spinnaker current-context)

hal config provider kubernetes account add spinnaker-account \
    --provider-version v2 \
	--kubeconfig-file "/home/ubuntu/.kube/config-spinnaker" \
    --context $CONTEXT 
	
hal config features edit --artifacts true
```

Environment Setup:

```
hal config deploy edit --type distributed --account-name spinnaker-account
```

Google Storage Setup:

```
SERVICE_ACCOUNT_NAME=spinnaker-gcs-account
SERVICE_ACCOUNT_DEST=~/.gcp/gcs-account.json

gcloud iam service-accounts create \
    $SERVICE_ACCOUNT_NAME \
    --display-name $SERVICE_ACCOUNT_NAME
	
SA_EMAIL=$(gcloud iam service-accounts list \
    --filter="displayName:$SERVICE_ACCOUNT_NAME" \
    --format='value(email)')

PROJECT=$(gcloud info --format='value(config.project)')

gcloud projects add-iam-policy-binding $PROJECT \
    --role roles/storage.admin --member serviceAccount:$SA_EMAIL
	
mkdir -p $(dirname $SERVICE_ACCOUNT_DEST)

gcloud iam service-accounts keys create $SERVICE_ACCOUNT_DEST \
    --iam-account $SA_EMAIL

BUCKET_LOCATION=us
BUCKET=ace-spin
gsutil mb  -l $BUCKET_LOCATION gs://$BUCKET/


hal config storage gcs edit --project $PROJECT \
    --bucket-location $BUCKET_LOCATION \
    --json-path $SERVICE_ACCOUNT_DEST \
	--bucket $BUCKET

hal config storage edit --type gcs
```

Important!! If you get 403 error from Google Storage, make sure spinnaker-gcs-account is explicitly has Storage Admin role by visiting Google Cloud Storage Bucket Details/Permissions page.



Deploy Spinnaker:

```
hal version list
hal config version edit --version 1.12.1
hal deploy apply
```

Connect to Spinnaker:

Edit the spin-deck service, spin-gate and change their type from ClusterIP to LoadBalancer and note the LoadBalancer IPs. Then run:

```
hal config security ui edit --override-base-url http://spin-deck-LB-IP
hal config security api edit --override-base-url http://spin-gate-LB-IP
hal deploy apply
```

Providing Docker Registry:

```
ADDRESS=index.docker.io
REPOSITORIES=oozkan/atolye15
hal config provider docker-registry enable

hal config provider docker-registry account add test-docker-registry \
    --address $ADDRESS \
    --repositories $REPOSITORIES
hal deploy apply	
```

These steps are slightly different in other registries. 

Just to note, we could configure all the things above and could run "hal deploy apply" once.

### Creating a Pipeline

The steps:

- Open Spinnaker web UI and click applications tab.
- Create a new application.
- Click pipelines tab and configure a new pipeline.
- Click "Expected Artifacts" tab and set "Docker image" as index.docker.io/aozturk12/atolye15.
- Click "Automated Triggers" tab, add new trigger of type Docker registry, select your registry and select the oozkan/hello image. Please leave "tag" blank.
- For now deselect the "Trigger Enabled" option.
- Add a new stage of type Deploy(Manifest).
- Copy the contents of atolye15.yaml Notice that there is no "tag" part in image name.
- Make sure you've filled the "Req. Artifacts To Bind" option with Docker image artifact ID.
- Create "test" namespace in k8s.
- Start manual execution and select the tag you want to deploy. It will create the deployment and service.
- If you want new versions to be automatically deployed whenever a new Docker image is uploaded to the registry, you should select "Trigger Enabled" option obviously.

### References

https://www.spinnaker.io/setup/install/


### Spineaker Config Yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: atolye15
  namespace: master
spec:
  replicas: 3
  selector:
    matchLabels:
      app: atolye15
  template:
    metadata:
      labels:
        app: atolye15
    spec:
      containers:
        - image: index.docker.io/oozkan/atolye15
          name: hello
          ports:
            - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: atolye15
  namespace: test
spec:
  ports:
    - port: 80
      protocol: TCP
      targetPort: 3000
  selector:
    app: atolye15
  sessionAffinity: None
  type: LoadBalancer
```

### JenkinsFile

```
def label = "mypod-${UUID.randomUUID().toString()}"
podTemplate(label: label, containers: [
        containerTemplate(name: 'docker', image: 'docker', command: 'cat', ttyEnabled: true),
        containerTemplate(name: 'git', image: 'alpine/git', ttyEnabled: true, command: 'cat'),
        containerTemplate(name: 'gcloud', image: 'greenwall/gcloud-kubectl-helm', command: 'cat', ttyEnabled: true),
        containerTemplate(name: 'gcpdocker', image: 'paulwoelfel/docker-gcloud', ttyEnabled: true, command: 'cat'),
        containerTemplate(name: 'yarn', image: 'yarn:v1.21.1', command: 'cat', ttyEnabled: true),

    ],
    volumes: [
        hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
        secretVolume(secretName: 'k8s-developer-sa', mountPath: '/root/')
        ],
    envVars: [
            envVar(key: 'yarn', value: '/var/yarn/process.env.CHILD_CONCURRENCY '),
        ],
    ){
    node(label) {
        def GIT_ID = ""
        def PROJECT_ID="api-project--194216"
        stage('omer CI') {
            
            git branch:'master', url:'https://github.com/ozkomer/atolye15-demo.git'
            container('git'){
                stage('Extract Git ID'){
                    sh "git rev-parse --short HEAD > .git/commit-id"
                    IMAGETAG = readFile('.git/commit-id').trim()
                }
            }

        container('yarn lint check') {
           sh """
                yarn lint:tsc 
            """
        }
	container('yarn format check') {
           sh """
               yarn format:check
            """
        }
	container('yarn test') {
           sh """
               yarn test
            """
        }
	container('yarn coverage') {
           sh """
               yarn test:cov
            """
        }
	container('yarn e3e') {
           sh """
              yarn test:e2e
            """
        }
	container('docker') {
           sh """
                docker build -f cicd/jenkins/atolye15/Dockerfile -t gcr.io/${PROJECT_ID}/atolye15-helloworld:latest .
                docker tag gcr.io/${PROJECT_ID}/atolye15:latest gcr.io/${PROJECT_ID}/atolye15-helloworld:${IMAGETAG}
            """
        }

        stage('omer CD') {
            container('gcpdocker'){
                sh """
                    gcloud auth activate-service-account jenkins-sa@${PROJECT_ID}.iam.gserviceaccount.com --key-file=/root/key.json --project=${PROJECT_ID}
                    gcloud docker -- push  gcr.io/${PROJECT_ID}/spring-helloworld:latest
                    gcloud docker -- push  gcr.io/${PROJECT_ID}/spring-helloworld:${IMAGETAG}
                 """
            }
            container('gcloud'){
                sh  "sed -i -e 's/#VERSION#/${IMAGETAG}/g' ./cicd/jenkins/atolye15-spring-boot/k8s/deployment.yaml"
                //sh  "sed -i -e 's/#VERSION#/${IMAGETAG}/g' ./k8s/service.yaml"
            }
            container('gcloud'){
            sh """
                gcloud auth activate-service-account jenkins-sa@${PROJECT_ID}.iam.gserviceaccount.com --key-file=/root/key.json --project=${PROJECT_ID}
                gcloud container clusters get-credentials demo --zone us-central1-a --project ${PROJECT_ID}
                kubectl get deployment
                kubectl apply -n frontend -f ./cicd/jenkins/atolye15-spring--boot/k8s
               """
            }
        }
      }
    }
}
```
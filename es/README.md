# A GKE and Filestore Jenkins

Instalacion paso a paso de un Jenkins Comunity usando Helm en GKE con su "jenkins home" en un Filestore (Servicio de NFS en GCP) y la configuracion por medio de JCasC.

## Requirimientos

- Aprox una hora de tiempo
- Un proyecto GCP activado (con billing)
- Certificados SSL (pueden ser self-signed)
- GCloud, Kubectl y Helm v3 o superior
- Una taza de café/té a tu bebida favorita (opcional)

### Proyecto GCP y cuentas de servicio

Para facilidad en el paso a paso definiremos como variable de entorno los parametros com variables de entorno a medida que avanzaremos con el paso a paso, con el prefix JSBS (Jenkins Step by Step).

```bash
export JSBS_GOOGLE_PROJECT='--reemplazar-por-tu-id-proyecto--' #TODO
export JSBS_GOOGLE_SERVICE_ACCOUNT='jenkins-sa'

# configurar proyecto y creacion de cuenta de servicio para Jenkins
gcloud config set project $JSBS_GOOGLE_PROJECT

gcloud iam service-accounts create $JSBS_GOOGLE_SERVICE_ACCOUNT --display-name "jenkins service account"

gcloud projects add-iam-policy-binding $JSBS_GOOGLE_PROJECT \
    --member "serviceAccount:$JSBS_GOOGLE_SERVICE_ACCOUNT@$JSBS_GOOGLE_PROJECT.iam.gserviceaccount.com" \
    --role "roles/viewer"

gcloud projects add-iam-policy-binding $JSBS_GOOGLE_PROJECT \
    --member "serviceAccount:$JSBS_GOOGLE_SERVICE_ACCOUNT@$JSBS_GOOGLE_PROJECT.iam.gserviceaccount.com" \
    --role "roles/container.developer"
```

### Creacion del Google Filestore

```bash
export JSBS_GOOGLE_FILESTORE_INSTANCE_ID='jsbs-jenkins-home'
export JSBS_GOOGLE_FILESTORE_NETWORK='default'
export JSBS_GOOGLE_FILESTORE_ZONE='us-central1-c'
```

```bash
gcloud beta filestore instances create $JSBS_GOOGLE_FILESTORE_INSTANCE_ID \
    --project=$JSBS_GOOGLE_PROJECT --zone=$JSBS_GOOGLE_FILESTORE_ZONE \
    --tier=BASIC_HDD --file-share=name="jenkins_home",capacity=1024 \
    --network=name=$JSBS_GOOGLE_NETWORK
```

### Creacion de Cluster GKE

```bash
export JSBS_GOOGLE_GKE_INSTANCE_ID='jsbs-gke-cluster'
export JSBS_GOOGLE_GKE_NETWORK='default'
export JSBS_GOOGLE_GKE_SUBNETWORK='default'
export JSBS_GOOGLE_GKE_REGION='us-central1'
export JSBS_GOOGLE_GKE_YOUR_IP='--tu-ip--/32'
```

```bash
gcloud beta container --project $JSBS_GOOGLE_PROJECT clusters create $JSBS_GOOGLE_GKE_INSTANCE_ID --region $JSBS_GOOGLE_GKE_REGION --no-enable-basic-auth --cluster-version "1.19.7-gke.1500" --release-channel "rapid" --machine-type "e2-standard-4" --image-type "COS_CONTAINERD" --disk-type "pd-ssd" --disk-size "100" --metadata disable-legacy-endpoints=true --service-account "$JSBS_GOOGLE_SERVICE_ACCOUNT@$JSBS_GOOGLE_PROJECT.iam.gserviceaccount.com" --num-nodes "1" --enable-stackdriver-kubernetes --enable-ip-alias --network $JSBS_GOOGLE_GKE_NETWORK --subnetwork $JSBS_GOOGLE_GKE_SUBNETWORK --default-max-pods-per-node "110" --enable-autoscaling --min-nodes "0" --max-nodes "1" --enable-master-authorized-networks --master-authorized-networks $JSBS_GOOGLE_GKE_YOUR_IP --addons HorizontalPodAutoscaling,HttpLoadBalancing,GcePersistentDiskCsiDriver --enable-autoupgrade --enable-autorepair --max-surge-upgrade 1 --max-unavailable-upgrade 0 --enable-shielded-nodes
```

```bash
gcloud container clusters get-credentials $JSBS_GOOGLE_GKE_INSTANCE_ID --region $JSBS_GOOGLE_GKE_REGION --project $JSBS_GOOGLE_PROJECT
```

```bash
kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user="$JSBS_GOOGLE_SERVICE_ACCOUNT@$JSBS_GOOGLE_PROJECT.iam.gserviceaccount.com"
```

### Montar el PV

```bash
export JSBS_GOOGLE_GKE_NAMESPACE='jenkins'
```

TODO: editar con la ip del filestore

```bash
kubectl apply -f persistent-volume.yaml --namespace $JSBS_GOOGLE_GKE_NAMESPACE
```

TODO: corregir permiso en filestore

### install jenkins HELM

```bash
export JSBS_HELM_JENKINS_NAME='cd-jenkins'
```

```bash
kubectl get ns $JSBS_GOOGLE_GKE_NAMESPACE || kubectl create namespace $JSBS_GOOGLE_GKE_NAMESPACE

helm repo add jenkins https://charts.jenkins.io || true

helm install --wait $JSBS_HELM_JENKINS_NAME -f values.yaml jenkins/jenkins --namespace $JSBS_GOOGLE_GKE_NAMESPACE
```

### crear ingress

```bash
gcloud compute addresses create "$JSBS_HELM_JENKINS_NAME-ip" --global
kubectl apply -f ingress.yaml --namespace $JSBS_GOOGLE_GKE_NAMESPACE
```

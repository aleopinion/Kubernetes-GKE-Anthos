## Setting up Migrate For Anthos to migrate a monolith application to a micro-service application

https://github.com/GoogleCloudPlatform/acm-essentials


### To check and make sure the project matches with the project you are working on
```
gcloud config list
```

### 1 = To set an environment variable for Dev environment
```
export PROJECT_ID=$DEVSHELL_PROJECT_ID
```

### 2 = Creating Source Vm
```
gcloud compute  instances create   source-vm  --zone=us-central1-a --machine-type=e2-standard-2   --subnet=default --scopes="cloud-platform"   --tags=http-server,https-server --image=ubuntu-minimal-1604-xenial-v20210119a   --image-project=ubuntu-os-cloud --boot-disk-size=10GB --boot-disk-type=pd-standard   --boot-disk-device-name=source-vm \
  --metadata startup-script=METADATA_SCRIPT
```

### 3 = Creating Firewall Rule for the vm just created
```
gcloud compute firewall-rules create default-allow-http --direction=INGRESS --priority=1000 --network=default --action=ALLOW   --rules=tcp:80 --source-ranges=0.0.0.0/0 --target-tags=http-server
```

### 4 = Creating Migration Processing Cluster
```
gcloud container clusters create migration-processing   --project=$PROJECT_ID --zone=us-central1-a --machine-type e2-standard-4   --image-type ubuntu_containerd --num-nodes 3 --enable-stackdriver-kubernetes --subnetwork default
```
 
### 5 =  Creating Service Account
```
gcloud iam service-accounts create m4a-install \
  --project=$PROJECT_ID
```

### 6 = Assigning Administrator Role for Cloud Storage
```
gcloud projects add-iam-policy-binding $PROJECT_ID  \
  --member="serviceAccount:m4a-install@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.admin"
```

### 7 = To create the Service Account keys that it will use during the installation process
```
gcloud iam service-accounts keys create m4a-install.json \
  --iam-account=m4a-install@$PROJECT_ID.iam.gserviceaccount.com \
  --project=$PROJECT_ID
```
```
ls  => To list the keys that have been created
```
```
cat m4a-install.json => To see the content ot the key
```
 
### 8 = To log in to the Migration Processing Cluster
```
gcloud container clusters get-credentials migration-processing   --zone us-central1-a
```

### = To see the nodes that are part of your Processing Cluster
```
kubectl get nodes
```

### 9 = Setting up Actual Utilities or Componnents for Migrate for Anthos
```
migctl setup install --json-key=m4a-install.json --gcp-project $PROJECT_ID --gcp-region region us-central1
```

### Confirming you have the right configurations 
```
migctl doctor
```

### 10 = To create the 2nd Service Account
```
gcloud iam service-accounts create m4a-ce-src \
  --project=$PROJECT_ID
```
### 1 = To set an environment variable for Dev environment
```
export PROJECT_ID=$DEVSHELL_PROJECT_ID
```
### 8 = To log in to the Migration Processing Cluster
```
gcloud container clusters get-credentials migration-processing   --zone us-central1-a
```
### = To see the nodes that are part of your Processing Cluster
```
kubectl get nodes
```

### 10 = To create the 2nd Service Account
```
gcloud iam service-accounts create m4a-ce-src \
  --project=$PROJECT_ID
```

### 11 = To add compute viewer role to the 2nd service account
```
gcloud projects add-iam-policy-binding $PROJECT_ID  \
  --member="serviceAccount:m4a-ce-src@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/compute.viewer"
```

### 12 = To add a Cloud Storage role to the 2nd service account
```
gcloud projects add-iam-policy-binding $PROJECT_ID  \
  --member="serviceAccount:m4a-ce-src@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.admin"
```

### 13 = To Create the Service Account Key
```
gcloud iam service-accounts keys create m4a-ce-src.json \
  --iam-account=m4a-ce-src@$PROJECT_ID.iam.gserviceaccount.com \
  --project=$PROJECT_ID
```
```
ls => To display the keys just created
```

### 14 = Creating the Actual Migration Source
```
migctl source create ce source-vm --project $PROJECT_ID --json-key=m4a-ce-src.json
``` 
=>Error<=
```
migctl doctor => To check if you have a proper setup within your Processing Cluster
```
```
kubectl get pods -n kube-system => To get the number of pods running within the system namespace
```
```
kubectl get ns => To get the namespaces
```

### 15 = Activating Migration Plan using Migctl Utilities
```
migctl migration create my-migration --source source-vm   --vm-id source-vm --type linux-system-container		
```

### 16 = To Test the Migration Status
```
migctl migration status my-migration 
```
```
output:  This is what you'll see after you run the above command
Name 	       TYPE                 	CURRENT-OPERATION	    PROGRESS	STEP	    STATUS		AGE
my-migration linux-system-container GenerateMigrationPlan [3/3]	    Discovery	Running	 	59s
```

### 17 = To view or get the Migration plan that we created/generated in yaml format 
```
migctl migration get my-migration
```
```
ls => To view the migration plan => 2 files generated = my-migration.data.yaml and my-migration.yaml
```
```
cat my-migration.data.yaml => The copy of the migration plan that you can play with
```
```
cat my-migration.yaml => Final Copy of the migration plan
```
```
migctl migration plan my-migration => Shows you some of the different migctl commands that you can run 
```

### 18 = Used to generate actual Deployment Artifacts
```
migctl migration generate-artifacts my-migration
```
 
### 19 = To see the status of the migration artifacts generation
```
migctl migration status my-migration 
```
```
output: This is what you'll see after you run the above command
Name 	        TYPE	                  CURRENT-OPERATION	PROGRESS  STEP	              STATUS		   AGE
my-migration  linux-system-container  GenerateArtifacts [1/1]	    GenerateArtifacts	  Completed    59s
```

### 20 = If you want to gain more output about the status => Verbosity
```
migctl migration status my-migration -vvv 
```
```
ls 
```

### 21 => This file exposes the source-vm http web app to the public
```
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  name: source-vm
spec:
  clusterIP: None
  selector:
    app: source-vm
  type: ClusterIP
status:
  loadBalancer: {}
---
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: source-vm
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```
```
ls
```
```
cat http-m4a-service.yaml
```
  
### 22 = This command targets the service manifest and executes the 2 services within the manifest
```
kubectl apply -f http-m4e-servive.yaml  or  kubectl create -f http-m4a-service.yaml
```

###
Copy External IP paste on Browser  => Run it

###
```
kubectl get svc
```

###
```
migctl docker-registry list
````

END
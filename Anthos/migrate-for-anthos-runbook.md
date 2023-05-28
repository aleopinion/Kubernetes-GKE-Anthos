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

### 2a = Creating Source VM - where the application that we are migrating reside
```
gcloud compute  instances create   source-prod-vm  --zone=us-central1-a --machine-type=e2-standard-2   --subnet=default --scopes="cloud-platform"   --tags=http-server,https-server --image=ubuntu-minimal-1604-xenial-v20210119a   --image-project=ubuntu-os-cloud --boot-disk-size=10GB --boot-disk-type=pd-standard   --boot-disk-device-name=source-vm \
--metadata startup-script=METADATA_SCRIPT
```
### 2b = Creating Source VM - where the application that we are migrating reside
```
gcloud compute  instances create   source-prod-vm  \
--zone=us-central1-a \
--machine-type=e2-standard-2   \
--subnet=default \
--scopes="cloud-platform"   \
--tags=http-server,https-server \
--image=ubuntu-minimal-1604-xenial-v20210119a   \
--image-project=ubuntu-os-cloud \
--boot-disk-size=10GB \
--boot-disk-type=pd-standard   \
--boot-disk-device-name=source-vm \
--metadata startup-script=METADATA_SCRIPT

```

### 2c = Creating Source VM - where the application that we are migrating reside => Using update from Google Bard for the application
```
gcloud compute  instances create   source-prod-vm  \
--zone=us-central1-a \
--machine-type=e2-standard-2   \
--subnet=default \
--scopes="cloud-platform"   \
--tags=http-server,https-server \
--image=ubuntu-minimal-1604-xenial-v20210119a   \
--image-project=ubuntu-os-cloud \
--boot-disk-size=10GB \
--boot-disk-type=pd-standard   \
--boot-disk-device-name=source-vm \
--metadata startup-script="echo '
#!/bin/bash

sudo apt-get update && sudo apt-get install -y docker

docker run -d -p 8080:80 hello-world
' > /tmp/startup.sh && sudo bash /tmp/startup.sh"

```
### The different commands that you can use to manage containers at the cluster level
```
gcloud containers cluster list
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
  --member=serviceAccount:m4a-install@$PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/owner
```

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

### => To list the keys that have been created
```
ls  
```

### => To see the content ot the key
```
cat m4a-install.json 
```
 
### 8 = To log into the Migration Processing Cluster
```
gcloud container clusters get-credentials migration-processing   --zone us-central1-a
```

### = To see the nodes that are part of your Processing Cluster
```
kubectl get nodes
```

### 9 = Setting up Actual Utilities or Componnents for Migrate for Anthos (migctl)
```
migctl setup install --json-key=m4a-install.json --gcp-project $PROJECT_ID --gcp-region region us-central1
```

### The Command to Uninstall migctl from the processing cluster
```
gcloud container hub migctl uninstall --project=[PROJECT_ID] --location=[LOCATION]
```
```
migctl uninstall --project=$PROJECT_ID --location=us-central1
```

### To Confirm/validate/verify the installation of all the utilities for migrate For Anthos - To confirm you have the right configurations 
```
migctl doctor
```

### To check the different clusters in your project 
```
gcloud container clusters list
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

### => To display the keys just created
```
ls 
```

### 14 = To Configure the Source
```
migctl source create ce source-anthos-vm --project $PROJECT_ID --json-key=m4a-ce-src.json
``` 
=>Error<=

### => To check if you have a proper setup within your Processing Cluster
```
migctl doctor 
```

### => To get the number of pods running within the system namespace
```
kubectl get pods -n kube-system 
```

### => To get the namespaces
```
kubectl get ns 
```

### 15 = Activating Migration Plan using Migctl Utilities
```
migctl migration create my-migration --source source-prod-vm   --vm-id source-prod-vm --type linux-system-container		
```

### 16 = To Test the Migration Status
```
migctl migration status my-migration 
```
   output:  This is what you'll see after you run the above command
   Name 	       TYPE                 	 CURRENT-OPERATION	    PROGRESS	STEP	     STATUS		  AGE
   my-migration  linux-system-container  GenerateMigrationPlan  [3/3]	    Discovery	 Running	 	59s

### 17 = To get or view the Migration plan that we created/generated in yaml format
```
migctl migration get my-migration
```

### => To view the 2 files generated from the migration plan => my-migration.data.yaml and my-migration.yaml
```
ls 
```

### => The copy of the migration plan that you can play with
```
cat my-migration.data.yaml 
```

### => Final Copy of the migration plan
```
cat my-migration.yaml 
```

### => Shows you some of the different migctl commands that you can run
```
migctl migration plan my-migration  
```

### 18 = Used to generate actual Deployment Artifacts from the Migration plan
```
migctl migration generate-artifacts my-migration
```
 
### 19 = To see the status of the migration artifacts generation
```
migctl migration status my-migration 
```
output: This is what you'll see after you run the above command
Name 	        TYPE	                  CURRENT-OPERATION	PROGRESS  STEP	              STATUS		   AGE
my-migration  linux-system-container  GenerateArtifacts [1/1]	    GenerateArtifacts	  Completed    59s

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
kubectl apply -f http-m4e-servive.yaml   
```
```
kubectl create -f http-m4a-service.yaml
```

###
Copy External IP paste on Browser 

###
```
kubectl get svc
```
```
kubectl get services
```

### To show the registry that we are intarracting with 
```
migctl docker-registry list
````

END
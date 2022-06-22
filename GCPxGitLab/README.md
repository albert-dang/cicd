#  CICD GCPxGitLab
This is an updated quickstart for existing projects based on Neil Kolban's [guide on Medium](https://medium.com/google-cloud/using-gitlab-and-cloud-build-to-achieve-ci-cd-for-cloud-run-4c6db26f04ed).

This pipeline will build and deploy on every commit pushed to a GitLab repo. It can be configured to trigger on pushes to the main branch, requiring a product leader to approve final merges to the main branch for a crude "staging" layer.

**Before we begin:**
 1. We should have an existing GitLab instance with traffic enabled over port 22
 2. We should have an existing GitLab repo and "Maintainer" access or higher
 3. We should have an existing GCP project with "Owner" access
 
 **What we will achieve:**
 1. Store an SSH key for Cloud Build in Secret Manager
 2. Set up a Cloud Build Trigger which can be called from GitLab
 3. Set up a Webhook on GitLab which calls the Cloud Build Trigger on a new push

# [Artifact Registry](https://console.cloud.google.com/artifacts)
Note that if we wish to use an existing repository which is already configured for Docker and Google-managed keys, this step is not necessary.

Navigate to the [Artifact Registry](%28https://console.cloud.google.com/artifacts%29) and create a new repository with the following configuration:

	Name: 				my-repo
	Format: 			Docker
	Location type:		Region
	Region: 			us-east4
	Encryption:			Google-managed encryption key
Or use the following command in cloud shell:

	gcloud artifacts repositories create my-repo \
	--repository-format=docker \
	--location=us-east4
If we do not specify a region with the --location flag in cloud shell, it will use our default region.

# [Secret Manager](https://console.cloud.google.com/security/secret-manager)
Note that if we can share an existing SSH key for GitLab, we can skip this step.

Navigate to the Secret Manager and enable the API. At the top, we can hit "+ CREATE SECRET" and name our secret
	
	gitlab-key

For the secret value, we will generate a private key to SSH into GitLab. Activate Cloud Shell using the command line icon at the top-right. Once our command line is booted up, enter the following command:
	
	cd ~/.ssh  
	ssh-keygen -t ed25519 -f gitlab-key -q -N ""  
	cat << EOF >> config  
	Host [GITLAB_IP]  
	  IdentityFile ~/.ssh/gitlab-key  
	EOF

Since we are now in the .ssh directory where keys are stored, we can run the next command to print our private key:
	
	cat gitlab-key
Copy the entire block, within and including:

	-----BEGIN OPENSSH PRIVATE KEY-----
	-----END OPENSSH PRIVATE KEY-----

We can paste the private key into the "Secret value" field. We can leave all other defaults and hit "CREATE SECRET".

Next, we need to print our public key for GitLab:
	
	cat gitlab-key.pub

Copy the entire line,
	
	ssh-ed25519 ... user@cs-334698-default

We need to navigate to our GitLab instance and go to "Edit profile" from the dropdown menu at the top-right. On the left navigation bar, head to "SSH Keys". Here, we can paste the public key into the "Key" field, edit the Title if we want, and hit "Add key". 

# [Cloud Build Trigger](https://console.cloud.google.com/cloud-build/triggers)
Let's break down the steps in our trigger configuration.

First, we use ssh-keyscan to build our known_hosts

	steps:
		- name: gcr.io/cloud-builders/git
		  args:
			- '-c'
			- |
			  echo "$$SSHKEY" > /root/.ssh/id_rsa
			  chmod 400 /root/.ssh/id_rsa
			  ssh-keyscan ${_GITLAB_IP} > /root/.ssh/known_hosts
		  entrypoint: bash
		  secretEnv:
			- SSHKEY
		  volumes:
			- name: ssh
			  path: /root/.ssh

Then, we clone the GitLab repository over SSH

		- name: gcr.io/cloud-builders/git
			args:
				- clone
				- 'git@${_GITLAB_IP}:${_GITLAB_NAME}.git'
				- .
			volumes:
				- name: ssh
				  path: /root/.ssh

Next, build the image using Cloud Build
	
		- name: gcr.io/cloud-builders/docker
			args:
				- build
				- '-t'
				- '${_ARTIFACT_REPO}'
				- .

Push the image
	
		- name: gcr.io/cloud-builders/docker
			args:
				- push
				- '${_ARTIFACT_REPO}'

Then, deploy the container with Cloud Run

		- name: gcr.io/google.com/cloudsdktool/cloud-sdk
			args:
				- run
				- deploy
				- my-deployment
				- '--image'
				- '${_ARTIFACT_REPO}'
				- '--port'
				- '80'
				- '--region'
				- us-east4
				- '--allow-unauthenticated''
			entrypoint: gcloud

We can define values for the variables in our configuration. This is useful in case we want to configure these details at build time or re-use this template with different values.

		substitutions:
			_GITLAB_IP: 12.123.123.12
			_ARTIFACT_REPO: region-docker.pkg.dev/project-id/my-repo/my-image
			_GITLAB_NAME: group/project
			_PROJECT_NUMBER: '123456789012'

_GITLAB_IP is an address where the code repository can be cloned from, typically this is the external IP of our GitLab instance.

_ARTIFACT_REPO is the full path for the container image in the [artifact registry](https://console.cloud.google.com/artifacts),  which can be found by clicking on the repository or image name and using the copy button. If no image exists yet, we can append any name to the repository path for the new images.
An image called "my-image" inside an Artifact Registry repository called "my-repo", in the North America northeast2 region would have a path similar to:

	northamerica-northeast2-docker.pkg.dev/project-257811/my-repo/my-image

_GITLAB_NAME is a partial path for the GitLab repository, we can find this from the "Clone" button on the project under "Clone with SSH"
	
	git@example.io:[group/project].git

_PROJECT_NUMBER can be found in the project info card on our [dashboard](https://console.cloud.google.com/home/dashboard) and [welcome page](https://console.cloud.google.com/welcome)

Lastly, we specify the secret and version for the GitLab SSH private key:
		
		availableSecrets:
			secretManager:
				- versionName: 'projects/${_PROJECT_NUMBER}/secrets/gitlab-key/versions/1'
			env: SSHKEY
This specifies the secret we named "gitlab-key" and uses version 1

# [IAM & Admin](https://console.cloud.google.com/iam-admin/)

Ensure that the Cloud Build Service Account 
		
	[project-number]@cloudbuild.gserviceaccount.com 
has the following Roles:

 - Cloud Build Service Account
 - Cloud Run Admin
 - Logs Writer
 - Secret Manager Secret Accessor

Cloud Build Service Account should be granted by default.

Cloud Run Admin grants permissions to deploy images to cloud run.

Logs Writer grants permissions for logging.

Secret Manager Secret Accessor lets the Service Account access the private SSH key.

Under [Service Accounts](https://console.cloud.google.com/iam-admin/serviceaccounts), we will add the same Cloud Build service account as a principal for the Compute Engine service account:

    [project-number]-compute@developer.gserviceaccount.com
We can click on the Service Account email to view details, navigate to the "Permissions" tab and hit "Grant Access".

Under "New principals", we'll add the Cloud Build Service Account from earlier.

Under "Role", we'll choose Service Account User and hit "Save".

# GitLab Webhook
We need to create a Webhook which will call the Cloud Build Trigger at its URL. Navigate to the GitLab project which we are automating. On the left navigation bar, we will go to Settings > Webhooks

For the URL, we need to use the Webhook URL from the [Cloud Build Trigger](https://console.cloud.google.com/cloud-build/triggers) we created earlier. We can find this by clicking on the trigger name or going to "Edit" the trigger. Under "Webhook URL", we can hit "SHOW URL PREVIEW" to get something similar to:

    https://cloudbuild.googleapis.com/v1/projects/
    project-id/triggers/trigger:webhook?key=...&secret=...

We can paste this into the "URL" field in GitLab.

With "Push events" checked, we will leave all other defaults unless we want to set up crude staging. This can be done by specifying the "main" branch next to "Push events". We would then push to new branches and only build and deploy on a successful merge.

We can hit "Add webhook". Since we are now done configuring all of our pipeline, we can use the "Test" feature right from GitLab to simulate "Push events".

Navigate to the [Cloud Build history](https://console.cloud.google.com/cloud-build/builds) to see if validate the pipeline or troubleshoot errors.
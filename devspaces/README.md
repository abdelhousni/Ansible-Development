# Software defined development environment on Kubernetes

OpenShift Dev Spaces Docs: https://access.redhat.com/documentation/en-us/red_hat_openshift_dev_spaces/3.10

Eclipse Che Docs: https://eclipse.dev/che/docs/stable/overview/introduction-to-eclipse-che/

Eclipse Che Repo: https://github.com/eclipse-che/che-server

# Modifying Permissions For Ansible-Lint And Bash To Work In OpenShift Dev Spaces or Eclipse Che

1) Modify the ContainerFile to utilize your Execution Environment. Make any changes to .bashrc for your terminal requirements.

2) Run 
```
podman build --tag YOURIMAGENAMEANDTAG -f Containerfile 
```
For example
```
podman build --tag quay.io/rhn_ssp_adworjan/ansibleee:1.0 -f Containerfile 
```
to modify your existing EE to work in Dev Spaces. 

3) Push the image to your container registry

4) Modify the devfile.yaml and adjust for your registry and image

## Note: You could also take the two COPY steps and two RUN steps and include them in your execution-environment.yml to make this part of your EE build process
https://github.com/shadowman-lab/Ansible-PAH/tree/main/roles/build_shadowmandevspaces

# Set Up Git Config:

## OpenShift Dev Spaces <3.10 or Eclipse Che <7.75

1) Login to OpenShift

2) Go to Workloads, ConfigMaps

3) Click Create ConfigMap

4) Paste and update the below yaml
```
kind: ConfigMap
apiVersion: v1
metadata:
  name: workspace-userdata-gitconfig-configmap
  namespace: <user_namespace> 
  labels:
    controller.devfile.io/mount-to-devworkspace: 'true'
    controller.devfile.io/watch-configmap: 'true'
  annotations:
    controller.devfile.io/mount-as: subpath
    controller.devfile.io/mount-path: /etc/
data:
  gitconfig: "[user] \n  name = <git_user_name> \n  email = <git_user_email>" 
```
Visit https://<openshift_dev_spaces_fqdn>/api/kubernetes/namespace to get your OpenShift Dev Spaces user_namespace and update your git_user_name and git_user_email

5) Click Create

6) You can verify once a new workspace is started by opening a new terminal and running

```
git config --get-regexp user.*
```

Your Git user name and email should appear in the output.

### Note if a workspace-userdata-gitconfig-configmap already exists, just update your existing config map per the YAML above

## OpenShift Dev Spaces >=3.10 or Eclipse Che >=7.75

1) Login to OpenShift Dev Spaces

2) Click username in top right and select User Preferences

3) Select Gitconfig

4) Click the Pencil icon next to name, update your Git name, and click the check icon

5) Click the Pencil icon next to email, update your Git email, and click the check icon

6) You can verify once a new workspace is started by opening a new terminal and running

```
git config --get-regexp user.*
```

Your Git user name and email should appear in the output.

# Pull From Password Protected Registry

1) Login to OpenShift Dev Spaces

2) Click username in top right and select User Preferences

3) Under Container Registries select Add Container Registry

4) Enter registry (without https) and username / password

5) Click Add

# Pull From Password Protected Repository / Set Up PAT To Push To Repository

1) Login to OpenShift Dev Spaces

2) Click username in top right and select User Preferences

3) Under Personal Access Tokens select Add Token

4) Provide a token name, select the provider, update the provider endpoint, and enter in your Personal Access Token acquired from your repository provider in Token

5) Click Add

# Set Up SSH Keys

## OpenShift Dev Spaces >=3.10 or Eclipse Che >=7.77

1) Login to OpenShift Dev Spaces

2) Click username in top right and select User Preferences

3) Under SSH Keys select Add SSH Key

4) Select Upload next to Private Key, browse and select your Private Key

5) Select Upload next to Public Key, browse and select your Public Key

6) Click Add

# Auto Install Extensions:

1) At the root of your repository, create a .vscode folder with a file called extensions.json

2) Add any desired extensions with the full extension name

```
{
    "recommendations": [
      "redhat.ansible",
      "redhat.vscode-yaml"
    ]
}
```
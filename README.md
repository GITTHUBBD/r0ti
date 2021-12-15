# Log4j Vulnerability (CVE-2021-44228)

This repo contains operational information regarding the vulnerability in the Log4j logging library (CVE-2021-44228). For additional information see:

* [NCSC-NL advisory](https://www.ncsc.nl/actueel/advisory?id=NCSC-2021-1052)
* [MITRE](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-44228)

## Repository contents

| Directory                          | Purpose |
|:-----------------------------------|:--------|
| [iocs](iocs/README.md)             | Contains any Indicators of Compromise, such as scanning IPs, etc |
| [mitigation](mitigation/README.md) | Contains info regarding mitigation, such as regexes for detecting scanning activity and more |
| [scanning](scanning/README.md)     | Contains references to methods and tooling used for scanning for the Log4j vulnerability |
| [software](software/README.md)     | Contains a list of known vulnerable and not vulnerable software |

**Please note that these directories are not complete, and are currently being expanded.**

**NCSC-NL has published a HIGH/HIGH advisory for the Log4j vulnerability. Normally we would update the HIGH/HIGH advisory for vulnerable software packages, however due to the extensive amounts of expected updates we have created a list of known vulnerable software in the software directory.**

## Contributions welcome

If you have any additional information to share relevant to the Log4j vulnerability, please feel free to open a Pull request. New to this? [Read how to contribute in GitHub's documentation](https://docs.github.com/en/repositories/working-with-files/managing-files/editing-files#editing-files-in-another-users-repository).
# This workflow will build a docker container, publish it to IBM Container Registry, and deploy it to IKS when there is a push to the main branch.
#
# To configure this workflow:
#
# 1. Ensure that your repository contains a Dockerfile
# 2. Setup secrets in your repository by going to settings: Create ICR_NAMESPACE and IBM_CLOUD_API_KEY
# 3. Change the values for the IBM_CLOUD_REGION, REGISTRY_HOSTNAME, IMAGE_NAME, IKS_CLUSTER, DEPLOYMENT_NAME, and PORT

name: Build and Deploy to IKS

on:
  push:
    branches:
      - main

# Environment variables available to all jobs and steps in this workflow
env:
  GITHUB_SHA: ${{ github.sha }}
  IBM_CLOUD_API_KEY: ${{ secrets.IBM_CLOUD_API_KEY }}
  IBM_CLOUD_REGION: us-south
  ICR_NAMESPACE: ${{ secrets.ICR_NAMESPACE }}
  REGISTRY_HOSTNAME: us.icr.io
  IMAGE_NAME: iks-test
  IKS_CLUSTER: example-iks-cluster-name-or-id
  DEPLOYMENT_NAME: iks-test
  PORT: 5001

jobs:
  setup-build-publish-deploy:
    name: Setup, Build, Publish, and Deploy
    runs-on: ubuntu-latest
    environment: production
    steps:

    - name: Checkout
      uses: actions/checkout@v2

    # Download and Install IBM Cloud CLI
    - name: Install IBM Cloud CLI
      run: |
        curl -fsSL https://clis.cloud.ibm.com/install/linux | sh
        ibmcloud --version
        ibmcloud config --check-version=false
        ibmcloud plugin install -f kubernetes-service
        ibmcloud plugin install -f container-registry

    # Authenticate with IBM Cloud CLI
    - name: Authenticate with IBM Cloud CLI
      run: |
        ibmcloud login --apikey "${IBM_CLOUD_API_KEY}" -r "${IBM_CLOUD_REGION}" -g default
        ibmcloud cr region-set "${IBM_CLOUD_REGION}"
        ibmcloud cr login

    # Build the Docker image
    - name: Build with Docker
      run: |
        docker build -t "$REGISTRY_HOSTNAME"/"$ICR_NAMESPACE"/"$IMAGE_NAME":"$GITHUB_SHA" \
          --build-arg GITHUB_SHA="$GITHUB_SHA" \
          --build-arg GITHUB_REF="$GITHUB_REF" .

    # Push the image to IBM Container Registry
    - name: Push the image to ICR
      run: |
        docker push $REGISTRY_HOSTNAME/$ICR_NAMESPACE/$IMAGE_NAME:$GITHUB_SHA

    # Deploy the Docker image to the IKS cluster
    - name: Deploy to IKS
      run: |
        ibmcloud ks cluster config --cluster $IKS_CLUSTER
        kubectl config current-context
        kubectl create deployment $DEPLOYMENT_NAME --image=$REGISTRY_HOSTNAME/$ICR_NAMESPACE/$IMAGE_NAME:$GITHUB_SHA --dry-run -o yaml > deployment.yaml
        kubectl apply -f deployment.yaml
        kubectl rollout status deployment/$DEPLOYMENT_NAME
        kubectl create service loadbalancer $DEPLOYMENT_NAME --tcp=80:$PORT --dry-run -o yaml > service.yaml
        kubectl apply -f service.yaml
        kubectl get services -o wide
- name: Python3.8 PyInstaller Linux AMD64
  # You may pin to the exact commit or the version.
  # uses: action-python/pyinstaller-py3.8@1839eaaab51a950d6a80bf199c2527cfb045421b
  uses: action-python/pyinstaller-py3.8@v1.0.0
  with:
    # Directory containing source code & .spec file (optional requirements.txt).
    path: # default is src
    # Specify a custom URL for PYPI
    pypi_url: # optional, default is https://pypi.python.org/
    # Specify a custom URL for PYPI Index
    pypi_index_url: # optional, default is https://pypi.python.org/simple
    # Specify a file path for .spec file
    spec: # optional, default is 
    # Specify a file path for requirements.txt file
    requirements: # optional, default is requirements.txt
    # Rename the binary file, this will only work when you have --onefile binary.
    rename: # optional, default is 

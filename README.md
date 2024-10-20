# Application Deployment with Supabase on DigitalOcean Kubernetes

This repository contains a Next.js application deployed using Helm on a DigitalOcean Kubernetes cluster. The application uses Supabase as the backend for storing user data. The deployment is managed via GitHub Actions workflows, ensuring automated builds, deployments, and environment management using ConfigMaps. Additionally, Route53 is used for domain management, and SSL/TLS is handled via Let's Encrypt for secure production access.

## Table of Contents
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Deployment](#deployment)
- [Environment Configuration](#environment-configuration)
- [GitHub Workflows](#github-workflows)
- [Helm Configuration](#helm-configuration)
- [SSL/TLS with Let's Encrypt](#ssltls-with-lets-encrypt)
- [Domain Management with Route53](#domain-management-with-route53)
- [Running Locally](#running-locally)

## Features
- **Next.js Application**: A powerful frontend built using Next.js.
- **Supabase Backend**: Supabase is used as the backend for storing user data.
- **Kubernetes Deployment**: The application is deployed on a DigitalOcean Kubernetes cluster.
- **Helm for Deployment**: Managed Kubernetes resources using Helm charts for easier deployment and updates.
- **GitHub Actions**: Automated CI/CD pipeline using GitHub Actions for testing, building, and deploying the application.
- **Environment Management**: ConfigMaps are used for managing environment-specific configurations (e.g., development, staging, production).
- **SSL/TLS Encryption**: Let's Encrypt is used to provide SSL/TLS certificates for secure communication.
- **Domain Management**: Integrated with AWS Route53 for domain routing.

## Tech Stack
- **Frontend**: [Next.js](https://nextjs.org/)
- **Backend**: [Supabase](https://supabase.io/)
- **CI/CD**: [GitHub Actions](https://github.com/features/actions)
- **Kubernetes**: [DigitalOcean Kubernetes](https://www.digitalocean.com/products/kubernetes/)
- **Deployment**: [Helm](https://helm.sh/)
- **DNS Management**: [AWS Route53](https://aws.amazon.com/route53/)
- **SSL/TLS**: [Let's Encrypt](https://letsencrypt.org/)

## Deployment
### Prerequisites
- DigitalOcean account with a Kubernetes cluster and kubeconfig.
- Supabase project with the necessary tables and configurations.
- AWS account with Route53 for domain management.
- SSL certificates from Let's Encrypt.

### Steps
1. **Clone the Repository**:
   ```bash
   git clone https://github.com/your-repo/nextjs-supabase-app.git
   cd nextjs-supabase-app
   ```

2. **Configure Helm**: 
   Customize the `values.yaml` file for Helm with your environment-specific configurations.

3. **Setup GitHub Secrets**:
   - Add necessary secrets to your GitHub repository (e.g., DigitalOcean API Token, AWS credentials, Supabase keys).

4. **Configure Route53**:
   Set up a hosted zone in AWS Route53 for your domain and link it with your DigitalOcean cluster.

5. **Run GitHub Workflow**:
   Push changes to your branch and let the GitHub Actions workflow handle the deployment:
   - The workflow will build the Next.js application, create a Docker image, push it to Docker Hub, and deploy it to the Kubernetes cluster using Helm.

## Environment Configuration
Different environments are managed using Kubernetes ConfigMaps. Each environment (development, staging, production) has its own set of configurations (API keys, URLs, etc.). You can find these configurations in the `config` folder.

## GitHub Workflows
This project uses GitHub Actions to automate the CI/CD pipeline. Key features:
- **Build**: Compiles the Next.js application.
- **Dockerize**: Creates a Docker image for the application.
- **Deploy**: Uses Helm to deploy the Docker image to the Kubernetes cluster.

You can find the workflows in the `.github/workflows/` directory.

### Example Workflow
```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main    
  workflow_dispatch:
    inputs:
      env:
          type: choice
          description: Select Environment
          required: true
          options:
            - main

permissions:
  actions: read
  contents: write
  security-events: write

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Set up Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18.x'

    - name: Install dependencies
      run: npm ci

    - name: Build Next.js app
      run: npm run build
      env:
        NEXT_PUBLIC_SUPABASE_URL: ${{ secrets.NEXT_PUBLIC_SUPABASE_URL }}
        NEXT_PUBLIC_SUPABASE_ANON_KEY: ${{ secrets.NEXT_PUBLIC_SUPABASE_ANON_KEY }}

    - name: Log in to Docker Hub
      run: echo "${{ secrets.DOCKER_TOKEN }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin

    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: .
        file: ./Dockerfile
        push: true
        tags: ${{ secrets.DOCKER_USERNAME }}/ticket-shark:latest
        build-args: |
          NEXT_PUBLIC_SUPABASE_URL=${{ secrets.NEXT_PUBLIC_SUPABASE_URL }}
          NEXT_PUBLIC_SUPABASE_ANON_KEY=${{ secrets.NEXT_PUBLIC_SUPABASE_ANON_KEY }}

  deploy_to_kubernetes:
    needs: build
    runs-on: ubuntu-latest
    
    steps:
    - name: Install doctl
      uses: digitalocean/action-doctl@v2
      with:
        token: ${{ secrets.DO_ACCOUNT_TOKEN }}

    - name: Checkout code
      uses: actions/checkout@v3

    - name: Configure Digital Ocean Credentials
      run: |
        doctl kubernetes cluster kubeconfig save your-cluster-id

    - name: Deploy Helm on DOKS
      run: |
        helm upgrade --install my-app helm/ \
          --set image.repository=${{ secrets.DOCKER_USERNAME }}/app-app \
          --set image.tag=latest \
          --set nextPublicSupabaseURL=${{ secrets.NEXT_PUBLIC_SUPABASE_URL }} \
          --set nextPublicSupabaseAnonKey=${{ secrets.NEXT_PUBLIC_SUPABASE_ANON_KEY }} \
          --namespace interns \
          -f helm/environments/dev/values.yaml
```

## Helm Configuration
Helm is used to manage Kubernetes resources. The `values.yaml` file contains the environment-specific configuration, such as:
- Docker image and tag
- Ingress settings for routing traffic
- Resource requests and limits

### Install Helm Chart
```bash
helm install nextjs-app ./helm-chart -f values.yaml
```

## SSL/TLS with Let's Encrypt
SSL/TLS certificates are automatically obtained using [cert-manager](https://cert-manager.io/) and Let's Encrypt. This ensures that your application is securely accessible over HTTPS.

### Steps
1. Install `cert-manager`:
   ```bash
   kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.6.0/cert-manager.yaml
   ```

2. Configure your Helm chart to use Let's Encrypt for SSL:
   ```yaml
   ingress:
     enabled: true
     hosts:
       - your-domain.com
     tls:
       - secretName: tls-secret
         hosts:
           - your-domain.com
   ```

3. Update DNS records in Route53 to point to your DigitalOcean Kubernetes load balancer.

## Domain Management with Route53
- Set up a hosted zone in AWS Route53 for your domain.
- Create A records to point to the IP address of your DigitalOcean load balancer.
- Ensure that your Route53 settings are linked to the Ingress controller in Kubernetes.

## Running Locally
To run the application locally:
1. Clone the repository.
2. Install dependencies:
   ```bash
   npm install
   ```
3. Set up your `.env` file with Supabase credentials.
4. Run the application:
   ```bash
   npm run dev
   ```
5. Open `http://localhost:3000` to view the application locally.

    


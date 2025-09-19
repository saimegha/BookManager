# Step-by-step guide — CI/CD: Build .NET 9 Web API and deploy to Amazon ECR + ECS using GitHub Actions  


## Overview (what you’ll achieve)
- Build a .NET 9 Web API on GitHub Actions.  
- Containerize the app (using .NET containerization support).  
- Push Docker image to Amazon ECR.  
- Deploy the pushed image to Amazon ECS (Fargate) using a task definition.  

---

## Prerequisites
- .NET 9 SDK (local dev).  
- Docker Desktop (for local build/test of image).  
- AWS account with permissions for ECR and ECS.  
- GitHub repository (private or public).  

---

## High-level flow
1. Create a minimal .NET 9 Web API.  
2. Enable containerization via `.csproj`.  
3. Create an ECR repository and an IAM user for GitHub Actions.  
4. Store AWS credentials + ECR repo URI as GitHub secrets.  
5. Create GitHub Actions workflow (`.github/workflows/deploy.yml`) with `build`, `publish` and `deploy` jobs.  
6. Create ECS cluster/task/service manually once, then automate deployments.  

---

## Step 1 — Create the minimal .NET Web API
```bash
dotnet new webapi -n BookManager
cd BookManager
```

Minimal `Program.cs`:
```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
app.MapGet("/", () => "Hello World!");
app.Run();
```

---

## Step 2 — Add container metadata to the project (.csproj)
Inside `<PropertyGroup>` in `BookManager.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net9.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <ContainerImageTags>1.0.0;latest</ContainerImageTags>
    <ContainerRepository>bookmanager</ContainerRepository>
    <PublishProfile>DefaultContainer</PublishProfile>
  </PropertyGroup>
</Project>
```

---

## Step 3 — Local test: produce the container image
```bash
dotnet publish --os linux --arch x64
```

---

## Step 4 — Create ECR repository (AWS Console)
- Create an ECR repo and copy the **Repository URI**.  

---

## Step 5 — Create an IAM user for GitHub Actions
- Create IAM user and generate Access/Secret keys.  
- Attach `AmazonEC2ContainerRegistryFullAccess` + `AmazonECS_FullAccess` (demo).  

---

## Step 6 — Add repository secrets in GitHub
Secrets → Actions → Add:
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`
- `REPOSITORY` → `<account-id>.dkr.ecr.<region>.amazonaws.com/<repo>`

---

## Step 7 — Create GitHub Actions workflow (`.github/workflows/deploy.yml`)
```yaml
name: Deploy 🚀

on:
  workflow_dispatch:
  push:
    branches: [ main ]

env:
  AWS_REGION: ap-south-1
  AWS_ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.x'
      - run: dotnet restore ./BookManager/BookManager.csproj
      - run: dotnet build ./BookManager/BookManager.csproj --no-restore --configuration Release
      - run: dotnet test ./BookManager/BookManager.csproj --no-build --verbosity normal

  publish:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '9.x'
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      - name: Login to Amazon ECR
        run: |
          aws ecr get-login-password --region ${{ env.AWS_REGION }} | docker login --username AWS --password-stdin ${{ secrets.REPOSITORY%%/* }}
      - name: Build Docker Image
        working-directory: ./BookManager
        run: |
          IMAGE_URI=${{ secrets.REPOSITORY }}:latest
          dotnet publish -c Release -p:ContainerRepository=${{ secrets.REPOSITORY }} -p:RuntimeIdentifier=linux-x64
          echo "IMAGE_URI=$IMAGE_URI" >> $GITHUB_ENV
      - name: Push Docker Image to ECR
        run: docker push ${{ env.IMAGE_URI }}

  deploy:
    needs: publish
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      - uses: aws-actions/amazon-ecs-deploy-task-definition@v2
        with:
          task-definition: ./deployments/ecs-task-definition.json
          service: bookmanager
          cluster: bookmanager-cluster
          wait-for-service-stability: true
```

---

## Step 8 — Prepare ECS resources
- Create ECS cluster, task definition, and service manually once.  
- Commit task definition JSON to repo under `./deployments/ecs-task-definition.json`.  

---

## Step 9 — Run the pipeline & verify
- Push to `main`.  
- Verify image in ECR + ECS service updated.  

---

## Troubleshooting
- **400 error on `docker login`**: Check ECR URI and region.  
- **Credentials could not be loaded**: Ensure GitHub secrets are correct.  
- **dotnet publish container error**: Ensure Docker is running on runner.  

---

## Security recommendations
- Use least-privilege IAM roles.  
- Store credentials securely.  

---

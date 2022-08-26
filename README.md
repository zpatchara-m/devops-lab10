# pttgc-devops-lab10
This module will help attendees to develop and include Continue Delivery (CD) into their DevOps workflows (to update “Pttgc-devops-lab10-cd)

## Exercise 1 - Dev workflow

- Create new repository and copy everything in this repository to your repository
- Create another repository for IaC (infrastructure as a code) and copy everything from `https://github.com/zeabix-cloud-native/pttgc-devops-lab10-cd` repository to your CD repository

- Examine the file `values.yaml` in both directory `dev` and `production`, and change this value according to your assigned value
- Create these following Repository Secret according to the values provided in the spreadsheet

`ACR_LOGIN_SERVER`
`ACR_USERNAME`
`ACR_PASSWORD`
`CD_PAT` (Personal access token)

- Create `dev-workflows.yaml` in `.github/workflows`
```yaml
name: Dev Workflow

on:
  workflow_dispatch:
  push:
    branches:
      - main

jobs:

  build:
    runs-on: ubuntu-latest
    name: Build Docker 
    steps:
      - uses: actions/checkout@v3
      - run: docker build -t ${{ secrets.ACR_LOGIN_SERVER }}/user-service:${{ github.sha }} .
      - uses: azure/docker-login@v1
        with:
          login-server: ${{ secrets.ACR_LOGIN_SERVER }}
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}
      - run: docker push ${{ secrets.ACR_LOGIN_SERVER }}/user-service:${{ github.sha }}

  deploy:
    runs-on: ubuntu-latest
    name: Deploy via argoCD
    needs: build
    steps:
      - uses: actions/checkout@v3
        with:
          repository: "zeabix-cloud-native/pttgc-devops-lab10-cd"  ## Change this to your CD repository
          ref: 'main'
          token: ${{ secrets.CD_PAT }}
      - name: Update image version
        run: yq -i e '.userService.tag="${{ github.sha }}"' values.yaml
        working-directory: ./production
      - name: Commit & Push changes
        run: |
          git config --global user.email 'DevOps'
          git config --global user.name  'devops@zeabix.com'
          git add .
          git commit -m "CD deployment with tag ${{ github.sha }}"
          git push https://${{ secrets.CD_PAT }}@github.com/zeabix-cloud-native/pttgc-devops-lab10-cd.git

```

- Commit and push the change, go to GitHub UI to see how your workflow is running

## Exercise 2 - Release workflow

- Use the same repository, add another workflow file `production-deploy.yaml`
```yaml
name: Production Deploy Workflow

on:
  release:
    types:
      - published
    branches:
      - main
  workflow_dispatch:

jobs:

  retag:
    runs-on: ubuntu-latest
    name: Retag image for deploy
    steps:
      - uses: azure/docker-login@v1
        with:
          login-server: ${{ secrets.ACR_LOGIN_SERVER }}
          username: ${{ secrets.ACR_USERNAME }}
          password: ${{ secrets.ACR_PASSWORD }}
      - name: Get the version
        id: get_version
        run: echo ::set-output name=VERSION::$(echo $GITHUB_REF | cut -d / -f 3)
      - run: echo "Retag image from tag ${{ github.sha }} to ${{ steps.get_version.outputs.VERSION }}"
      - run: docker pull ${{ secrets.ACR_LOGIN_SERVER }}/user-service:${{ github.sha }}
      - run: docker tag ${{ secrets.ACR_LOGIN_SERVER }}/user-service:${{ github.sha }} ${{ secrets.ACR_LOGIN_SERVER }}/user-service:${{ steps.get_version.outputs.VERSION }}
      - run: docker push ${{ secrets.ACR_LOGIN_SERVER }}/user-service:${{ steps.get_version.outputs.VERSION }}

  deploy:
    runs-on: ubuntu-latest
    name: Deploy via argoCD
    needs: retag
    steps:
      - uses: actions/checkout@v3
        with:
          repository: "zeabix-cloud-native/pttgc-devops-lab10-cd"
          ref: 'main'
          token: ${{ secrets.CD_PAT }}
      - name: Get the version
        id: get_version
        run: echo ::set-output name=VERSION::$(echo $GITHUB_REF | cut -d / -f 3)
      - run: echo "Retag image from tag ${{ github.sha }} to ${{ steps.get_version.outputs.VERSION }}"
      - name: Update image version
        run: yq -i e '.userService.tag="${{ steps.get_version.outputs.VERSION }}"' values.yaml
        working-directory: ./production
      - name: Commit & Push changes
        run: |
          git config --global user.email 'DevOps'
          git config --global user.name  'devops@zeabix.com'
          git add .
          git commit -m "CD deployment to production with tag ${{ github.ref }}"
          git push https://${{ secrets.CD_PAT }}@github.com/zeabix-cloud-native/pttgc-devops-lab10-cd.git

```

- Commit and push the change

- Create the new release, assign tag number, then check the GitHub Actions to see how the workflow is running
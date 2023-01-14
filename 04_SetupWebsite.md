# 1. Introduction to Application Workflow

You should now have Completed the Following things:
1. Setup your GitHub Account
2. Setup your Git Repository
3. Setup Infrastructure Workflow

Next you will deploy the code to your web application. We already created **the code and the workflow** for you. The major task left for you is to run the application workflow.

# 2. Setting up the Application Workflow

## Run the Application Workflow

Running the workflow is done the same way as in the infrastructure case.

As stated you don't have to code something anymore. However, our major point is know-how transfer and not just typing. Therefore below a few comments what the YAML file`.github/workflows/azure_webapp.yml` is actually doing.

```
# File: .github/workflows/workflow.yml

on: 
  workflow_dispatch:

name: app

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
    # checkout the repo
    - name: 'Checkout Github Action' 
      uses: actions/checkout@master
      
    - name: Setup Node 10.x
      uses: actions/setup-node@v1
      with:
        node-version: '10.x'

    - name: 'npm install, build, and test'
      run: |
        cd app
        npm install
        npm run build --if-present
        npm run test --if-present
    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: ${{ secrets.AZURE_CREDENTIALS }}

    - name: 'Azure webapp deploy'
      uses: azure/webapps-deploy@v2
      with:
        app-name: ${{ secrets.WEBAPP }}
        package:  ${{ github.workspace }}/app
```

The file can be broken down into three major activities:
- **Installing node environment** including node itself and additional tooling like the node package manager. This step creates the required zip file that is needed for deployment later on.
- **Login to Azure** as you know it from the infrastructure workflow.
- **Deployment to webapp** based on the name you specified in the settings. `package` refers to the zip file you created previously.

## Workflow Progress

Wait for your Workflow to finish.
If the Task does not run through you may ask one of us to Help you out.
## Check your WebApp is online after approx. 2 minutes

https://`[yourWebAppName]`.azurewebsites.net/

You should see a &quot;WELCOME TO MICROSOFT CLOUD GUILD&quot; welcome screen.

Congratulations, you have deployed your first WebApp infrastructure.
 Now, you can go ahead and deploy some code to your WebApp.

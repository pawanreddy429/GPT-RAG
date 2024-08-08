# Multi-Environment GitHub Setup

This document outlines the steps to set up a multi-environment workflow to deploy infrastructure and services to Azure using GitHub Actions, taking the solution from proof of concept to production-ready.

# Assumptions:

- This example assumes you're using a GitHub organization with GitHub environments
- This example deploys the infrastructure in the same pipeline as all of the services.
- This example deploys three environments: dev, test, and prod. You may modify the number and names of environments as needed.
- This example uses [`azd pipeline config`](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/configure-devops-pipeline?tabs=azdo) to rapidly set up GitHub workflows and federated identity configuration for enhanced security.
- All below commands are run as a one-time setup on a local machine by an admin who has access to the GitHub Repository and Azure tenant.
- This example does not cover configuring any naming conventions.
- The original remote versions of the [orchestrator](https://github.com/Azure/gpt-rag-orchestrator), [frontend](https://github.com/Azure/gpt-rag-frontend), and [ingestion](https://github.com/Azure/gpt-rag-ingestion) repositories are used; in a real scenario, you would fork these repositories and use your forked versions. This would require updating the repository URLs in the `scripts/fetchComponents.*` files.
- This example uses federated identity for authentication. If you prefer to use client secret authentication, you will need to modify the workflow files accordingly.
- Bicep is the IaC language used in this example.

# Decisions required:

- Service Principals that will be used for each environment
- Whether to use federated identity or client secret for authentication
- Decisions on which GitHub repository, Azure subscription, and Azure location to use

# Prerequisites:

- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-windows?tabs=azure-cli)
- [Azure Developer CLI](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/install-azd?tabs=winget-windows%2Cbrew-mac%2Cscript-linux&pivots=os-windows)
- [GitHub CLI](https://cli.github.com/)
- [PowerShell 7](https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell?view=powershell-7.4)
- [Git](https://git-scm.com/downloads)
- Bash shell (e.g., Git Bash)
- GitHub organization with ability to provision environments (e.g., GitHub Enterprise)
- Personnel with Azure admin (can create Service Principals) and GitHub admin (owns repository/organization) access
- The code in the repository needs to exist in Azure Repos and you need to have it cloned locally. [This guide](https://github.com/Azure/azure-dev/blob/main/cli/azd/docs/manual-pipeline-config.md) may be useful if you run into issues setting up your repository.

# Steps:

> [!NOTE]
> All commands below are to be run in a Bash shell.

## 1. Create azd environments & Service Principals

### Setup

`cd` to the root of the repo. Before creating environments, you need to define the environment names. Note that these environment names are reused as the GitHub environment names later.

```bash
dev_env='<dev-env-name>' # Example: dev
test_env='<test-env-name>' # Example: test
prod_env='<prod-env-name>' # Example: prod
```

Next, define the names of the Service Principals that will be used for each environment. You will need the name in later steps.

```bash
dev_principal_name='<dev-sp-name>'
test_principal_name='<test-sp-name>'
prod_principal_name='<prod-sp-name>'
```

### `azd` environments

Next, you will create an `azd` environment per target environment alongside a pipeline definition. In this guide, pipeline definitions are created with `azd pipeline config`. Read more about azd pipeline config [here](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/configure-devops-pipeline?tabs=azdo). View the CLI documentation [here](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/reference#azd-pipeline-config).

For each below environment, when running `azd pipeline config` for each environment, choose **GitHub** as the provider, choose your target Azure subscription, and Azure location. When prompted to commit and push your local changes to start the configured CI pipeline, say 'N'.

Login to Azure:

```bash
az login
```

#### Dev

```bash
azd env new $dev_env
azd pipeline config --auth-type federated --principal-name $dev_principal_name
```

#### Test

```bash
azd env new $test_env
azd pipeline config --auth-type federated --principal-name $test_principal_name
```

#### Prod

```bash
azd env new $prod_env
azd pipeline config --auth-type federated --principal-name $prod_principal_name
```

> [!NOTE]
> Note that `azd pipeline config` creates a new Service Principal for each environment.

> [!NOTE]
> By default, `azd pipeline config` uses OpenID Connect (OIDC), called federated credentials. If you'd rather not use OIDC, run `azd pipeline config --auth-type client-credentials`. This scenario is not covered in this guide.

After performing the above steps, you will see corresponding files to your azd environments in the `.azure` folder.

If you run `azd env list`, you will see the newly created environments.

You may change the default environment by running `azd env select <env-name>`, for example:

```bash
azd env select $dev_env
```

## 2. Set up GitHub Environments

### Environment setup

Set up initial variables:

```bash
org='<your-org-name>'
repo='<your-repo-name>'
```

Run GitHub CLI commands to create the environments:

```bash
gh auth login

gh api --method PUT -H "Accept: application/vnd.github+json" repos/$org/$repo/environments/$dev_env
gh api --method PUT -H "Accept: application/vnd.github+json" repos/$org/$repo/environments/$test_env
gh api --method PUT -H "Accept: application/vnd.github+json" repos/$org/$repo/environments/$prod_env
```

### Variables setup

Configure the repository and environment variables: Delete the `AZURE_CLIENT_ID` and `AZURE_ENV_NAME` variables at the repository level as they aren't needed and only represent what was set for the environment you created last. `AZURE_CLIENT_ID` will be reconfigured at the environment level, and `AZURE_ENV_NAME` will be passed as an input to the deploy job.

```bash
gh variable delete AZURE_CLIENT_ID
gh variable delete AZURE_ENV_NAME
```

Get the client IDs of the Service Principals you created. Ensure you previously ran `az login`.

```bash
dev_client_id=$(az ad sp list --display-name $dev_principal_name --query "[].appId" --output tsv)
test_client_id=$(az ad sp list --display-name $test_principal_name --query "[].appId" --output tsv)
prod_client_id=$(az ad sp list --display-name $prod_principal_name --query "[].appId" --output tsv)
```

> [!NOTE] 
> _Alternative approach to get the client IDs in the above steps:_
> In the event that there are multiple Service Principals containing the same name, the `az ad sp list` command executed above may not pull the correct ID. You may execute an alternate command to manually review the list of Service Principals by name and ID. The command to do this is exemplified below for the dev environment.
>
> ```bash
> az ad sp list --display-name $dev_principal_name --query "[].{DisplayName:displayName, > AppId:appId}" --output table # return results in a table format
> dev_client_id='<guid>' # manually assign the correct client ID
> ```
>
> Also note you may also get the client IDs from the Azure Portal.

Set these values as variables at the environment level:

```bash
gh variable set AZURE_CLIENT_ID -b $dev_client_id -e $dev_env
gh variable set AZURE_CLIENT_ID -b $test_client_id -e $test_env
gh variable set AZURE_CLIENT_ID -b $prod_client_id -e $prod_env
```

> [!TIP]
> After environments are created, consider setting up deployment protection rules for each environment. See [this article](https://docs.github.com/en/actions/administering-github-actions/managing-environments-for-deployment#deployment-protection-rules) for more.

> [!NOTE]
> If you want to manage and authenticate with a client secret rather than using federated identity, you would need to create a secret for each Service Principal, store it as an environment secret in GitHub, and modify the workflow to use the secret for authentication. This is not covered in this example. If you choose to use a client secret, you may skip 3.

## 3. Configure Azure Federated credentials to use newly set up GitHub environments

Run the following commands to create a federated credential within the Service Principals you created. These commands will enable authentication between GitHub and Azure for each environment.

```bash
issuer="https://token.actions.githubusercontent.com"
audiences="api://AzureADTokenExchange"

echo '{"name": "'"${org}-${repo}-${dev_env}"'", "issuer": "'"${issuer}"'", "subject": "repo:'"$org"'/'"$repo"':environment:'"$dev_env"'", "description": "'"${dev_env}"' environment", "audiences": ["'"${audiences}"'"]}' > federated_id.json
az ad app federated-credential create --id $dev_client_id --parameters ./federated_id.json

echo '{"name": "'"${org}-${repo}-${test_env}"'", "issuer": "'"${issuer}"'", "subject": "repo:'"$org"'/'"$repo"':environment:'"$test_env"'", "description": "'"${test_env}"' environment", "audiences": ["'"${audiences}"'"]}' > federated_id.json
az ad app federated-credential create --id $test_client_id --parameters ./federated_id.json

echo '{"name": "'"${org}-${repo}-${prod_env}"'", "issuer": "'"${issuer}"'", "subject": "repo:'"$org"'/'"$repo"':environment:'"$prod_env"'", "description": "'"${prod_env}"' environment", "audiences": ["'"${audiences}"'"]}' > federated_id.json
az ad app federated-credential create --id $prod_client_id --parameters ./federated_id.json

rm federated_id.json # clean up temp file
```

> [!NOTE]
> The existing/unmodified federated credentials created by Azure Developer CLI in the Service Principals may be deleted.

## 4. Modify the workflow files as needed for deployment

- The following files in the `.github/workflows` folder are used to deploy the infrastructure and services to Azure:
  - `azure-dev.yml`
    - This is the main file that triggers the deployment workflow. The environment names are passed as inputs to the deploy job, **which needs to be edited to match the environment names you created**.
    - You may edit the workflow_dispatch to suit your workflow trigger needs.
  - `deploy-template.yml`
    - This is a template file invoked by `azure-dev.yml` that is used to deploy the infrastructure and services to Azure. This file needs to be edited if you are using client secret authentication.

# Additional Resources:

- [Support multiple environments with `azd` (github.com)](https://github.com/jasontaylordev/todo-aspnetcore-csharp-sqlite/blob/main/OPTIONAL_FEATURES.md)

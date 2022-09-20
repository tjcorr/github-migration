# Migrating to GitHub from Azure DevOps

## Prerequistes 

### Required Access

To perform migrations you will need to have read permissions on the source projects/organizations in ADO. On the GitHub side you will either need to be an organization owner or be granted the migrator role. `TODO` add link. 

To utilize Valet during the private beta you will need to request access to the valet-customers organization. GitHub staff can see [instructions]( https://github.com/github/valet-customers#onboard-a-new-customer) to onboard new users.

### Install GitHub CLI and Extensions

- Install the GitHub CLI following the [instructions](https://github.com/cli/cli#installation) for your operating system.
- Install the following GitHub CLI extensions:
  - **GEI** - `gh extension install github/gh-gei` 
  - **ADO2GH** - `gh extension install github/gh-ado2gh`
  - **Valet** - `gh extension install github/gh-valet` 
  
  _Note: Valet requires Docker_
   

## Setup credentials on both platforms

- Create an ADO personal access token (PAT). See [instructions](https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate?view=azure-devops&tabs=Windows#create-a-pat). Necessary permission vary for different steps:
  - Most operations require basic access: `work item (read)`, `code (read)`, and `identity (read)` scopes.
  - Additional functionality in ADO2GH also requires: `code (read/write/manage)`, `security (manage)`, `build (read)`, and `service connection (read)` 
  - Full control is needed for performing the inventory report and to integrate boards.
  
  Additional details can be found [here](https://docs.github.com/en/early-access/enterprise-importer/preparing-to-migrate-with-github-enterprise-importer/managing-access-for-github-enterprise-importer#personal-access-tokens-for-azure-devops).

- Create a GitHub PAT. See [instructions](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token). The token will need `repo`, `admin:org`, and `workflow` permissions. See [here](https://docs.github.com/en/early-access/enterprise-importer/preparing-to-migrate-with-github-enterprise-importer/managing-access-for-github-enterprise-importer#personal-access-tokens-for-github-products) for details.

- Store the credentials as environment variables in your shell:
  - `export GH_PAT=.....`
  - `export ADO_PAT=....`

## Run a migration assessment

`gh ado2gh inventory-report --ado-org <org-name>`

Observe the following files:

- `orgs.csv`: name,url,owner,teamproject-count,repo-count,pipeline-count,is-pat-org-admin,pr-count
- `pipelinse.csv`: org,teamproject,repo,pipeline,url
- `repos.csv`: org,teamproject,repo,url,last-push-date,pipeline-count,compressed-repo-size-in-bytes,most-active-contributor,pr-count,commits-past-year
- `team-projects.csv`: org,teamproject,url,repo-count,pipeline-count,pr-count

## Perform a migration of a single repository

Using gei:

`gh gei migrate-repo --ado-source-org <ado-org> --ado-team-project <ado-project> --source-repo <ado-repo> --github-target-org <github-org> --target-repo <github-repo> --wait`

Using ado2gh (prefered):

`gh ado2gh migrate-repo --ado-org <ado-org> --ado-team-project <ado-project> --ado-repo <ado-repo> --github-org <github-org> --github-repo <github-repo> --wait`

Either should migrate the same data including commits, branches, and tags along with all PR data. A full list of items migrated can be found [here](https://docs.github.com/en/early-access/enterprise-importer/understanding-github-enterprise-importer/migration-support-for-github-enterprise-importer#azure-devops-migration-support).

## Reclaiming mannequins

Normally the migration tool will automatically map users between ADO and GH by utilizing email addresses. However, if you have users in ADO that don't currently match an existing GitHub user the migration tool will create placeholder or mannequin users in GitHub. This allows us to properly preserve the history of the repo. A separate step can be performed adter all migrations are complete to identify and map those mannequins to proper users. This can be performed either: 

- Through the UI

  under org settings -> access -> import/export -> mannequins https://github.com/organizations/<org-name>/settings/import-export
  
- Through the CLI

  `gh ado2gh generate-mannequin-csv --github-org <org-name> --output mannequin.csv`

  Add user mappings in the csv file.

  `gh ado2gh reclaim-mannequin --github-org <org-name> --csv mannequin.csv` 

## Rewire Azure Pipelines to point at GitHub

Now that the code lives in GitHub, ADO pipelines need to be updated to reference the new code location.

These steps need to be performed once per GitHub organization:
1. Install the Azure Pipelines app from the GitHub marketplace at [link](https://github.com/marketplace/azure-pipelines)
2. Configured the GitHub App to have access to the necessary repositories.
3. Create a service connection in ADO to GitHub. 

Then for each pipeline:
4. `gh ado2gh rewire-pipeline --ado-org <ado-org> --ado-team-project <ado-project> --ado-pipeline <ado-pipeline-name> --github-org <github-org> --github-repo <github-repo> --service-connection-id <service-connection-id>`

This is applicable to classic and YAML based pipelines. Release pipelines will need to be adjusted manually depending upon their trigger type.

## Link Azure Boards to GitHub

You can setup automatic linking of GitHub commits/PRs to Azure Boards work items:

`gh ado2gh integrate-boards --ado-org <ado-org> --ado-team-project <ado-project> --github-org <github-org> --github-repo <github-repo>`

Note this currently utilizes a PAT that can expire and break the integration. The prefered approach is to utilize a GitHub App but that is not yet supported through the CLI. Instead I would recommend configuring the integration via the UI (see [instructions](https://learn.microsoft.com/en-us/azure/devops/boards/github/install-github-app?view=azure-devops))x

## Other ado2gh functionality

The `ado2gh` tool supports a variety of other functions to facilitate the migration process including:

- `lock-ado-repo`: This locks the repo in ADO to prevent anyone from pushing new commits. 

Full command: `gh ado2gh lock-ado-repo --ado-org <ado-org> --ado-team-project <ado-project> --ado-repo <ado-repo>`

- `disable-ado-repo`: This completely disables the ADO repo removing visibility. Note: No data is deleted here just archived.

Full command: `gh ado2gh disable-ado-repo --ado-org <ado-org> --ado-team-project <ado-project> --ado-repo <ado-repo>` 	

- `configure-autolink`: This sets up an autolinnking pattern in GitHub to make it easy to reference ADO work items.

Full command: `gh ado2gh configure-autolink --github-org <github-org> --github-repo <github-repo> --ado-org <ado-org> --ado-team-project <ado-project>

Others tools are aldo available and can be discovered by running `gh ado2gh --help`


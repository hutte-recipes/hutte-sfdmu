# Hutte Recipe - SFDMU

> This recipe uses [SFDX Data Move Utility (SFDMU)](https://github.com/forcedotcom/SFDX-Data-Move-Utility) to import and export data using a Custom Button (with input parameters) in Hutte.

<img src="./docs/images/import-data-button.png" alt="drawing" width="700"/>
<img src="./docs/images/commit-data-button.png" alt="drawing" width="700"/>

## Prerequisites

- Salesforce CLI installed on your computer
- a valid sfdx project
- a `hutte.yml` file (e.g. the default one shown in the `CONFIGURATION` tab)
- a source org authenticated with Salesforce CLI locally from which you want to export data

The following steps assume that we use the `data` directory to store `CSV` files.

## Step 1: Install SFDMU on your machine

Run the following code from your console (terminal):

```console
echo y | sf plugins install sfdmu
sf sfdmu --help
```

## Step 2: Prepare your data import

Prepare the `data/<datafolder>/data-import.json` file per each of the data folders. For example, in this recipe we have three different data folders, namely `qa-data`, `package-configuration-data` and `development-baseline-data`. The `data-import.json` file will specify which data will be loaded when running the `Import Data` custom button, for the specific folder specified as parameter.

Example:

```json
{
	"objects": [
		{
			"query": "SELECT Id, Name FROM Account",
			"operation": "Upsert",
			"externalId": "Name"
		}
	]
}
```

Within the `data` folder, create a `.gitignore` file with the content:

```text
# Internal .gitignore to avoid tracking SFDMU execution files that are not required for the Data package.

*.log
*target
MissingParentRecordsReport.csv

# We don't commit the final export.json given that this is created on runtime by the Hutte custom button logic
export.json
```

## Step 3: Add custom buttons

- Edit the `hutte.yml` file in your default branch
- Add the following content within `custom_scripts` section.

```yaml
custom_scripts:
  # This scripts will be displayed on the scratch org's page
  scratch_org: &custom_scripts
    "Commit Data":
      description: "Commit data into Git using SFDMU"
      run: |
        set -euo pipefail # fail fast
        envsubst < data/template-export.json > "${HUTTE_INPUT_destinationfolder}/export.json"
        cat "${HUTTE_INPUT_destinationfolder}/export.json"
        echo y | sf plugins install sfdmu
        sf sfdmu run --path "${HUTTE_INPUT_destinationfolder}" --sourceusername "${SALESFORCE_USERNAME}" --targetusername csvfile --filelog 0 -n
        git add . && git commit -m "${HUTTE_INPUT_commitmessage}" && git push origin "${HUTTE_GIT_SOURCE_BRANCH}"
      inputs:
        soql:
          label: "Data SOQL"
          description: "Specify the SOQL that returns the data to commit, including the more relevant fields and filters."
          type: string
          default: "SELECT Id FROM Account LIMIT 10"
          required: true
        destinationfolder:
          label: "Destination Folder"
          description: "Choose the repository folder where you'd like to commit the data records."
          type: choice
          default: "data/qa-data"
          required: true
          options:
            - data/development-baseline-data
            - data/qa-data
            - data/package-configuration-data
        commitmessage:
          label: "Git Commit Message"
          type: string
          required: true
    "Import Data":
      description: "Import data using SFDMU"
      run: |
        set -euo pipefail # fail fast
        cp "${HUTTE_INPUT_sourcefolder}/data-import.json" "${HUTTE_INPUT_sourcefolder}/export.json"
        echo y | sf plugins install sfdmu
        sf sfdmu run --path "${HUTTE_INPUT_sourcefolder}" -s csvfile -u "${SALESFORCE_USERNAME}" --filelog 0 -n
      inputs:
        sourcefolder:
          label: "Source Folder"
          description: "Choose the repository folder to load data from."
          type: choice
          default: "data/qa-data"
          required: true
          options:
            - data/development-baseline-data
            - data/qa-data
            - data/package-configuration-data
  sandbox: *custom_scripts
```

Note that these custom buttons will be available for both Scratch Orgs and Sandboxes.

## Step 4: Validate

1. Create a Scratch Org or open an existing Scratch Org in Hutte.
2. Verify that the buttons are displayed.
3. Execute the buttons and verify that data was committed/imported.

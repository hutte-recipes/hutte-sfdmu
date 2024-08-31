# Hutte Recipe - SFDMU

> This recipe uses [SFDX Data Move Utility (SFDMU)](https://github.com/forcedotcom/SFDX-Data-Move-Utility) to import and export data using a Custom Button (with input parameters) in Hutte.

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

Prepare the `data/<datafolder>/data-import.json` file per each of the data folders. For example, in this recipe we have three different data folders, namely `qa-data`, `package-configuration-data` and `development-baseline-data`. The `data-import.json` file will specify which data will be loaded when running the `Data Import` custom button, for the folder specified in the parameter.

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
        set -e
        cp data/template-export.json "$HUTTE_INPUT_destinationfolder/export.json"
        # If it contains "LIMIT", use the query as is, otherwise add LIMIT 10 to avoid accidental commit of several data records
        if [[ "$HUTTE_INPUT_soql" =~ [Ll][Ii][Mm][Ii][Tt] ]]; then
            soql="$HUTTE_INPUT_soql"
        else
            soql="$HUTTE_INPUT_soql LIMIT 10"
        fi
        escaped_soql=$(printf '%s\n' "$soql" | sed 's/[&/\]/\\&/g')
        sed -i "s|<insertQuery>|$escaped_soql|g" "$HUTTE_INPUT_destinationfolder/export.json"
        cat "$HUTTE_INPUT_destinationfolder/export.json"
        cd $HUTTE_INPUT_destinationfolder
        echo y | sf plugins install sfdmu
        sf sfdmu run --sourceusername "${SALESFORCE_USERNAME}" --targetusername csvfile --filelog 0 -n
        git add . && git commit -m "$HUTTE_INPUT_commitmessage" && git push origin "${HUTTE_GIT_SOURCE_BRANCH}"
      inputs:
        soql:
          label: "Data SOQL"
          description: "Specify the SOQL that returns the data to commit, including the more relevant fields and filters. Note: If you don't include a LIMIT, default 'LIMIT 10' will be used as security measure."
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
        cp $HUTTE_INPUT_sourcefolder/data-import.json $HUTTE_INPUT_sourcefolder/export.json
        echo y | sf plugins install sfdmu
        sf sfdmu run -p $HUTTE_INPUT_sourcefolder -s csvfile -u "${SALESFORCE_USERNAME}" --filelog 0 -n
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

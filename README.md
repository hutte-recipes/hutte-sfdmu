# Hutte Recipe - SFDMU

> This recipe uses [SFDX Data Move Utility (SFDMU)](https://github.com/forcedotcom/SFDX-Data-Move-Utility) to import and export data using a Custom Button in Hutte.

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

## Step 2: Prepare your data export

Prepare the `data/export.json` file:

```json
{
  "objects": [
    {
      "query": "SELECT readonly_false FROM Account ORDER BY Name ASC",
      "operation": "Upsert",
      "externalId": "Name"
    },
    {
      "query": "SELECT readonly_false FROM Contact ORDER BY Email,LastName,FirstName ASC",
      "operation": "Upsert",
      "externalId": "Email;LastName;FirstName"
    }
  ],
  "excludeIdsFromCSVFiles": true
}
```

Add the following lines to the `.gitignore` file:

```text
# SFDMU
/data/source/
/data/target/
/data/logs/
```

Export the data from an org to the Git repository:

```console
sf sfdmu run -s "<THE_TARGET_ORG_ALIAS>" -u csvfile --filelog 0 -n
git add data
git commit -m "add Salesforce data"
git push
```

## Step 3: Add custom button

- Edit the `hutte.yml` file in your default branch
- Add the following button in `custom_scripts > scratch_org`

```yaml
custom_scripts:
  scratch_org:
    "Import Data":
      description: "Import data using SFDMU"
      run: |
        echo y | sf plugins install sfdmu
        sf sfdmu run -p data -s csvfile -u "${SALESFORCE_USERNAME}" --filelog 0 -n
    'Export Data':
      description: "Export data using SFDMU"
      run: |
        echo y | sf plugins install sfdmu
        sf sfdmu run -p data -s "${SALESFORCE_USERNAME}" -u csvfile --filelog 0 -n
        git add data
        git commit -m "add data"
        git push
```

## Step 4: Validate

- Create a Scratch Org or open an existing Scratch Org in Hutte
- Verify that the button is displayed
- Execute the button and verify that data was imported

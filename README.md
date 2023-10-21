# Hutte Recipe - SFDMU

> This recipe uses [SFDX Data Move Utility (SFDMU)](https://github.com/forcedotcom/SFDX-Data-Move-Utility) to import data using a Custom Button in Hutte.

## Prerequisites

- a valid Sfdx Project
- a `hutte.yml` file (e.g. the default one shown in the `CONFIGURATION` tab)
- a source org authenticated with Salesforce CLI locally from which you want to export data

## Steps

The following assumes that we use the `data` directory to store `CSV` files.

### Step 1

Install `SFDMU` on your machine.

```console
echo y | sf plugins install sfdmu
sf sfdmu --help
```

### Step 2

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

### Step 3

- Edit the `hutte.yml` file in your default branch
- Add the following button in `custom_scripts > scratch_org`

```yaml
custom_scripts:
  scratch_org:
    'Import Data':
      description: "Import data using SFDMU"
      run: |
        echo y | sf plugins install sfdmu
        sf sfdmu run -p data -s csvfile -u "${SALESFORCE_USERNAME}" --filelog 0 -n
```

### Step 4

- Create a Scratch Org or open an existing Scratch Org
- Verify that the button is displayed

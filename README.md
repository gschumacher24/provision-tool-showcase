## Purpose
The purpose of this project is to showcase the phData Provision Tool and to provide instructions for initial setup in GitHub    .

---

## Pre-requisites
- phData Toolkit CLI installed on your local machine. Follow the instructions [here](https://toolkit.phdata.io/docs/toolkit-cli#installation).
  - Dependency: Java LTS versions 11, 17, or 21 must be installed on your machine to run the CLI.
- A Snowflake account with the necessary permissions to create databases, schemas, and warehouses TODO.
- A GitHub repository TODO
- OpenSSL TODO

---

## Create Local Toolkit Project
In order to run Provision Tool commands locally, you need to create a local Toolkit project. This can be done by running the [toolkit init command](https://toolkit.phdata.io/docs/toolkit-cli#init-command) in your terminal, in the directory you wish to create the project:

`toolkit init`

That should create the following files:
- .gitignore
- .toolkit-metadata
- toolkit.conf

The toolkit.conf file is where you will configure the Provision Tool settings. You can find more information about the configuration options [here](https://toolkit.phdata.io/docs/provision#configuration). Those instructions will be recorded here for convenience.

---

## Snowflake Environment Setup
The Provision Tool documentation has instructions for the [snowflake environment setup](https://toolkit.phdata.io/docs/provision#snowflake-environment-setup).

### Create Dedicated Resources
The Provision Tool requires a dedicated Snowflake environment with the following components:
- Warehouse
- Roles
- User
- Resource monitor

You can generate the SQL necessary to create these components by running the [Provision Tool init command](https://toolkit.phdata.io/docs/provision#init-command) in your terminal from within your Toolkit project: `toolkit provision init --generate-sql`

Execute the generated SQL in your Snowflake account to create the required components.

### Setup Key-Pair Authentication
Key-pair authentication is needed to have the Provision Tool authenticate to Snowflake without using a password.  Snowflake has instructions for [how to configure key-pair authentication](https://docs.snowflake.com/en/user-guide/key-pair-auth#configuring-key-pair-authentication):

Generate a private key (without encryption) by running the following command in your terminal: 
`openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out rsa_key.p8 -nocrypt`

Generate a public key, based on the private key you just created, by running the following command in your terminal: 
`openssl rsa -in rsa_key.p8 -pubout -out rsa_key.pub`

While you're here, you'll need to encode the private key in base64 format to use in GitHub repo. To do this, run the following command in your terminal: 
`base64 -i rsa_key.p8 `

Keep the generated value handy for later.

To grant the privilege to assign a public key to the Provision Tool user, run the following SQL command in your Snowflake account: 
`GRANT MODIFY PROGRAMMATIC AUTHENTICATION METHODS ON USER PROVISION_USER TO ROLE PROVISION_ADMIN;`

To assign the public key to the Provision Tool user, run the following SQL command in your Snowflake account, replacing "[PUBLIC KEY HERE]" with the contents of the rsa_key.pub file you generated earlier. Do not include the header nor footer lines in this SQL command, but do include the line breaks: 
`ALTER USER PROVISION_USER SET RSA_PUBLIC_KEY='[PUBLIC KEY HERE]';`

To verify that the public key was assigned correctly, run the following SQL command in your Snowflake account, recording the output:
```
DESC USER PROVISION_USER
  ->> SELECT SUBSTR(
        (SELECT "value" FROM $1
           WHERE "property" = 'RSA_PUBLIC_KEY_FP'),
        LEN('SHA256:') + 1) AS key;
```

The value from the previous Snowflake SQL command should match the output of running the following command in your terminal: 
`openssl rsa -pubin -in rsa_key.pub -outform DER | openssl dgst -sha256 -binary | openssl enc -base64`

### Configure JDBC Connection Info
The toolkit.conf file created earlier needs to be updated with the connection information for your Snowflake account. Update the `provision` section of the file to look like the following:
```
provision {
  connection {
    url = "jdbc:snowflake://[SNOWFLAKE ACCOUNT IDENTIFIER HERE].snowflakecomputing.com"
    properties {
      user = PROVISION_USER
      role = PROVISION_ADMIN
      private_key_file = rsa_key.p8
    }
  }
}
```

You can find additional connection options in the [Provision Tool connection documentation](https://toolkit.phdata.io/docs/provision#connection).

Snowflake has documentation on [account identifiers](https://docs.snowflake.com/en/user-guide/admin-account-identifier) and where to find them. One option is opening the account info in the Snowsight UI.  Alternatively, this SQL command can also be used to get the account identifier: 
`SELECT CURRENT_ORGANIZATION_NAME() || '-' || CURRENT_ACCOUNT_NAME() AS ACCOUNT_IDENTIFIER;`

### Setup Metadata Database
The tool needs its own database for caching and concurrency control. You can create the tool metadata in Snowflake by executing the [Provision Tool init command](https://toolkit.phdata.io/docs/provision#init-command) in your teminal from within your Tookit project: 
`toolkit provision init --metadata`

After running that command, you should see a new database in your Snowflake account named "PHDATA".

---

## GitHub Repo Setup
Now that your Snowflake is setup with everything the Provison Tool needs, you can setup your repo to use the Provision Tool with GitHub actions.

### Repo Secrets & Variables
GitHub has documentation on [using secrets in GitHub actions](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets), which includes instructions on how to add secrets and variables for a repo. You'll need to add the following variables and secrets to the repo:
- Repository variables:
  - TOOLKIT_CLI_VERSION: Whichever version of the Toolkit CLI you want to use. Check the [Toolkit Updates page](https://toolkit.phdata.io/updates) for the latest version.
- Repository secrets:
  - SNOWFLAKE_URL: The full JDBC URL for your Snowflake account. It should match the URL you put in the toolkit.conf file earlier.
  - TOOLKIT_CLI_DOWNLOAD_TOKEN: On the [Toolkit Access page](https://toolkit.phdata.io/tool-access), there is a download url that can be used to download the Toolkit. The download token can be extracted from this url: `https://repo.phdata.io/[DOWNLOAD TOKEN]/toolkit-cli/maven/io/phdata/toolkit/toolkit-cli/[VERSION NUMBER]/toolkit-cli-[VERSION NUMBER].zip`
  - TOOLKIT_AUTH_TOKEN: An authentication token for the Toolkit can be generated on the [Toolkit Access page](https://toolkit.phdata.io/tool-access).
  - SNOWFLAKE_PRIVATE_KEY_PEM: The base64 encoded private key you generated earlier.

### Create PR Ruleset
GitHub has documentation on [creating a branch or tag ruleset](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/creating-rulesets-for-a-repository#creating-a-branch-or-tag-ruleset). You'll want to create a branch ruleset for your default branch that includes the following rules:
- Require status checks to pass
- Require branches to be up to date before merging

---
## GitHub Repo Troubleshooting
If you're running into any issues running the Provision Tool workflow, try double-checking the following.

### Provision Tool Workflow Name
If you're not seeing Provision Tool build show up in the list of GitHuba actions when setting up the ruleset, double-check the top level "[name](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#name)" field in the provision.yaml file. If there is no top level "name" field in a workflow .yaml, GitHub should display the workflow file path relative to the root of the repository.

### Toolkit toolkit.comf in Repo
In order to run Toolkit CLI commands, you need to be in a Toolkit project directory. This means that the "provision" directory needs to have a toolkit.conf file for the workflow to run properly.

### Project Name for Reporting
If the report related toolkit commands in the provision.yaml file are failing, double-check the project name in the snapshot identifier.  It should follow the following format: "[SNOWFLAKE ACCOUNT IDENTIFIER OR ORGANIZATION NAME]:[plan or apply]:latest".

### Double Check Default Branch
If the provision workflow isn't running when you expect it to, double-check that the branch set to trigger the workflow in the provision.yaml file matches your default branch name.

---
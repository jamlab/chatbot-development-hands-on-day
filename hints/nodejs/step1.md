# Hints for completing Step 1

## Setting up the Resource Group and Storage Account

Create a new Resource Group for our bot:

```
az group create --location "West Europe" --name chatbotday-nodejs
```

(*Optional*) Create a new Storage Account for holding the bot's state:
```
az storage account create --resource-group chatbotday-nodejs --sku Standard_LRS --name cbdbotstate

az storage table create --account-name cbdbotstate --name botstateprod

#Optional: use a separate table for local testing
az storage table create --account-name cbdbotstate --name botstatedev
```

Upon bot creation, Azure automatically creates a Storage Account for you, but given the random naming, it might be less confusing to create your own.

Get the connection string for storage account access:

```
az storage account show-connection-string --resource-group chatbotday-nodejs --name cbdbotstate
{
  "connectionString": "DefaultEndpointsProtocol=https;EndpointSuffix=core.windows.net;AccountName=xxxxx;AccountKey=xxxxxxxxxxxxx"
}
```

## Create Bot Service and setup deployment via git

Firstly, install the following two tools:

1. Install [Azure Storage Explorer](https://azure.microsoft.com/en-us/features/storage-explorer/)
1. Install [Bot Framework Emulator](https://github.com/Microsoft/BotFramework-Emulator/releases)

Next, you can create the Bot Service either through the Azure Portal or via Azure CLI.

### Create Bot Service via Azure Portal

1. Create Bot Service based on App Service (Web App Bot) with the `Basic Node.js` template
1. Configure local Git repo: Navigate to your bot --> `Build` --> `Configure Continuous Deployment` --> `Setup` --> `Local Git Repository`
1. Retrieve Git Clone URL: `All App Service Settings` --> `Overview` --> `Git Clone URL` (might need to set up password first)

### Create Bot Service via CLI

Make sure you have the latest version of the Azure CLI installed. In order to install the `botservice` extension, run:

```
az extension add -n botservice
az bot create --location westeurope --resource-group chatbotday-nodejs --kind webapp --name cbdhodcs --display-name "Chatbot Hands-On Day Bot" --lang Node --storage cbdbotstate
```

Let's check if the bot has been provisioned:

```
az bot show --resource-group chatbotday-nodejs --name cbdhodcs
```

Configure deployment via a `Local Git Repository`:

```
az webapp deployment source config-local-git --resource-group chatbotday-nodejs --name cbdhodcs
```

This will give us the Git Clone URL:

```json
{
  "url": "https://xxxxx@cbdhodcs.scm.azurewebsites.net/cbdhodcs.git"
}
```

## Clone and install packages (locally)

Clone repo locally and install Node packages:

```
git clone https://<user>@<botname>.scm.azurewebsites.net:443/<botname>.git
cd <botname>
npm install
```

You can now use [VS Code](https://code.visualstudio.com/) for easy development, debugging and committing to `git`:

```
code .
```

## Setup bot state table

Open `app.js` and adapt it to use our new storage account and table storage (for bot state):

```javascript
const STORAGE_CONNECTION = 'DefaultEndpointsProtocol=https;EndpointSuffix=core.windows.net;AccountName=xxxxx;AccountKey=xxxxxxxx';

var tableName = 'botdata';
var azureTableClient = new botbuilder_azure.AzureTableClient(tableName, STORAGE_CONNECTION);
var tableStorage = new botbuilder_azure.AzureBotStorage({ gzipData: false }, azureTableClient);
```

*Important:* See below on how to keep the secrets out of source code!

## Test bot locally

Start bot locally via command line:

```
node app.js
```

Alternative, in VS Code, goto the bug symbol on the left, hit the green play button, and select `Node.js` from the suggestion menu.

* Connect via Bot Framework Emulator to `http://localhost:3978/api/messages` and test if the bot is working
* Check table storage via Storage Explorer and see what happened

## Push changes to production 

Commit changes to `git`:

```
git add app.js
git commit -m "Added table storage"
git push azure
```

This will commit your code to the git repository in Azure and automatically trigger the deployment process. The easiest way to test if your bot works in Azure is by using the `Test in Web Chat` functionality.

## Add Bot to Microsoft Teams

1. Change name of bot: `Settings` --> `Change Display Name` --> `Your new bot's name` (don't forget to hit `Save`)
1. Add to channel: `Channels` --> `Teams` --> `Enable`
1. Test: Click `Teams` entry --> link opens --> Teams app opens --> Bot should be there and functional (might take a minute or two until the bot starts responding)

Or, if you prefer to use the Azure CLI, just execute:

```
az bot msteams create --resource-group chatbotday-nodejs --name cbdhodcs
```

## Proper setup of secrets

Setup proper env variables via `dotenv` library:

```
npm install dotenv --save
```

Create a new file called `.env` in the project root directory:

```
BOT_STATE_TABLE=botdata
STORAGE_CONNECTION=DefaultEndpointsProtocol=https;EndpointSuffix=core.windows.net;AccountName=xxx;AccountKey=xxxx
```

Update code:

```javascript
require('dotenv').config();
...
const BOT_STATE_TABLE = process.env.BOT_STATE_TABLE;
const STORAGE_CONNECTION = process.env.STORAGE_CONNECTION;
...
var azureTableClient = new botbuilder_azure.AzureTableClient(BOT_STATE_TABLE, STORAGE_CONNECTION);
var tableStorage = new botbuilder_azure.AzureBotStorage({ gzipData: false }, azureTableClient);
```

Go to `Bot Service in Azure Portal` --> `Application Settings` --> `App Settings` and add two new fields

* `BOT_STATE_TABLE` --> `botdata`
* `STORAGE_CONNECTION` --> `DefaultEndpointsProtocol=https;EndpointSuffix=core.windows.net;AccountName=xxx;AccountKey=xxxxx`

Alternatively, you can use the `az webapp config appsettings set` Azure CLI command to set the parameters.

If you've created a separate Table for testing, make sure to add the `dev` Table to the `.env` file (will be used locally) and add the `prod` Table's name in the App Service.

Test if your local changes work.

Add `.env` to `.gitignore` and commit changes to Git:

```
git add .gitignore app.js package.json
git commit -m "Proper storing of secrets in .env file"
git push
```

*Note:* There are even cleaner ways to keep your credentials out of the code (and code repository in general), e.g., by using [Azure Key Vault](https://azure.microsoft.com/en-us/services/key-vault/).

Congratulations, you finished the basic bot setup and you're now able to easily push code updates to Azure!

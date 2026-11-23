# Introduction 
TODO: Give a short introduction of your project. Let this section explain the objectives or the motivation behind this project. 

# Getting Started
TODO: Guide users through getting your code up and running on their own system. In this section you can talk about:
1.	Installation process
2.	Software dependencies
3.	Latest releases
4.	API references

# Build and Test
TODO: Describe and show how to build your code and run the tests. 

# Storage provider

Object storage runs on Azure Blob, a folder on the customer's own server, or any
S3-compatible store. One setting chooses, and the same binaries serve all three.

    Storage__Provider = AzureBlob | FileSystem | S3

Unset means `AzureBlob`, so a deployment that has never heard of this setting keeps
behaving exactly as it did. See **`storage-provider.sample.json`** for every key,
which hosts need which, and the traps — the signing key must be real Base64, and
`PublicBaseUrl` must be the address of the host that hands out the link.

Switching does not move your files. Copy them across first, keeping container and
path identical, then flip the setting and restart.

To vary storage **per client** instead, add rows to the Nucleus `ClientSetting`
table; `SFM.Nucleus.Infrastructure/Data/Migrations/CreateClientSettingTable.sql`
lists them. A client with no rows follows the configuration above.
`DLPProcessLaterService` has no database access and always uses configuration, so
if clients are on different storage, run one worker per backend.

# Changing the database

Schema changes ship as **migrations**, never as a script someone runs by hand —
we are database-per-tenant, so a hand-run script lands on one customer and
silently misses the rest. Add `V000n__name.sql` under
`SFM.DLP.API/Migrations/Dlp/`, then `POST /admin/migration/run` after deploy.

See **[DATABASE-CHANGES.md](DATABASE-CHANGES.md)** — the file shape, which of the
three folders to use, and why a migration that touches a table some tenants lack
must skip rather than fail.

# Contribute
TODO: Explain how other users and developers can contribute to make your code better. 

If you want to learn more about creating good readme files then refer the following [guidelines](https://docs.microsoft.com/en-us/azure/devops/repos/git/create-a-readme?view=azure-devops). You can also seek inspiration from the below readme files:
- [ASP.NET Core](https://github.com/aspnet/Home)
- [Visual Studio Code](https://github.com/Microsoft/vscode)
- [Chakra Core](https://github.com/Microsoft/ChakraCore)
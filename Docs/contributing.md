All the code for ImgBot is available on GitHub. We will gladly accept contributions for the service, the website, and the documentation.
If you are unsure what to work on, but still want to contribute you can look for an [existing issue](https://github.com/dabutvin/ImgBot/issues) in the repo.

The following is where you can find out how to get set up to run locally as well as detailed information on exactly how ImgBot works.

### ImgBot Service

The core of ImgBot runs on a serverless stack called [Azure Functions](https://azure.microsoft.com/en-us/services/functions/).
The Function apps are running the image compression, pushing the commits, and opening the Pull Requests.
Once you get the tools you need to work with Azure functions you can run the apps locally. 

You can either get the tools [integrated with Visual Studio](https://blogs.msdn.microsoft.com/webdev/2017/05/10/azure-function-tools-for-visual-studio-2017/) and use `F5` 
or you can [get the CLI](https://github.com/Azure/azure-functions-cli) standalone and use `func run ImgBot.Function`.
If you are using Visual Studio for Mac there is [built-in support](https://docs.microsoft.com/en-us/visualstudio/mac/azure-functions) for Azure functions.

Azure Functions operate on top of storage. To run the function locally you will need to bring your own storage account and add a `local.settings.json` in the root with `AzureWebJobsStorage/AzureWebJobsDashboard` filled out. 

You can see the schema of this file in [the doc](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local#local-settings-file)

Alternatively, running `func init` from the function directory will stamp out everything for you, or if you have a deployed function you can fetch the storage it is using.

`func azure functionapp fetch-app-settings <functionName>`

Now that you are running the service locally, within the root of the repo you will see the following directories:

 - `CompressImagesFunction` - The function that does the work of cloning, compressing, and pushing commits
 - `Install` - The shared library to provide Installation Tokens to the functions
 - `OpenPrFunction` - The function that opens Pull Requests
 - `RouterFunction` - The orchestration layer


The following file locations may be helpful if you are looking for specific functionality:

 - `CommitMessage.cs` - generation of commit message compression report
 - `CompressImages.cs` - clones the repo, compresses the images, pushes the commit
 - `CommitSignature.cs` - uses a pgp private key and password to sign a commit message
 - `ImageQuery.cs` - searches and extracts all the images out of the repo to be optimized
 - `InstallationToken.cs` - uses a pem file and the GitHub API to get access to repos
 - `LocalPath.cs` - generates the location to clone from
 - `PullRequest.cs` - opens the pull request and sets the title and description
 - `Schedule.cs` - logic to limit the frequency of ImgBot PRs

#### Triggers

ImgBot uses [QueueTriggers](https://github.com/Azure/azure-webjobs-sdk/wiki/Queues#trigger) to kick off workflows.

The triggers in place today are `routermessage`, `openprmessage`, `compressimagesmessage`.

Triggers are the entry points for the service.

#### Compression workflow

ImgBot uses [LibGit2Sharp](https://github.com/libgit2/libgit2sharp) to perform all the git operations. 

The following is the high-level workflow for the `CompressImagesFunction`:

 1. Clone the repo
 2. Create the 'imgbot' branch
 3. Run the optimize routine 
 4. Commit the changes
 5. Push the 'imgbot' branch to the remote

The clone directory is read from environment variable `%TMP%` or falls back to `/private/tmp/` if this environment variable is not set.

Once the branch is pushed, ImgBot uses [Octokit](https://github.com/octokit/octokit.net) to create the pull request in GitHub.

#### Installation tokens

ImgBot uses [BouncyCastle](http://www.bouncycastle.org/csharp/) to help generate an [installation token](https://developer.github.com/apps/building-integrations/setting-up-and-registering-github-apps/about-authentication-options-for-github-apps/#authenticating-as-an-installation).

This requires a combination of a Private Key, an App Id, and an InstallationId to generate a token to be used on behalf of the installation.

The benefit of using installation tokens is that the user is in full control of the permissions at all times. We never have to store any tokens and they expire after about 10 minutes.

The installation token serves as a password to clone private repos, push to remotes, and open pull requests.

The username to accompany the installation token password is `x-access-token`.

For security reasons we cannot provide contributors with a pem file as this is a secret that delegates permissions in GitHub. You can run every part of the function except parts where authentication is required without this secret. If you are working on a part of the function that requires this secret then you can generate one for yourself to test with. [Register a GitHub app](https://github.com/settings/apps/new) for development purpose and install this app into the repo you are using to test with. Set the AppId in the code and you should be good to go.

 If there is a part of this process that isn't clear or you have any questions at all feel free to open an issue in this repo and we'll work it out :)

#### Schedules

This Schedule class is responsible for throttling optimization routines.
ImgBot is triggered when there is a new image added to a repo and by default will submit a PR as soon as it can.

Some users prefer to defer the pull requests and do the optimization in bigger batches. This is implemented by offering three options 'daily', 'weekly', and 'monthly'.

ImgBot will check the commit log to find the last time we committed an optimization. If it has been long enough since ImgBot has last committed in this branch then we will try to optimize the images again. Otherwise we will skip running optimizations for this run.

#### Commit messages

ImgBot uses a standard commit title and generates a report of the image optimizations to be used in the commit message body.

The input is a dictionary where the key is the filename and the value is a pair of numbers that represent file size before and after compression.

This dictionary is transformed into an optimization report in the form of a commit message.

#### Image query

ImgBot locates all the images that are to be sent through the optimization routine with file directory access against a local clone.

The known image extensions are used to find all the images recursively and the ignored files from imgbotconfig are parsed.

#### Local Paths

For each execution, ImgBot generates the folder for the git operations to take place in.

Today this is done by combining the name of the repo with a random number.

### ImgBot website

The frontend that drives https://imgbot.net/ is a lightweight web app running on [ASP.NET Core](https://github.com/aspnet/Home). This framework runs on all operating systems :)

The purpose of this website is to run the landing page and docs for ImgBot as well as an endpoint for the webhooks from GitHub for installed repos.

The website uses bootstrap for a grid. To copy the lib files out of node_modules we use grunt (one time only).

```
npm run copy-libs
```

Within the Web directory you will see the following key files

 - `Views/Home/Index.cshtml` - the landing page markup
 - `Views/Home/Docs.cshtml` - the docs page markup
 - `Views/Shared/_Layout.cshtml` - the layout for the landing page
 - `wwwroot/css/site.less` - the landing page stylesheet
 - `Controllers/HookController.cs` - the route for the webhooks
 - `Controllers/HomeController.cs` - the routes for the landing page and the docs
 
The stylesheet is compiled using a grunt task mapped in the `package.json`.

```
npm run compile-less
```

The stylesheet can be compiled on save through grunt as well.

```
npm run watch
```

The 2 main events we deal with through webhooks are installation events and push events.
Each time a repo has ImgBot installed, GitHub fires a hook to `HookController.cs` and we start the installation workflow.

Each time a repo that already has ImgBot installed gets pushed to, GitHub fires a hook to `HookController.cs` and, if the commit contains an image update, we start the compression worflow.

You need to make a connection to a real storage account to run the website. To do this you can add the `appsettings.json` locally with the following format. Note: this should be the same storage account as the service is using for Azure Functions to enable message passing from the website to the service.

```
{
  "Logging": {
    "IncludeScopes": false,
    "LogLevel": {
      "Default": "Warning"
    }
  },
    "Storage": {
        "ConnectionString": "DefaultEndpointsProtocol=https;AccountName=ACCOUNT_NAME;AccountKey=ACCOUNT_KEY;EndpointSuffix=core.windows.net"
    }
}

```

### ImgBot Docs

The docs are published from checked in markdown files to the [ImgBot website](https://imgbot.net/docs) to view in a browser. Alternatively, the docs can be browsed and edited [within GitHub](https://github.com/dabutvin/ImgBot/tree/master/Docs).

To work on the docs within the context of the website locally ImgBot uses a [gruntjs task](https://github.com/treasonx/grunt-markdown) to compile the markdown into HTML. To compile the docs locally run the following npm commands.

```
npm install
npm run compile-docs
``` 

To compile the markdown automatically on save

```
npm run watch
```

Check out the [gruntfile.js](https://github.com/dabutvin/ImgBot/tree/master/ImgBot.Web/gruntfile.js) and [package.json](https://github.com/dabutvin/ImgBot/tree/master/ImgBot.Web/package.json) to see how it is configured.

When the docs are compiled to HTML they are copied into the Web/Docs directory. In this directory there is a `metadata.json` file that will define the order and title of the doc. For example:

```
{
    "slug": "contributing-website",
    "title": "Contribute to ImgBot website"
}
```

The slug matches the name of the markdown file and also drives the URL segment to browse this doc.

This metadata file is read within the [HomeController.cs](https://github.com/dabutvin/ImgBot/tree/master/Web/Controllers/HomeController.cs) file.  The controller arranges all the documentation in memory in preparation for rendering.

The template that renders the documentation is [Docs.cshtml](https://github.com/dabutvin/ImgBot/tree/master/Web/Views/Home/Docs.cshtml). This is a razor file that renders the documentation navigation and content as HTML.

### Tests

ImgBot uses VSTest with NSubstitue to manage unit testing of the logic within the ImgBot codebase.
The tests live in the Test directory.
Please do your best to add tests as new features or logic are added so we can keep Imgbot running smoothly.
:)

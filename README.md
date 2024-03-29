# rl-scanner-cloud template for Azure DevOps Pipelines

ReversingLabs provides the official template for
[Azure DevOps Pipelines](https://learn.microsoft.com/en-us/azure/devops/pipelines/get-started/what-is-azure-pipelines?view=azure-devops)
to enable faster and easier deployment of the
[ReversingLabs Spectra Assure Portal](https://docs.secure.software/portal/integrations/)
solution in CI/CD workflows.

The template provided in this repository uses the official
[ReversingLabs rl-scanner-cloud Docker image](https://hub.docker.com/r/reversinglabs/rl-scanner-cloud) to scan a single build artifact with the Spectra Assure Portal,
generate the analysis report, and display the analysis status.

The template requires that you define the `RLPORTAL_ACCESS_TOKEN` secret environment variable to store your [Portal access token](https://docs.secure.software/api/generate-api-token).

This template is most suitable for experienced users who want to integrate the Spectra Assure Portal into their existing Azure DevOps pipelines.
To successfully work with the template, you should be familiar with the basic
[Azure DevOps Pipelines concepts](https://learn.microsoft.com/en-us/azure/devops/pipelines/get-started/key-pipelines-concepts?view=azure-devops).


## What is the Spectra Assure Portal?

The Spectra Assure Portal is a SaaS solution that's part of the Spectra Assure platform - a new ReversingLabs solution for software supply chain security. More specifically, the Portal is a web-based application for improving and managing the security of your software releases and verifying third-party software used in your organization.

ReversingLabs Spectra Assure Portal is capable of scanning [nearly any type](https://docs.secure.software/concepts/language-coverage) of software artifact or package that results from a build.


## How this template works

This template relies on user-specified [template parameters](#parameters) to:

- create a directory for analysis reports
- use the `rl-scanner-cloud` Docker image to upload and scan a single build artifact on the Spectra Assure Portal 
- place the analysis reports into the previously created directory and optionally publish them as pipeline artifacts
- output the scan result as a build status message (also displayed on the pipeline summary page in Azure DevOps interface).

The template is intended to be used in the `test` stage of a standard build-test-deploy pipeline.
It expects that the build artifact is produced in a previous stage
and requires specifying the location of the artifact with the `BUILD_PATH` parameter.
The path must be relative to `$(System.DefaultWorkingDirectory)`.

Analysis reports generated by the Spectra Assure Portal after scanning the artifact are saved to the location specified with the `REPORT_PATH` parameter.
The reports are always created regardless of the scan result (pass or fail).

By default,
the reports are also automatically uploaded to Azure DevOps Pipelines
and displayed on the job build level in the interface (not in the *Artifacts* tab).
To disable automatic report uploads,
you must explicitly set the `WITH_UPLOAD` template parameter to `false`.


### Requirements

1. **An [Azure DevOps Services account](https://learn.microsoft.com/en-us/azure/devops/pipelines/get-started/pipelines-sign-up?view=azure-devops)**
to create an Azure DevOps organization and use Azure Pipelines.
If you're already in an Azure DevOps organization,
make sure you can access the Azure DevOps project where you want to use this template.

2. **An [Azure Pipelines agent with the Docker capability enabled](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/agents?view=azure-devops&tabs=yaml%2Cbrowser)**.
The example pipeline in this repository runs on a Microsoft-hosted agent using the `ubuntu-latest` VM image.

3. **A valid Portal Access Token**.
The template requires that you define the `RLPORTAL_ACCESS_TOKEN`
secret environment variable to store your [Portal access token](https://docs.secure.software/api/generate-api-token)


## How to use this template

The most common use-case for this template
is to include it in the "test" stage of an existing pipeline,
after the build artifact you want to scan has been created.

1. Copy the template file into the repository associated with your Azure DevOps project.

2. Make sure your Portal access token `RLPORTAL_ACCESS_TOKEN`
is configured as a secret in your Azure DevOps organization.
Add them as a
[variable group](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/set-secret-variables?view=azure-devops&tabs=yaml%2Cbash)
to your pipeline like in the following example:

        variables:
        - group: rl-scanner-cloud

3. Specify the required
[template parameters](#parameters)
in the `variables` section of your pipeline like in the following example:

        variables:
        - group: rl-scanner-cloud
        - name: RLPORTAL_SERVER
          value: test
        - name: RLPORTAL_ORG
          value: Test
        - name: RLPORTAL_GROUP
          value: Default
        - name: RL_PACKAGE_URL
          value: 'ProjectName/PackageName@Version'
        - name: BUILD_PATH
          value: '.'
        - name: MY_ARTIFACT_TO_SCAN
          value my-package.rpm
        - name: REPORT_PATH
          value: report


4. [Include the template](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops&pivots=templates-includes)
into the `steps` section of your pipeline like in the following example:

        steps:
        # placeholder for build step
        - template: secure-software-portal-scan-ado.yml
          parameters:
            RLPORTAL_ACCESS_TOKEN: ${{ variables.RLPORTAL_ACCESS_TOKEN }}
            RLPORTAL_SERVER: ${{ variables.RLPORTAL_SERVER }}
            RLPORTAL_ORG: ${{ variables.RLPORTAL_ORG }}
            RLPORTAL_GROUP: ${{ variables.RLPORTAL_GROUP }}
            RL_PACKAGE_URL: ${{ variables.RL_PACKAGE_URL }}
            BUILD_PATH: ${{ variables.BUILD_PATH }}
            MY_ARTIFACT_TO_SCAN: ${{ variables.MY_ARTIFACT_TO_SCAN }}
            REPORT_PATH: ${{ variables.REPORT_PATH }}
            WITH_UPLOAD: true
            VERBOSE: true
        # placeholder for deploy step


5. Save and commit your changes to the repository/the Azure DevOps project.


### Parameters

The following template parameters can be modified in the pipeline.

**Note:** All optional string parameters have a default empty string value and do not have to be specified if not used.


| Parameter name          | Required | Type    | Description |
| ---------               | ------   | ------  | ------      |
| `RLPORTAL_ACCESS_TOKEN` | **Yes** | string | A Personal Access Token for authenticating requests to the Spectra Assure Portal. Before you can use this example, you must [create the token](https://docs.secure.software/api/generate-api-token) in your Portal settings. Tokens can expire and be revoked, in which case you'll have to update this value. Define it as a secret in a group `rl-scanner-cloud` |
| `RLPORTAL_SERVER`       | **Yes** | string | Name of the Spectra Assure Portal instance to use for the scan. The Portal instance name usually matches the subdirectory of `my.secure.software` in your Portal URL. For example, if your portal URL is `my.secure.software/demo`, the instance name to use with this parameter is `demo`. |
| `RLPORTAL_ORG`          | **Yes** | string | Name of the Spectra Assure Portal organization to use for the scan. The organization must exist on the Portal instance specified with `RLPORTAL_SERVER`. The user account authenticated with the token must be a member of the specified organization and have the appropriate permissions to upload and scan a file. Organization names are case-sensitive. |
| `RLPORTAL_GROUP`        | **Yes** | string | Name of the Spectra Assure Portal group to use for the scan. The group must exist in the Portal organization specified with `RLPORTAL_ORG`. Group names are case-sensitive. |
| `RL_PACKAGE_URL`        | **Yes** | string | The package URL (purl) used to associate the file with a project and package on the Portal. Package URLs are unique identifiers in the format `[pkg:type/]<project></package><@version>`. When scanning a file, you must assign a package URL to it, so that it can be placed into the specified project and package as a version. If the project and package you specified don't exist in the Portal, they will be automatically created.  |
| `BUILD_PATH`            | **Yes** | string | The directory where the build artifact specified with the `MY_ARTIFACT_TO_SCAN` parameter is located. The path must be relative to `$(System.DefaultWorkingDirectory)`. **The default value is `.`** |
| `MY_ARTIFACT_TO_SCAN`   | **Yes** | string | The name of the file you want to scan. Must be relative to `BUILD_PATH`. The file must exist in the specified location before the scan starts. |
| `REPORT_PATH`           | No      | string | The directory where analysis reports will be stored after the scan is finished. The path must be relative to `$(System.DefaultWorkingDirectory)`. The directory must be empty before the scan starts. **The default value is `RlReport`** |
| `RL_WITH_UPLOAD`        | No      | boolean | Automatically uploads analysis reports into the Azure DevOps pipeline after the scan is finished. **The default value is `true`** ; the option is enabled by default. |
| `RL_DIFF_WITH`          | No      | string | Use this parameter to specify the Version part of a previously scanned version of the artifact to compare (diff) against. The previous version must exist in the same project and package as the scanned artifact. |
| `RL_PROXY_SERVER`       | No      | string |An optional proxy server. |
| `RL_PROXY_PORT`         | No      | string | An optional proxy port. |
| `RL_PROXY_USER`         | No      | string | An optional proxy user for authentication. |
| `RL_PROXY_PASSWORD`     | No      | string | An optional proxy password for authentication. |
| `RL_VERBOSE`            | No      | boolean | Includes detailed progress feedback into the pipeline output and displays the `stdout` and `stderr` messages from the run in the Docker container. **The default value is `false`**; the option is disabled by default. |


## Examples

The `azure-pipeline-example.yml` file in this repository
is an example of a basic Azure DevOps pipeline
that uses the ReversingLabs rl-scanner-cloud template.

If you want to try out the template before integrating it into your pipelines,
you can clone this repository and create an Azure DevOps project for it.

The template is already added to the `azure-pipeline-example.yml` file,
so all you need to do is associate the pipeline with your project
and configure your `rl-secure` licensing information
as described previously in this text.

By default, the pipeline will scan this README file
and the scan should pass without any issues.
The example deploy stage will be triggered
and the analysis reports will be uploaded to Azure DevOps as job artifacts.

If you want to test what happens when a scan fails:

- download the [test file](https://secure.eicar.org/eicarcom2.zip)
called `eicarcom2.zip` and add it to the same directory as this README file
(the repository root directory).
This is a test malware that is safe to use
because it only contains a virus signature,
but does not cause any harm to the system.
For more information, check the official
[European Institute for Computer Anti-Virus Research (EICAR)](https://www.eicar.org/) website.

- in the `azure-pipeline-example.yml` file,
replace the current value of the `MY_ARTIFACT_TO_SCAN` parameter with `eicarcom2.zip`

After saving your changes and running the pipeline again,
the scan should fail.
The example deploy stage will not be triggered,
but the analysis reports will still be uploaded to Azure DevOps as job artifacts.
The example pipeline also shows a meaningful output message
to indicate that the scan has failed because of detected malicious components.


## Notes

There is currently no rl-html report.

## Useful resources

- The official Microsoft documentation on [using templates with Azure DevOps Pipelines](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops&pivots=templates-includes)
- The official `reversinglabs/rl-scanner-cloud` Docker image [on Docker Hub](https://hub.docker.com/r/reversinglabs/rl-scanner-cloud)
- [Supported file formats](https://docs.secure.software/concepts/filetypes) and [language coverage](https://docs.secure.software/concepts/language-coverage)
- Introduction to [secure software release processes](https://www.reversinglabs.com/solutions/secure-software-release-processes) with ReversingLabs


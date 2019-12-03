# Validation pipeline example

This example covers how PSRule can be used to validate Azure resource templates within a continuous integration (CI) pipeline.

This scenario covers the following:

- [Installing PSRule within a CI pipeline](#installing-psrule-within-a-ci-pipeline)
- [Exporting rule data for analysis](#exporting-rule-data-for-analysis)
- [Validating exported resources](#validating-exported-resources)
- [Formatting output](#formatting-output)
- [Failing the pipeline](#failing-the-pipeline)
- [Generating NUnit output](#generating-nunit-output)
- [Complete example](#complete-example)
- [Additional options](#additional-options)

## Installing PSRule within a CI pipeline

Typically, PSRule is not pre-installed on CI worker nodes and must be installed within the pipeline.
PSRule PowerShell modules need to be installed prior to calling PSRule cmdlets.

If your CI pipeline runs on a persistent virtual machine that you control, consider pre-installing PSRule.
The following examples focus on installing PSRule dynamically during execution of the pipeline.
Which is suitable for cloud-based CI worker nodes.

To install PSRule within a CI pipeline, execute the `Install-Module` PowerShell cmdlet.

Depending on your environment, the CI worker process may not have administrative permissions.
To install modules into the current context running the CI pipeline use `-Scope CurrentUser`.
The PowerShell Gallery is not a trusted source by default.
Use the `-Force` switch to suppress a prompt to install modules from PowerShell Gallery.

For example:

```powershell
$Null = Install-Module -Name PSRule.Rules.Azure -Scope CurrentUser -Force;
```

Installing `PSRule.Rules.Azure` also installs the base `PSRule` module and associated Azure dependencies.
The `PSRule.Rules.Azure` module includes cmdlets and pre-built rules for validating Azure resources.
Using the pre-built rules is completely optional.

In some cases, installing NuGet and PowerShellGet may be required to connect to the PowerShell Gallery.
The NuGet package provider can be installed using the `Install-PackageProvider` PowerShell cmdlet.

```powershell
$Null = Install-PackageProvider -Name NuGet -Scope CurrentUser -Force;
```

The example below includes both steps together with checks:

```powershell
if ($Null -eq (Get-PackageProvider -Name NuGet -ErrorAction SilentlyContinue)) {
    $Null = Install-PackageProvider -Name NuGet -Scope CurrentUser -Force;
}

if ($Null -eq (Get-InstalledModule -Name PowerShellGet -MinimumVersion 2.2.1 -ErrorAction Ignore)) {
    Install-Module PowerShellGet -MinimumVersion 2.2.1 -Scope CurrentUser -Force -AllowClobber;
}

if ($Null -eq (Get-InstalledModule -Name PSRule.Rules.Azure -MinimumVersion '0.6.0' -ErrorAction SilentlyContinue)) {
    $Null = Install-Module -Name PSRule.Rules.Azure -Scope CurrentUser -MinimumVersion '0.6.0' -Force;
}
```

Add `-AllowPrerelease` to install pre-release versions.
See the [change log](https://github.com/BernieWhite/PSRule.Rules.Azure/blob/master/CHANGELOG.md) for the latest version.

## Exporting rule data for analysis

Azure Resource Manager (ARM) templates are a JSON-based file structure.
While JSON can be natively read by PSRule, ARM templates are typically not static.
Depending on the parameters provided to a template, resources may differ significantly.

Important resource properties that should be validated are often variables, parameters or deployed conditionally.
Under these circumstances, to correctly validate resources in a template, expressions must be resolved.

In PSRule, the `Export-AzTemplateRuleData` cmdlet resolves a template and returns a resultant set of resources.
The resultant set of resources can then be validated.

No connectivity to Azure is required by default when calling `Export-AzTemplateRuleData`.

### Export cmdlet parameters

To run `Export-AzTemplateRuleData` two key parameters are required:

- `-TemplateFile` - An absolute or relative path to the template JSON file.
- `-ParameterFile` - An absolute or relative path to one or more parameter JSON files.

The `-ParameterFile` parameter is optional when all parameters defined in the template have `defaultValue` set.

Optionally the following parameters can be used:

- `-Name` - The name of the deployment. If not specified a default name of `export-<xxxxxxxx>` will be used.
- `-OutputPath` - An absolute or relative path where the resultant resources will be written to JSON.
If not specified the current working path be used.
- `-ResourceGroup` - The name of a resource group where the deployment is intended to be run.
If not specified placeholder values will be used.
- `-Subscription` - The name or subscription Id of a subscription where the deployment is intended to be run.
If not specified placeholder values will be used.

See cmdlet help for a full list of parameters.

If `-OutputPath` is a directory or is not set, the output file will be automatically named `resources-<name>.json`.

For example:

```powershell
Export-AzTemplateRuleData -TemplateFile .\template.json -ParameterFile .\parameters.json;
```

Multiple parameter files that map to the same template can be supplied in a single cmdlet call.
Additional templates can be exported by calling `Export-AzTemplateRuleData` multiple times.

### Use of placeholder values

A number of functions that can be used within Azure templates retrieve information from Azure.
Some examples include `reference`, `subscription`, `resourceGroup`, `list*`.

The default for `Export-AzTemplateRuleData` is to operate without requiring authenticated connectivity to Azure.
As a result, functions that retrieve information from Azure use placeholders such as `{{Subscription.SubscriptionId}}`.

To provide a real value for `subscription` and `resourceGroup` use the `-Subscription` and `-ResourceGroup` parameters.
When using `-Subscription` and `-ResourceGroup` the subscription and resource group must already exist.
Additionally the context running the cmdlet must have at least read access (i.e. `Reader`).

It is currently not possible to provide a real value for `reference` and `list*`, only placeholders will be used.

Key Vault references in parameter files use placeholders instead of the real value to prevent accidental exposure of secrets.

## Validating exported resources

To validate exported resources use `Invoke-PSRule`, `Assert-PSRule` or `Test-PSRuleTarget`.
In a CI pipeline, `Assert-PSRule` is recommended.
`Assert-PSRule` outputs preformatted results ideal for use within a CI pipeline.

Use `Assert-PSRule` with the resolved resource output as an input using `-InputPath`.

In the following example, resources from `.\resources.json` are validated against pre-built rules:

```powershell
Assert-PSRule -InputPath .\resources-export-*.json -Module PSRule.Rules.Azure;
```

Example output:

```text
 -> vnet-001 : Microsoft.Network/virtualNetworks

    [PASS] Azure.Resource.UseTags
    [PASS] Azure.VirtualNetwork.UseNSGs
    [PASS] Azure.VirtualNetwork.SingleDNS
    [PASS] Azure.VirtualNetwork.LocalDNS

 -> vnet-001/subnet2 : Microsoft.Network/virtualNetworks/subnets

    [FAIL] Azure.Resource.UseTags
```

To process multiple input files a wildcard `*` can be used.

```powershell
Assert-PSRule -InputPath .\out\*.json -Module PSRule.Rules.Azure;
```

## Formatting output

When executing a CI pipeline, feedback on any validation failures is important.
The `Assert-PSRule` cmdlet provides easy to read formatted output instead of PowerShell objects.

Additionally, `Assert-PSRule` supports styling formatted output for Azure Pipelines and GitHub Actions.
Use the `-Style AzurePipelines` or `-Style GitHubActions` parameter to style output.

For example:

```powershell
Assert-PSRule -InputPath .\out\*.json -Style AzurePipelines -Module PSRule.Rules.Azure;
```

## Failing the pipeline

When using PSRule within a CI pipeline, a failed rule should stop the pipeline.
When using `Assert-PSRule` if any rules fail, an error will be generated.

```text
Assert-PSRule : One or more rules reported failure.
At line:1 char:1
+ Assert-PSRule -Module PSRule.Rules.Azure -InputPath .\out\tests\Resou ...
+ ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
+ CategoryInfo          : InvalidData: (:) [Assert-PSRule], FailPipelineException
+ FullyQualifiedErrorId : PSRule.Fail,Assert-PSRule
```

A single PowerShell error is typically enough to stop a CI pipeline.
If you are using a different configuration additionally `-ErrorAction Stop` can be used.

For example:

```powershell
Assert-PSRule -Module PSRule.Rules.Azure -InputPath .\out\*.json -ErrorAction Stop;
```

## Generating NUnit output

NUnit is a popular unit test framework for .NET.
NUnit generates a test report format that is widely interpreted by CI systems.
While PSRule does not use NUnit directly, it support outputting validation results in the NUnit3 format.
Using a common format allows integration with any system that supports the NUnit3 for publishing test results.

To generate an NUnit report:

- Use the `-OutputFormat NUnit3` parameter.
- Use the `-OutputPath` parameter to specify the path of the report file to write.

```powershell
Assert-PSRule -OutputFormat NUnit3 -OutputPath .\reports\rule-report.xml -Module PSRule.Rules.Azure -InputPath .\out\*.json;
```

The output path will be created if it does not exist.

### Publishing NUnit report with Azure DevOps

With Azure DevOps, an NUnit report can be published using [Publish Test Results task][publish-test-results].

An example YAML snippet is included below:

```yaml
# PSRule results
- task: PublishTestResults@2
  displayName: 'Publish PSRule results'
  inputs:
    testRunTitle: 'PSRule'
    testRunner: NUnit
    testResultsFiles: 'reports/rule-report.xml'
    mergeTestResults: true
    publishRunAttachments: true
  condition: succeededOrFailed()
```

## Complete example

Putting each of these steps together.

### Install dependencies

```powershell
# Install dependencies for connecting to PowerShell Gallery
if ($Null -eq (Get-PackageProvider -Name NuGet -ErrorAction Ignore)) {
    Install-PackageProvider -Name NuGet -Force -Scope CurrentUser;
}

if ($Null -eq (Get-InstalledModule -Name PowerShellGet -MinimumVersion 2.2.1 -ErrorAction Ignore)) {
    Install-Module PowerShellGet -MinimumVersion 2.2.1 -Scope CurrentUser -Force -AllowClobber;
}
```

### Validate templates

```powershell
# Install PSRule.Rules.Azure module
if ($Null -eq (Get-InstalledModule -Name PSRule.Rules.Azure -MinimumVersion '0.6.0' -ErrorAction SilentlyContinue)) {
    $Null = Install-Module -Name PSRule.Rules.Azure -Scope CurrentUser -MinimumVersion '0.6.0' -Force;
}

# Resolve resources
Export-AzTemplateRuleData -TemplateFile .\template.json -ParameterFile .\parameters.json -OutputPath out/;

# Validate resources
$assertParams = @{
    InputPath = 'out/*.json'
    Module = 'PSRule.Rules.Azure'
    Style = 'AzurePipelines'
    OutputFormat = 'NUnit3'
    OutputPath = 'reports/rule-report.xml'
}
Assert-PSRule @assertParams;
```

### Azure DevOps Pipeline

```yaml
steps:

# Install dependencies
- powershell: ./pipeline-deps.ps1
  displayName: 'Install dependencies'

# Validate templates
- powershell: ./validate-template.ps1
  displayName: 'Validate templates'

# Publish pipeline results
- task: PublishTestResults@2
  displayName: 'Publish PSRule results'
  inputs:
    testRunTitle: 'PSRule'
    testRunner: NUnit
    testResultsFiles: 'reports/rule-report.xml'
    mergeTestResults: true
    publishRunAttachments: true
  condition: succeededOrFailed()
```

## Additional options

### Using Invoke-Build

`Invoke-Build` is a build automation cmdlet that can be installed from the PowerShell Gallery by installing the _InvokeBuild_ module.
Within Invoke-Build, each build process is broken into tasks.

The following example shows an example of using _PSRule.Rules.Azure_ with _InvokeBuild_ tasks.

```powershell
# Synopsis: Install PSRule modules
task InstallPSRule {
    if ($Null -eq (Get-InstalledModule -Name PSRule.Rules.Azure -MinimumVersion '0.6.0' -ErrorAction SilentlyContinue)) {
        $Null = Install-Module -Name PSRule.Rules.Azure -Scope CurrentUser -MinimumVersion '0.6.0' -Force;
    }
}

# Synopsis: Run validation
task ValidateTemplate InstallPSRule, {
    # Resolve resources
    Export-AzTemplateRuleData -TemplateFile .\template.json -ParameterFile .\parameters.json -OutputPath out/;

    # Validate resources
    $assertParams = @{
        InputPath = 'out/*.json'
        Module = 'PSRule.Rules.Azure'
        Style = 'AzurePipelines'
        OutputFormat = 'NUnit3'
        OutputPath = 'reports/rule-report.xml'
    }
    Assert-PSRule @assertParams;
}

# Synopsis: Run all build tasks
task Build ValidateTemplate
```

```powershell
Invoke-Build Build;
```

### Calling from Pester

Pester is a unit test framework for PowerShell that can be installed from the PowerShell Gallery.

Typically, Pester unit tests are built for a particular pipeline.
PSRule can complement Pester unit tests by providing dynamic and sharable rules that are easy to reuse.
By using `-If` or `-Type` pre-conditions, rules can dynamically provide validation for a range of use cases.

When calling PSRule from Pester use `Invoke-PSRule` instead of `Assert-PSRule`.
`Invoke-PSRule` returns validation result objects that can be tested by Pester `Should` conditions.

Additionally, the `Logging.RuleFail` option can be included to generate an error message for each failing rule.

For example:

```powershell
Describe 'Azure' {
    Context 'Resource templates' {
        It 'Use content rules' {
            Export-AzTemplateRuleData -TemplateFile .\template.json -ParameterFile .\parameters.json -OutputPath .\out\resources.json;

            # Validate resources
            $invokeParams = @{
                InputPath = 'out/*.json'
                Module = 'PSRule.Rules.Azure'
                OutputFormat = 'NUnit3'
                OutputPath = 'reports/rule-report.xml'
                Option = (New-PSRuleOption -LoggingRuleFail Error)
            }
            Invoke-PSRule @invokeParams -Outcome Fail,Error | Should -BeNullOrEmpty;
        }
    }
}
```

## More information

- [pipeline-deps.ps1](pipeline-deps.ps1) - Example script installing pipeline dependencies.
- [validate-template.ps1](validate-template.ps1) - Example script for running template validation.
- [azure-pipelines.yaml](azure-pipelines.yaml) - An example Azure DevOps Pipeline.
- [template.json](template.json) - Example template file.
- [parameters.json](parameters.json) - Example parameters file.

[publish-test-results]: https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/test/publish-test-results
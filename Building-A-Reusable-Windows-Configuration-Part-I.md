##Part I - Introduction and Setup

Ever since I learned of the existence of [Ghost](https://www.symantec.com/products/threat-protection/endpoint-management/ghost-solutions-suite) (now owned by Symantec), I have used the concept to create a custom Windows configuration. The process was as follows:

 1. Partition the physical drive in two partitions.
 2. Install Windows on the first partition. Use the second partition for data.
 3. Install drivers.
 4. Install and configure applications.
 5. Use the system for a few days.
 6. Make an image of the Windows partition and store in a safe place.
 7. Use the machine for 3-6 months, re-image back from the image.
 8. Use the system some more, make adjustments and create an incremental image.
	
This model works, and I still use it. Instead of Ghost, I now use Acronis True Image. True Image is more modern, has compression, provides incremental/differential backups, and more. However, installing and configuring applications (#4 above) has always been a manual and lengthy process. In this guide, I will show you a practical way to create a reusable Windows configuration, one which you can apply on any machine.

###Goals

 - Create a script that can run on any Windows machine.
 - Ability to install/remove Windows features.
 - Ability to apply Windows updates.
 - Ability to install our applications of choice either from a local installer or the web.
 - Ability to restore settings for the applications of our choosing.
 - Ability to restore various Windows profile settings.

Let's start!

We will be using the following tools:

###[NuGet](https://www.nuget.org)
NuGet is a package manager that is widely known in the Microsoft world. It's a way to package and redistribute source code, that integrates flawlessly into Visual Studio. NuGet packages have the .nupkg extension, and contain the package manifest (a file called NuSpec using the .nuspec extension, an XML file), and usually the PowerShell script to install the source code/software. The NuGet package file is just a zip file with some metadata attached to it.
###[Chocolatey](https://chocolatey.org)
Chocolatey takes the NuGet concept to the next level. Chocolatey builds on top of NuGet and allows us to use NuGet packages to install not just source code inside our development environment, but also Windows applications and features. If you don't have Chocolatey, go to their page and install it by running the PowerShell provided on their page. The [Chocolatey package gallery](http://chocolatey.org/packages) has over 4000 packages ready to install. All you do is:
```
# Install Google Chrome
choco install googlechrome
```
###[PowerShell](https://technet.microsoft.com/en-us/library/bb978526.aspx)
Powershell is a powerful scripting language built on top of the .NET framework. Think BAT or Linux bash but with the whole .NET framework behind it.

By itself Chocolatey will let you install all the applications you want, but we want more than just applications. Also, by default Chocolatey packages will only install from the web (redistribution rights and all). For our purpose, we want to download the installers once, and be able to run everything locally. To provide some of that functionality, we will need to extend Chocolatey with extensions. Chocolatey extensions are just PowerShell modules that Chocolatey loads and we can put our common functionality in those modules. A Chocolatey extension itself is a NuGet/Chocolatey package. Let's build our extensions first.

Create a new directory, I called mine **Boyan-Choco.Extensions**, inside the directory create a new **.nuspec** file, I called mine **Boyan-Choco.extension.nuspec**. This file will contain the metadata for our extensions. The important metadata are the ID, and the version.

```
<?xml version="1.0" encoding="utf-8"?>
<package xmlns="http://schemas.microsoft.com/packaging/2015/06/nuspec.xsd">
  <metadata>
    <id>Boyan-Choco.extension</id>
    <version>1.0.0</version>
    <title>Install Helpers</title>
    <authors>Boyan Kostadinov</authors>
    <projectUrl>https://github.com/Boyan-Kostadinov/Packages</projectUrl>
    <owners>Boyan Kostadinov</owners>
    <packageSourceUrl>https://github.com/Boyan-Kostadinov/BoxStarter</packageSourceUrl>
    <summary>Common Install Functionality</summary>
    <description>
    Chocolatey installation extensions
    </description>
  </metadata>
</package>
```

The actual PowerShell modules will reside in the **extensions** directory (it must be called **extensions**), so create one inside the same directory where the NuSpec file is. Let's create some essential PowerShell functions. We will split functionality into separate PowerShell modules (PowerShell modules end with the **.psm1** extension). Let's create the following:

 - **[ChocoHelpers.psm1](https://github.com/Boyan-Kostadinov/BoxStarter/blob/master/Boyan-Choco.extension/extensions/ChocoHelpers.psm1)** - Will contain functions related to Chocolatey itself.
 - **[FileHelpers.psm1](https://github.com/Boyan-Kostadinov/BoxStarter/blob/master/Boyan-Choco.extension/extensions/FileHelpers.psm1)** - Will contain any file related functions.
 - **[InstallHelpers.psm1](https://github.com/Boyan-Kostadinov/BoxStarter/blob/master/Boyan-Choco.extension/extensions/InstallHelpers.psm1)** - Will contain any functions that will help us install packages.
 - **[RegisteryHelpers.psm1](https://github.com/Boyan-Kostadinov/BoxStarter/blob/master/Boyan-Choco.extension/extensions/..psm1)** - Will contain any functions related to manipulating the registry.
 - **[SystemHelpers.psm1](https://github.com/Boyan-Kostadinov/BoxStarter/blob/master/Boyan-Choco.extension/extensions/SystemHelpers.psm1)** - Will contain any system related functions.
 - **[WindowsFeatures.psm1](https://github.com/Boyan-Kostadinov/BoxStarter/blob/master/Boyan-Choco.extension/extensions/WindowsFeatures.psm1)** - Will contain functions related to dealing with Windows features.

Note: I am not going to cover every function in every module. The full source can be found under my GitHub repository at https://github.com/Boyan-Kostadinov/BoxStarter/tree/master/Boyan-Choco.extension.

Let's implement some of the functions in the **[ChocoHelpers](https://github.com/Boyan-Kostadinov/BoxStarter/blob/master/Boyan-Choco.extension/extensions/ChocoHelpers.psm1)** module. 

First is the **Invoke-Commands** function. It will take a file, and a line template, and for each line that doesn't start with a **#** (which is the PowerShell character for a comment), it will replace the **##token##** with the current file line, and  execute that expression by using the PowerShell **Invoke-Expression**
```
function Invoke-Commands([string] $file, [string] $commandTemplate) {
    try {
        foreach ($line in Get-Content -Path $file | Where-Object {$_.trim() -notmatch '(^\s*$)|(^#)'})
        {
            $commmand = $commandTemplate.replace("##token##", $line)

            Write-Host "Running: $commmand"

            Invoke-Expression $commmand
        }
    }
    catch {
         Write-Host "Failed: $($_.Exception.Message)"
    }
}
```
This way we can pass a file with Chocolatey packages, one per line, and have Chocolatey install each one. Our file would look like so:
```
GoogleChrome
JRE8
MsSQLServer2014Express --packageParameter "….." --installArguments "…."
```
And the template we will pass to the above function will be:
```
choco install ##token## --execution-timeout 14400 -y
```
Where we call **choco** (shortcut for Chocolatey) with **install** and execution timeout of 4 hours. We also pass the **-y** flag so Chocolatey does not asks us for confirmation for every package.

The function that abstracts that is fairly simple. It takes only the file containing our Chocolatey packages:
```
function Install-Applications([string] $file)
{
    Write-Host "Installing Applications from $file"

    if ($env:packagesSource) {
        $packagesSource = "-s ""$env:packagesSource;chocolatey"""
    } 

    Invoke-Commands $file "choco install ##token## --execution-timeout 14400 -y $packagesSource"
}
```
Onto the next helper, the **[FileHelper](https://github.com/Boyan-Kostadinov/BoxStarter/blob/master/Boyan-Choco.extension/extensions/FileHelpers.psm1)** module.

The most important function here is the **Get-ConfigurationFile** function. This function will be used to get our Chocolatey package configuration file. The file can be a local file, embedded in the package, or a URL of a remote file, that will be downloaded. The function will also take a parameter for a default configuration file, in case one wasn't provided by the user. Here is the full listing:
```
function Get-ConfigurationFile()
{
    param(
        [string] $configuration,
        [string] $defaultConfiguration
    )

    if ([System.IO.File]::Exists($configuration))
    {
        return $configuration
    }

    if (($configuration -as [System.URI]).AbsoluteURI -ne $null)
    {
        $localConfiguration = Join-Path $env:Temp (Split-Path -leaf $defaultConfiguration)

        if (Test-Path $localConfiguration)
        {
            Remove-Item $localConfiguration
        }

        Get-ChocolateyWebFile 'ConfigurationFile' $localConfiguration $configuration | Out-Null

        return $localConfiguration
    }

    return $defaultConfiguration
}
```
The **[FileHelper](https://github.com/Boyan-Kostadinov/BoxStarter/blob/master/Boyan-Choco.extension/extensions/FileHelpers.psm1)** also has unzip functions, path helpers, etc., nothing interesting to write home about.

The most interesting, and most useful of the helpers is the **[InstallHelper](https://github.com/Boyan-Kostadinov/BoxStarter/blob/master/Boyan-Choco.extension/extensions/InstallHelpers.psm1)**. I will cover the **Get-InstallerPath**, **Install-LocalOrRemote** and the **Install-WithScheduledTask** functions.

The purpose of **Get-InstallerPath** is to look for a setup executable, installer executable, installer path, an ISO image or a URL provided in the arguments. It uses the **Get-Parameters** function to parse the Chocolatey provided parameters, and then looks for one of the following in this exact order:

	1. Setup path - An executable path provided with the parameter **/setup="Path.To.Exe"**
	2. Installer path - An executable that unpacks the setup for the package.
	3. A package installer - A combination of the path defined in **$env:packagesInstallers** and the **file** argument.
	4. An ISO path - The path to the ISO image provided with the parameter **/iso="Path.To.Exe"**
	5. A URL to the installer - Provided in the url argument. The file is downloaded and stored in the file argument.

The point of all this is to account for all possible scenarios when it comes to installing a package. This will make more sense when we get to the package creation stage.

Once we figured out where the executable of the package is, **Install-LocalOrRemote**, is really simple. It gets the file from the **file** argument and it uses the Chocolatey function **Install-ChocolateyInstallPackage** to install the package. The full listing:
```
function Install-LocalOrRemote()
{
    param(
        [Hashtable] $arguments
    )
    
    $arguments['file'] = Get-InstallerPath $arguments

    if ([System.IO.File]::Exists($arguments['file']))
    {
        Write-Debug "Installing from: $($arguments['file'])"
        
        Install-ChocolateyInstallPackage @packageArgs

        CleanUp
    }
    else {
        throw 'No Installer or Url Provided. Aborting...'
    }
}
```
Things get slightly more interesting with the **Install-WithScheduledTask**. Because some installers (I'm looking at you **Spotify**, shame on you!) don't allow you to install the application if you are running as administrator, we have to come up with clever hacks to get around that. This function uses the **Windows Task Scheduler** to schedule a task with the installer executable, start that task and then delete the scheduled task. The **StartAsScheduledTask** function is in the **[SystemHelpers](https://github.com/Boyan-Kostadinov/BoxStarter/blob/master/Boyan-Choco.extension/extensions/SystemHelpers.psm1)** module. Here is the full listing:
```
function StartAsScheduledTask() {
    param(
        [string] $name,
        [string] $executable,
        [string] $arguments
    )

    $action = New-ScheduledTaskAction -Execute $executable -Argument $arguments
    $trigger = New-ScheduledTaskTrigger -Once -At (Get-Date)

    Register-ScheduledTask -TaskName $name -Action $action -Trigger $trigger
    Start-ScheduledTask -TaskName $name
    Start-Sleep -s 1
    Unregister-ScheduledTask -TaskName $name -Confirm:$false
}
```
Next comes the **[RegistryHelper](https://github.com/Boyan-Kostadinov/BoxStarter/blob/master/Boyan-Choco.extension/extensions/RegistryHelpers.psm1)** module. It contains the following registry related functions:

 - **Test-RegistryValue** - Checks if the registry value exists at the provided registry path.
 - **Import-RegistrySettings** - Given a file system path, it iterates over all the .reg files found and imports them with the Windows registry utility regedit.
 - **Import-RegistryFile** - Given a file, executable and a process name, it starts the executable, kills the process with the process name provided, and then imports the registry file provided in the file argument. The purpose is to start the program, and have it create it's default settings, kill it, and then import the provided registry settings.

The **[SystemHelpers](https://github.com/Boyan-Kostadinov/BoxStarter/blob/master/Boyan-Choco.extension/extensions/SystemHelpers.psm1)** module is nothing special. It defines the **StartAsScheduledTask** function and the **IsSystem32Bit** function.

The [**WindowsFeatures**](https://github.com/Boyan-Kostadinov/BoxStarter/blob/master/Boyan-Choco.extension/extensions/WindowsFeatures.psm1) module is slightly more interesting. It defines a way to install Windows features by using Chocolatey (and to be exact the special source in Chocolatey called **WindowsFeatures**). 
	- **Enable-WindowsFeatures** - Provided a file path, passes that file to Invoke-Commands, using the template choco install ##token## -r -source WindowsFeatures -y.
	- **Disable-WindwosFeature** - Does the opposite of Enable-WindowsFeatures by using choco uninstall.
	- **Enable-WindowsFeature** - Enables a single Windows feature where the argument is the feature name. Uses **Get-WindowsOptionalFeature** and  **Enable-WindowsOptionalFeature**, which are only available in Windows 2016 and Windows 10. To get the name of the features you can run choco list -source WindowsFeatures.

Now that we have all the modules defined, we can build our extensions package by navigating to the directory and running
```
choco pack Boyan-Choco.extensions.nuspec
```
That will produce a **Boyan-Choco.extension.1.0.0.nupkg**, which you can install by hand with
```
choco install Boyan-Choco.extension -source P:\ath\To\NuPkg\Directory
```
I say by hand, because we will not be installing the extension package, but instead it will be a dependencies that gets installed as part of the  packages we will create.

####Continue to Part II - Creating Your Packages...Coming Soon
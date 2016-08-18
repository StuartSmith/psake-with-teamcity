<h1> PSAKI with Team city </h1>

The is an example project to work with team city and PSAKE.

A few things need to be known before we can start this project. One is we need to create a boot strapper to start the PSAKE build process.

Here is a sample boot strapper, the boot strapper 
will ...
<ul>

<li>Remove any installed modules of PSAKE</li>
<li>Find all the versions of PSake installed which should only be one</li>
<li>Load the latest version of PSake installed</li>
</ul>

To install nuget to install psake run the following command <br>
nuget.exe update Solution file<br>
ie<br>
nuget.exe update psake.sln<br>


    cls

    # '[p]sake' is the same as 'psake' but $Error is not polluted
    remove-module [p]sake

    # Find psake's path, also finds the latest version of PSake
    # Run solution with Nuget to get latest version 
    $psakeModule = (Get-ChildItem (".\Packages\psake*\tools\psake.psm1")).FullName | Sort-Object $_ | select -last 1
    
    Import-Module $psakeModule

    # you can put arguments to task in multiple lines using `
    Invoke-psake -buildFile .\Build\default.ps1 `
			 -taskList Test `
			 -framework 4.6 `
		     -properties @{ 
				 "buildConfiguration" = "Release"
				 "buildPlatform" = "Any CPU"} `
			 -parameters @{ 
				 "solutionFile" = "..\psake.sln"}

    Write-Host "Build exit code:" $LastExitCode

    # Propagating the exit code so that builds actually fail when there is a problem
    exit $LastExitCode


Now that we have a boot strapper it needs to call a build project to compile our Solution and Project files

    Example PSake Build file 

    properties
    {
      $cleanMessage='Executed Clean!'
      $testMessage='Executed Test!'
      $solutionDirectory = (Get-Item $solutionFile).DirectoryName
      $outputDirectory = "$solutionDirectory\.build"
      $temporaryOutputDirectory = "$outputDirectory\temp"
      $buildConfiguration = "Release"
      $buildPlatform = "Any CPU"
    }

      FormatTaskName "`r`n`r`n-------- Executing {0} Task --------"

      task default -depends Test

      task Init `
          -description "Initialises the build by removing previous artifacts and creating output directories" `
          -requiredVariables outputDirectory, temporaryOutputDirectory `
	  	{
	        	Write-Output "buildConfiguration $buildConfiguration"
	        	Assert ("Debug", "Release" -contains $buildConfiguration) `
	        	"Invalid build configuration '$buildConfiguration'. Valid values are 'Debug' or 'Release'"
	        	
	        	Write-Output "buildConfiguration $buildPlatform"
	        	Assert ("x86", "x64", "Any CPU" -contains $buildPlatform)
	        		"Invalid build platform '$buildPlatform'. Valid values are 'x86', 'x64' or 'Any CPU'"
	        		
	        	# Remove previous build results
	        	if (Test-Path $outputDirectory) 
	        	{
	        		Write-Host "Removing output directory located at $outputDirectory"
	        		Remove-Item $outputDirectory -Force -Recurse
	        	}
				Write-Host "Creating output directory located at $outputDirectory"
				New-Item $outputDirectory -ItemType Directory | Out-Null
	
				Write-Host "Creating temporary directory located at $temporaryOutputDirectory"
				New-Item $temporaryOutputDirectory -ItemType Directory | Out-Null
		}
 
 
 <h2>#MSbuild task must be wrapped in an Exec so the script will stop executing if the MSBuild tasks fails</h2>
 
	task Compile `
		-depends Init `
		-description "Compile the code" `
		-requiredVariables solutionFile, buildConfiguration, buildPlatform, temporaryOutputDirectory `
		{ 
		Write-Host "Building solution $solutionFile"
			Exec { 
			msbuild $SolutionFile
			"/p:Configuration=$buildConfiguration;Platform=$buildPlatform;OutDir=$temporaryOutputDirectory"
		}

	task Clean -description "Remove temporary files"
		{
		Write-Host $cleanMessage
		}
		
	
	task Test -depends Compile, Clean -description "Run unit tests" 
		{ 
		Write-Host $testMessage
		}

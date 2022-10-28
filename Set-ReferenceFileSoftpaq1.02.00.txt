<#
    Set-ReferenceFileSoftpaq
    by Dan Felman/HP Inc

    Script will parse an HPIA reference file and replace a Softpaq solution 
    with a superseded version (anywhere it's referenced)

    If changes are made, a backup is made of the original with extension '.orig'

    8/2/2022 Version 1.01.00 - obtain Ref File with CMSL command
             add -platform -OS and -OSVer in command line
    8/3/2022 Version 1.01.01 Better management of cache folder
             Using current folder for updated reference file
    8/9/2022 Version 1.01.02 fixed -ToReplace search error
    8/9/2022 Version 1.01.03 Added source reference file option
    8/15/2022 Version 1.01.05 created functions
             Fixed issue where superseded entry was ALSO in main /Solutions
    8/16/2022 Version 1.01.06 Added -ListNoSupersedes switch
             Added -ListByCategory <category array> (e.g. bios, or 'bios,driver')
    8/17/2022 Version 1.01.10 added -ListSuperseded <SoftpaqID>, fixed bugs
    8/29/2022 Version 1.10.11 made function out of -ListSuperseded option
    10/27/2022 Version 1.10.12 Fix bug when $ToSoftpaq has no Supersedes entry (last element in chain)
    10/28/2022 Version 1.10.13 Fix bug with -ReferenceFile code
    10/28/2022 Version 1.10.14 ...

    Notes:
        -ReplaceSoftpaq AND -ToSoftpaq <Softpaq_IDs> MUST exist in Reference File
    
        Find Softpaqs: Ex Find Nvidia driver Softpaq with CMSL
            get-softpaqlist | ? { $_.name -match 'nvidia' }
            get-softpaqlist | ? { $_.Category -match 'network' -and ($_.name -match 'Intel') }

    Options: 
        -Platform <SysID>               # REQUIRED - also positional
        -OS Win10|win11                 # REQUIRED - also positional
        -OSVer <as per Get-SoftpaqList> # REQUIRED - also positional
        -ReplaceSoftpaq <Softpaq_ID>    # REQUIRED
                                        # a .bak file is created, but ONLY ONCE, then it is overwritten
        [-ToSoftpaq <Softpaq_ID>]       # will use 'Previous' SOfptaq if omitted from command line
        [-CacheDir <path>]              # where Reference file will be downloaded
                                          If omitted, will use current folder
        [-ReferenceFile <path>]         # location of reference XML file to modify
                                          instead of d/l latest from HP
                                          (-CacheDir option not used in this case)
        [-ListNoSupersedes]             # Lists Softpaq with no Superseded entry
        [-ListByCategory]               # lists Softpaqs by Category
        [-ListSuperseded <Softpaq_ID>]  # Softpaq_ID must be latest recommendation

        All output can be routed to text file: ' > out.txt'

    Examples:
         // download and update a reference file
        Set-ReferenceFileSoftpaq -Platform 842a -OS win10 -OSVer 2009 -ReplaceSoftpaq sp139952 [-ToSoftpaq sp139166] [CacheDir <path>]

         // update already downloaded reference file, make a backup of existing file (ONCE)
        Set-ReferenceFileSoftpaq 842a win10 21H2 -ReplaceSoftpaq sp139952 -ReferenceFile .\842a_64_10.0.2009.xml

         // show Softpaqs that do not supersede any version
        Set-ReferenceFileSoftpaq 842a win10 21H2 -ListNoSupersedes | ListByCategory <bios,firmware,driver,dock,etc>

        // find the "intel wlan" driver in reference file
        Set-ReferenceFileSoftpaq1.01.06.ps1 842a win10 2009 -ListByCategory 'driver' | 
            where { $_ -match 'intel wlan'} 

        // list the superseded chain for a Softpaq
        Set-ReferenceFileSoftpaq1.01.06.ps1 842a win10 2009 -ListSuperseded sp139952

#>
param(
    [Parameter( Mandatory = $True, Position = 0 )] 
    [string]$Platform,
    [Parameter( Mandatory = $True, Position = 1 )] [ValidateSet('win10', 'win11')]
    [string]$OS,
    [Parameter( Mandatory = $True, Position = 2 )] 
    [string]$OSVer,
    [Parameter( Mandatory = $false )] 
    $CacheDir,
    [Parameter( Mandatory = $false )] 
    $ReplaceSoftpaq,
    [Parameter( Mandatory = $false )] 
    $ToSoftpaq,
    [Parameter( Mandatory = $false )] 
    $ReferenceFile,
    [Parameter( Mandatory = $false )] 
    [switch]$ListNoSupersedes,
    [Parameter( Mandatory = $false )] 
    $ListByCategory,
    [Parameter( Mandatory = $false )] 
    $ListSuperseded
) # param

$ReFileVersion = '1.02.00 Oct-28-2022'
'Set-ReferenceFileSoftpaq - version '+$ReFileVersion

#################################################################
# Function Get_ReferenceFileArg
#
#   1) copy reference file argument to the caching folde
#   2) if file with same reference file name exists in 
#      current folder, renames it as .bak (only once)
#   3) copies file from cache folder to current folder
#
#   Returns: path of reference file in current folder
#################################################################

Function Get_ReferenceFileArg {
    [CmdletBinding()]
	param( $pReferenceFile, $pCacheDir ) 

    if ( Test-Path $pReferenceFile ) {
        $f_CurrentFolder = Get-Location
        $f_ReferenceFileFolder = Split-Path -Path $pReferenceFile -Parent
        $pReferenceFileName = Split-Path -Path $pReferenceFile -Leaf
        $f_DestinationXmlFile = $f_CurrentFolder.Path+'\'+$pReferenceFileName       # Destination path
        $pReferenceFile = (Resolve-Path -Path $pReferenceFile).Path

        # if Reference File Argument is already in Current Folder, nothing to do (e.g., paths match)
        if ( (Join-Path $f_DestinationXmlFile '') -eq (Join-Path $pReferenceFile '') ) {
            '-- Use existing Reference File' | out-host
            $f_DestinationXmlFile = $pReferenceFile
        } else {
            Try {
                $Error.Clear()
                if ( Test-Path $f_DestinationXmlFile) {
                    Move-Item -Path $f_DestinationXmlFile -Destination $f_DestinationXmlFile'.bak' -Force -EA Stop
                }
                Copy-Item $pReferenceFile -Destination $f_DestinationXmlFile -Force -EA Stop
            } catch {
                $error[0].exception          # $error[0].exception.gettype().fullname 
                exit 2
            }
        } # else if ( (Join-Path $f_DestinationXmlFile '') -eq (Join-Path $pReferenceFile '') )
    } else {
        '-- Reference File does not exist'
        exit 1
    } # else if ( Test-Path $pReferenceFile )

    return $f_DestinationXmlFile

} # Function Get_ReferenceFileArg
#################################################################

#################################################################
# Function Get_ReferenceFileFromHP
#
#   1) retrieves latest reference file from HP to cache folder
#      (with CMSL Get-SofptaqList)
#   2) finds downloaded reference (xml) file
#   3) copies file from cache folder to current folder
#      replacing file if same file name exists in folder
#
#   Returns: path of reference file in current folder
#################################################################

Function Get_ReferenceFileFromHP {
    [CmdletBinding()]
	param( $pPlatform, $pOS, $pOSVer, $pCacheDir ) 

    Try {
        $Error.Clear()
        get-softpaqList -platform $pPlatform -OS $pOS -OSVer $pOSVer -Overwrite 'Yes' -CacheDir $pCacheDir -EA Stop | Out-Null
    } Catch {
        $error[0].exception          # $error[0].exception.gettype().fullname 
        return
    }
    # find the downloaded Reference_File.xml file
    $f_XmlFile = Get-Childitem -Path $pCacheDir'\cache' -Include "*.xml" -Recurse -File |
        where { ($_.Directory -match '.dir') -and ($_.Name -match $pPlatform) `
            -and ($_.Name -match $pOS.Substring(3)) -and ($_.Name -match $pOSVer) }

    Copy-Item $f_XmlFile -Destination $Pwd.Path -Force

    return "$($Pwd.Path)\$($f_XmlFile.Name)"   # final destination in current folder

} # Function Get_ReferenceFileFromHP
#################################################################

#################################################################
# Function ListSupersededChain
#
#   1) retrieves latest reference file from HP to cache folder
#      (with CMSL Get-SofptaqList)
#   2) scans the supersede chaing for the argument
#
#################################################################

Function ListSupersededChain { 
    [CmdletBinding()]
	param( $pSolutionsNodes, $pssNodes, [string]$pListSuperseded ) 

    $f_ssNode = $pSolutionsNodes | where { $_.id -eq $pListSuperseded }
    if ( $f_ssNode ) {
        "// List of Superseded Softpaqs for $pListSuperseded $($f_ssNode.name)"
        "   $($f_ssNode.id) / $($f_ssNode.version)"
        # check if superseded is in /Solutions by mistake first (assume only possible once)
        $f_ssNodeNext = $pSolutionsNodes | where { $_.id -eq $f_ssNode.supersedes }
        if ( $f_ssNodeNext ) {
            ".... $($f_ssNodeNext.id) / $($f_ssNodeNext.version)"
            $f_ssNode = $pSolutionsNodes | where { $_.id -eq $f_ssNode.supersedes }
        }
        # ... find and list the superseded chain of Softpaqs
        do {
            if ( $f_ssNode = $pssNodes | where { $_.id -eq $f_ssNode.supersedes } ) {
                "   $($f_ssNode.id) / $($f_ssNode.version)"
            } else {
                break
            }
        } while ( $f_ssNode -ne $null )
    } else {
        'Softpaq not found'
    } # if ( $f_ssNode )

} # Function ListSupersededChain

#################################################################
# Step 1. make sure Reference File is available
#################################################################

# Set up the Cache folder path that hosts the TEMP reference file

if ( $CacheDir -eq $null ) { 
    $CacheDir = $Pwd.Path 
}
$CacheDir = (Convert-Path $CacheDir)

# check for -ReferenceFile argument, in case we should use it

if ( $ReferenceFile -eq $null ) {
    $ReferenceFile = Get_ReferenceFileFromHP $Platform $OS $OSVer $CacheDir
    "-- Caching folder: $CacheDir\cache"
} else {
    $ReferenceFile = Get_ReferenceFileArg $ReferenceFile $CacheDir
    "-- Caching folder: $CacheDir"
    } # else if ( $ReferenceFile -ne $null )

Try {
    "-- Working Reference file: '$ReferenceFile'"
    $Error.Clear()
    $xmlContent = [xml](Get-Content -Path $ReferenceFile)
} Catch {
    $error[0].exception          # $error[0].exception.gettype().fullname 
    return 3
}

# get each section of the XML file
$SystemNode = $xmlContent.SelectNodes("ImagePal/SystemInfo")
$SolutionsNodes = $xmlContent.SelectNodes("ImagePal/Solutions/UpdateInfo")
$ssNodes = $xmlContent.SelectNodes("ImagePal/Solutions-Superseded/UpdateInfo")
$swInstalledNodes = $xmlContent.SelectNodes("ImagePal/SystemInfo/SoftwareInstalled/Software")
$deviceNodes = $xmlContent.SelectNodes("ImagePal/Devices/Device")

###################################################
# List all Softpaqs that do not supersede any other
###################################################
# find all Softpaqs that do not Superseded any other
if ( $ListNoSupersedes ) {
    "// Liting Softpaqs with no Superseded version"
    foreach ( $entry in $SolutionsNodes ) {
        if ( $entry.Supersedes -eq $null ) {
            "   $($entry.id) $($entry.name) / $($entry.version)"
        }
    } # foreach ( $entry in $SolutionsNodes )
    return
} # if ( $ListNoSupersedes )

###################################################
# List Softpaqs by each category (as per cmd line)
###################################################
if ( $ListByCategory ) {
    [array]$CatArray = $ListByCategory.split(',')
    "// Listing by Category $ListByCategory"

    foreach ( $i in $CatArray ) {
        "// Category: $($i)"
        $SoftpaqByCategoryNodes = $SolutionsNodes | where { $_.category -match $i }
        foreach ( $sol in $SoftpaqByCategoryNodes ) {
            if ( $sol.category -match $i ) {
                "   $($sol.id) $($sol.name) / $($sol.version)"
            }
        } # foreach ( $sol in $SoftpaqByCategoryNodes )
    } # foreach ( $i in $CatArray )
    return
} # if ( $ListCategory )

###################################################
# List the Superseded chain for a specific Softpaq
###################################################
if ( $ListSuperseded ) {
    ListSupersededChain $SolutionsNodes $ssNodes $ListSuperseded
    return # nothing else to do, so exit
} # if ( $ListSuperseded -ne $null )

#################################################################
# Step 2. Find the Softpaq to replace and its replacement in file
#################################################################

###################################################
# Find -ReplaceSoftpaq to replace in /Solutions
###################################################
$SoftpaqNode = $SolutionsNodes | where { $_.id -eq $ReplaceSoftpaq }

"-- Begin XML reference file modification"
if ( $SoftpaqNode -eq $null ) {
    return "Softpaq $ReplaceSoftpaq not found in Reference File"
}
"-- /Solutions: ReplaceSoftpaq Found - $ReplaceSoftpaq/$($SoftpaqNode.Version) - $($SoftpaqNode.Category)"

###################################################
# Find -ToSoftpaq in /Solutions-Superseded
###################################################

if ( $ToSoftpaq -eq $null ) { $ToSoftpaq = $SoftpaqNode.Supersedes }
if ( $ToSoftpaq -eq $null ) { return '-- Error: No superseded Softpaq listed' }

# ... first check for the node in /Solutions (can be a file ERROR - SHOULD BE REPORTED)
$ssNode = $SolutionsNodes | where { $_.id -eq $ToSoftpaq }

if ( $ssNode.id -eq $null ) {

    # ... next, search the supersede chain for the Softpaq node
    do {
        #$ssNode = $ssNodes | where { $_.id -eq $SSSoftpaqID }
        $ssNode = $ssNodes | where { $_.id -eq $ToSoftpaq }
        if ( ($SSSoftpaqID = $ssNode.Supersedes) -eq $null) { break }
    } while ( ($ssNode.id -ne $ToSoftpaq) -and ($SSSoftpaqID -ne $null) )

    if ( $ssNode.id -ne $ToSoftpaq ) {
        if ( $ssNode -eq $null ) {
            return "-- ToSoftpaq not found - $($ToSoftpaq) must be a superseded Softpaq for $($SoftpaqNode.id)"
        } else {
            "-- /Solutions: ToSoftpaq found - $($ssNode.id)/$($ssNode.Version)"
        } # else if ( $ssNode -eq $null )
    } else {
        "-- /Solutions-Superseded: ToSoftpaq found - $($ssNode.id)/$($ssNode.Version)"
    } # else if ( $ssNode.id -ne $ToSoftpaq )

} # if ( $ssNode.id -eq $null )

#################################################################
# Step 3. Replace content with superseded Softpaq node
#################################################################

###################################################
# Handle the case this is a BIOS
###################################################
#if BIOS Softpaq, check /System node area (top of file)

if ( ($SoftpaqNode.Category -eq 'BIOS') -and ($SystemNode.System.Solutions.UpdateInfo.IdRef -eq $ReplaceSoftpaq) ) {
    $SystemNode.System.Solutions.UpdateInfo.IdRef = $ssNode.Id 
    "-- /System: (/Category:BIOS) updated /UpdateInfo IdRef= entry"
}

###################################################
# Solutions: Replace contents of Softpaq node w/replacement
###################################################

if ( $null -ne $ssNode.Supersedes ) {
    $SoftpaqNode.Supersedes = [string]$ssNode.Supersedes
} else {
    $SoftpaqNode.RemoveAttribute("Supersedes")
}
$SoftpaqNode.ColId = $ssNode.ColId
$SoftpaqNode.ItemId = $ssNode.ItemId
$SoftpaqNode.Id = $ssNode.Id
$SoftpaqNode.Name = $ssNode.Name
$SoftpaqNode.Category = $ssNode.Category
$SoftpaqNode.Version = $ssNode.Version
$SoftpaqNode.Vendor = $ssNode.Vendor
$SoftpaqNode.ReleaseType = $ssNode.ReleaseType
$SoftpaqNode.SSMCompliant = $ssNode.SSMCompliant
$SoftpaqNode.DPBCompliant = $ssNode.DPBCompliant
$SoftpaqNode.SilentInstall = $ssNode.SilentInstall
$SoftpaqNode.Url = $ssNode.Url
$SoftpaqNode.ReleaseNotesUrl = $ssNode.ReleaseNotesUrl
$SoftpaqNode.CvaUrl = $ssNode.CvaUrl
$SoftpaqNode.MD5 = $ssNode.MD5
$SoftpaqNode.SHA256 = $ssNode.SHA256
$SoftpaqNode.Size = $ssNode.Size
$SoftpaqNode.DateReleased = $ssNode.DateReleased
$SoftpaqNode.SupportedLanguages = $ssNode.SupportedLanguages
$SoftpaqNode.SupportedOS = $ssNode.SupportedOS
$SoftpaqNode.Description = $ssNode.Description
"-- /solutions: ReplaceSoftpaq node Updated [$($ReplaceSoftpaq) with $($ssNode.id)]"

###################################################
# SoftwareInstalled: Upate contents of node w/replacement
###################################################
$swInstalledFound = $false
foreach ( $sw in $swInstalledNodes ) {
    if ( $sw.Solutions.UpdateInfo.IdRef -eq $ReplaceSoftpaq ) {
        $sw.Version = [string]$ssNode.Version
        $sw.Vendor = [string]$ssNode.Vendor
        $sw.Solutions.UpdateInfo.IdRef = $SoftpaqNode.Id
        $swInstalledFound = $true
    }
} # foreach ( $sw in $swInstalledNodes )

if ( $swInstalledFound ) {
    "-- /SoftwareInstalled: Replaced values for ReplaceSoftpaq $($ReplaceSoftpaq)"
} else {
    "-- /SoftwareInstalled: No matches found for ReplaceSoftpaq $($ReplaceSoftpaq)"
}

$DeviceCount = 0
foreach ( $dev in $deviceNodes ) {
    if ( $dev.Solutions.UpdateInfo.IdRef -eq $ReplaceSoftpaq ) {
        $DeviceCount += 1
        $dev.DriverDate = [string]$ssNode.DateReleased
        $Dev.DriverProvider = [string]$ssNode.Vendor
        $Dev.DriverVersion = [string]$ssNode.Version   # $Dev.DriverVersion comes from Device Manager
        $dev.Solutions.UpdateInfo.IdRef = $ssNode.Id
    }
} # foreach ( $dev in $deviceNodes )

if ( $DeviceCount -gt 0 ) {
    "-- /Devices: Found $DeviceCount matches - Replaced info with new Softpaq $([string]$ssNode.Id)"
} else {
    "-- /Devices: No matches found for ReplaceSoftpaq $($ReplaceSoftpaq)"
} # else if ( $DeviceCount -gt 0 )

$xmlContent.Save((Convert-Path $ReferenceFile))
"-- Reference File Updated: '$ReferenceFile'"

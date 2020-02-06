# SGWPoC-ParallelFileCopyTest
CSVリストに従ってパラレルでファイルコピーを実行し実行時間を測定するpythonツール
# Precondition
### Windows Client
- Windows Instance
    - instance size: m5ad.4xlarge
- [python3 for windows](https://www.python.org/downloads/windows/) (for this tool)
- [git for windows ](https://gitforwindows.org/) (Used to install verification tools)
### File Gateway
- instance type: c5.4xlarge
- ebs:
    - rootvol: io1, 240GiB
    - datavol: io1, 16TiB 3000IOPS
### S3 and KMS
- Any one bucket
- KMS CMK for encrypting S3 objects
- IAM Role for File Gateway

# Setting up
## (a) File Gateway
Prerequisite: File gateway has been created and activated.
### Set SMB guest password
```sh
GATEWAY_ARN=<REPLACE_WITH_THE_COLLECT_GW_ARN>
PASSWORD="HogeHoge@"

aws --profile ${PROFILE} storagegateway \
    set-smb-guest-password \
        --gateway-arn ${GATEWAY_ARN} \
        --password ${PASSWORD}
```
## (b) Windows Client
Attention: CLI is assumed to be executed by powershell.
- install [python3](https://www.python.org/downloads/windows/) and [git](https://gitforwindows.org/)
- Add python PATH to environment valiables
```PowerShell
$oldpath=[System.Environment]::GetEnvironmentVariable("Path", "User")
$oldpath += ";$home\AppData\Local\Programs\Python\Python38\"
[System.Environment]::SetEnvironmentVariable("Path", $oldpath, "User")
```
- initialize local ssd drives
```PowerShell
# Check SSD Dirve
get-disk

# Initialize
initialize-disk 1
New-Partition 1 -UseMaximumSize -DriveLetter D
Format-Volume -DriveLetter D -FileSystem NTFS

```
- install this tool
```PowerShell
cd $Home\Desktop
git clone https://github.com/Noppy/ParallelFileCopyTest.git
```
- Mount SMB File share on F: Drive
```PowerShell
#Password is "HogeHoge@"
net use F: "CHANGE-THE-CORRECT-FileGW-SMB-PATH" /user:"CHANGE-CORRECT-FGW-ID\smbguest"
```

# Usage
## Create File Share
Create a new file share so that there is no medatada registered object.
- Execution environment: BASH shell(mac terminal, linux, etc.)
- Operation user: the Storage Gateway administrator user.
```sh
# Set parameters
PROFILE="<REPLACE_PROFILE>"
BUCKETARN="<REPLACE_WITH_THE_CORRECT_BUCKET_ARN>"
ROLEARN="<REPLACE_WITH_THE_CORRECT_IAM_ROLE_ARN_FOR_File_Gateway>"
GATEWAY_ARN="<REPLACE_WITH_THE_COLLECT_GW_ARN>"
CMK_ARN="<REPLACE_WITH_THE_COLLECT_CMK_ARN>"

CLIENT_TOKEN=$(cat /dev/urandom | base64 | fold -w 38 | sed -e 's/[\/\+\=]/0/g' | head -n 1)

# Create a file share
aws --profile ${PROFILE} storagegateway \
    create-smb-file-share \
        --client-token ${CLIENT_TOKEN} \
        --gateway-arn "${GATEWAY_ARN}" \
        --location-arn "${BUCKETARN}" \
        --role "${ROLEARN}" \
        --object-acl bucket-owner-full-control \
        --default-storage-class S3_STANDARD \
        --guess-mime-type-enabled \
        --authentication GuestAccess \
        --kms-encrypted \
        --kms-key ${CMK_ARN} ;
```
## Execute this tool
```powershell
cd $Home\Desktop\ParallelFileCopyTest
python.exe .\FileCopyTest.py --PreReadDir F:/20150101 .\copy_list.csv
```
- This program behavior
    - First, register objects in metadata by recursively getting the file list of the specified directory(F:/20150101).
    - Execute one process per line of copy_list.csv. Since CSV file has 50 lines, 50 processes are started
    - In the first to tenth lines of "copy_list.csv", objects of the folder for which metadata has been registered is read.(Directory: "F:/20150101") From the 11th line onwards, read objects that are not registered in metadata.

## Execution result
The execution result is recorded in the following file.
- ResultsSummary.csv : A summary of the results.
- ResultsDetail__XXXX_YYYYMMDD_HHMMSS.csv : Detailed results. "XXXX" part of the file name is the number of simultaneous executions.

### Field : ResultsSummary.csv
- Number of tasks
- Number of simultaneous executions
- Overall execution time(sec)
- Start Time
- Finish time
- Number of successful tasks
- Number of failed tasks
- Number of tasks with unknown results
- Total number of tasks 

### Field : ResultsDetail__XXXX_YYYYMMDD_HHMMSS.csv
- First read file path
- Second read file path
- Destination path
- Task start time
- Time to start copying the 1st file based on the test program (parent program) start time (msec)
- Time to finish copying the 1st file based on the test program (parent program) start time (msec)
- Time to finish copying the 2nd file based on the test program (parent program) start time (msec)
- Result of task(Success or Failed)
- Error message when task becomes Failed

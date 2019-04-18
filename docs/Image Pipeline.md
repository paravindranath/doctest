Image Pipeline

Status

NOT STARTED

Impact

LOW

Driver

[Stefania Bruschi](https://atrihub.atlassian.net/wiki/display/~bruschi),  [Hongmei Qiu](https://atrihub.atlassian.net/wiki/display/~hongmeiq)

Approver

[Gustavo Jimenez-Maggiora](https://atrihub.atlassian.net/wiki/display/~gustavoj)

Contributors

[Stefania Bruschi](https://atrihub.atlassian.net/wiki/display/~bruschi),  [Hongmei Qiu](https://atrihub.atlassian.net/wiki/display/~hongmeiq),  [Jia-Shing So](https://atrihub.atlassian.net/wiki/display/~jiashins)

Informed

[Pradeep Ravindranath](https://atrihub.atlassian.net/wiki/display/~paravindranath)

Due date

18 Jan 2019

Outcome

  

note: subject and participant are interchangeable, Record ID = subjecteventcrf.id, upload form name = ddcrf.name, upload field name = ddfile.name

# Background

## Objective

Design and create a pipeline that synchronizes image files between two s3 buckets: EDC source bucket and image bucket. Notify all stakeholders when new image files are available in the image bucket 'quarantine' folder. Track the transaction logs in the EDC database.

The pipeline should be triggered manually on a specific file or set of files.

### EDC source bucket

Image file are uploaded by the EDC system users via the file uploader widget and stored in the EDC source bucket.

-   bucket name: atri-edc-trc-production-_[aws account ID]_
-   source folder: subjecteventcrffile
-   file path: subjecteventcrffile/_[subject.id]_/_[subjecteventcrf.id]_/_[ddfile.name]_/
-   file name: file version code + file extension

### Image destination bucket

Image files in the destination bucket will be made available to collaborators (e.g. labs) for further process.

-   bucket name: atri-edc-trc-production-img-_[aws account ID]_
-   destination folder: quarantine
-   file path: quarantine/[ddcrf.name][ddfile.name]/
-   file name: [ddcrf.name]_[ddfile.name]_[participant code]_[event code]_[subjecteventcrf.id]_[subjecteventcrffile.revisionnumber]_[edcpipelinefile.id]_[timestamp] + file extension  
    -   note: strip out any whitespace
    -   timestamp
        -   Q: timestamp when file is uploaded to source bucket? destination bucket?
            -   when the file is first uploaded into the quarantine folder (edcpipelinefile.ts_create)  
                
        -   format: YYYYMMDD_HHMMSS

  

# Brainstorming

## Design

### Flow

**Q: should we suggest to add an unquarantine folder? to keep quarantine as read only for users? (depends on whether the SFTP wrapper can handle folder level permissions)**

### Track

Track the following information in the EDC database:

-   subjecteventcrffile.id
-   filepath - file path in the destination s3 bucket
-   filename - file name in the destination s3 bucket
-   FileID - unqiue ID for each image file transferred to the destination folder
    -   **Q: can we use SubjectEventCrfFile.id as the FileID?**
-   studyUID**:** DICOM HEADER (0020,000D) Unique identifier for the Study.
-   serieUID**:** DICOM HEADER (0020,000E) Unique identifier of the Series.
-   serienum**:** DICOM HEADER (0020,0011) A number that identifies this Series.
-   ScanID -  a unique combination of studyUID and serieUID for each fileID
-   ATRIUID - Uniqeue ID store in DICOM header to identify the image set: [fileID].[scanID]

Track the following information in a log file (s3 bucket -  **where?**):

-   execution error
-   execution result

### Notification

#### Type of users

-   pipeline user: labs who reviews and process the image files when they are available in the quarantine folder
-   admin user: developers of the image pipeline

#### Type of notifications

-   for pipeline users: a digest report runs [nightly|configurable schedule]  
    -   create a mailing list:  [trc-images-l@atrihub.io](mailto:trc-images-l@atrihub.io)
    -   report includes a list of new files made available to the quarantine folder
    -   report includes count of new files, count of files failed to be transferred/processed
-   for admin users: error logs (immediately after execution), success logs (**immediately? on a schedule?**)
    -   create a mailing list:  [trc-images-admin-l@atrihub.io](mailto:trc-images-admin-l@atrihub.io)
-   Q: do people need to be notified when new files uploaded to the quarantine folder?

## Implementation

### (AWS) EDC RDS read replica

-   Create a read only replica for the EDC database, so the pipeline can find subject, crf information during the transfer  
    -   **Q: what's the delay between real database and the replica? (trigger wait time depends on this response)**

### Database

File information tracked in the database tables.

#### Pipeline.EDCPipeline  

register each pipeline (e.g. pipeline for image file, pipeline for audio files, etc.

Field

  

ID

  

Code

  

Label

  

DDFiles

  

  

#### pipeline.EDCPipelineFile

|  |  |
|--|--|
|  |  |

Field Name

  

ID

  

EDCPipeline.id

  

SubjectEventCrfFile.id

  

file_name

  

Status

Started, In Progress, Completed, Failed

has_error

T/F

####   

#### pipeline.EDCPipelineFileDicom

Field Name

ID

EDCPipelineFile.id

ATRIUID

studyUID

serieUID

serienum

number_of_instances

### API

/pipeline/edcpipeline/file

  

    input: { pipeline_code: xxx subject_event_crf_file_id: xxx *edc_pipieline_file_id: xxx (required for update) } output: { success: T/F error_code: reference dictionary (only show if success is F) error: xxxx (only show if success is F) data: {"edc_pipeline_file: object} }
    

  

/pipeline/edcpipeline/file/dicom/add_scan_info

-   post
    -   request  body (json)

  

`input:`

`{`

`pipeline_code: xxx,`

`subject_event_crf_file_id: xxx,`

`edc_pipieline_file_id: xxxx`

`scans:[`

`{`

`study_uid: xxx,`

`series: [`

`{series_uid: xxx, series_number: yyy, number_of_instance: 123},`

`{series_uid: blah, series_number: halb, number_of_instance: 123},`

`...`

`]`

`},`

`{`

`study_uid: xxx,`

`series: [`

`{series_uid: xxx, series_number: yyy, number_of_instance: 123},`

`{series_uid: blah, series_number: halb, number_of_instance: 123}, ...`

`]`

`}`

`}`

  

`output:`

`{`

`"success"``:` `true``,`

`"error_code"``: reference dictionary (only show` `if`  `success is F)`

`"error"``: xxxx (only show` `if`  `success is F)`

`"data"``:`

`{`

`edc_pipeline_file_id: 123,`

`pipeline_code: xxx,`

`subject_event_crf_file_id: xxx,`

`scans: [`

`{`

`"study_uid"``: xxx,`

`"series"``: [`

`{``"series_uid"``: aaa,` `"atri_uid"``: yyy},`

`{``"series_uid"``: aaa,` `"atri_uid"``: ddd},`

`...`

`]`

`},`

`{`

`"studyuid"``: xxx,`

`"series"``: [`

`{``"series_uid"``: aaa,` `"atri_uid"``: yyy},`

`{``"series_uid"``: aaa,` `"atri_uid"``: ddd},`

`}`

`]`

`}`

`}`

  

### Mailing List

Study specific

-   [trc-images-l@atrihub.io](mailto:trc-images-l@atrihub.io)  - for users (e.g. labs)
-   [trc-images-admin-l@atrihub.io](mailto:trc-images-admin-l@atrihub.io)  - for admins

### (AWS) S3 bucket - temporary

-   create a temporary s3 bucket used by the step function for file processing
-   bucket name: atri-edc-trc-production-img-tmp-[AWS account ID]

### (AWS) Step Function

-   triggered when a new file is uploaded / updated in S3 source bucket folder "subjecteventcrffile"
-   Job name convention: [subject.id]_[subjecteventcrf.id]_[ddfile.name]_[file version code]_[timestamp]

![](https://atrihub.atlassian.net/wiki/download/thumbnails/920256553/Screen%20Shot%202019-01-17%20at%201.41.22%20PM.png?version=1&modificationDate=1547761392073&cacheVersion=1&api=v2&width=275&height=250)

### (AWS) Lambda Function

-   QUERY READ REPLICA:
    -   L1 - query file info from the read replica  
        -   input: file path and file name from source bucket
        -   output: file metadata json
            -   if file is found  
                -   ddcrf.name
                -   ddfile.name
                -   participant code
                -   event code
                -   subjecteventcrf.id
                -   subjecteventcrffile.revisionnumber
                -   subjecteventcrffile.id
                -   pipeline_code
            -   if file is not found
                -   return ERROR
        -   step fn - retry function, checks return status of L1 (expect raised ERROR and try for max attempt)
        -   duration:15 minutes
        -   frequency: every 30 sec
-   Process DCM:
    -   **Notes:**
        -   All the following functions process the same file.
        -   Size may be a restriction for lambda.
        -   AWS batch can an be an alternate solution.
    -   L5 - parse file and extract the DICOM header
        -   input: file path and name
        -   output: DICOM header info for each scan (list)
            -   studyUID
            -   serieUID
            -   serienum
    -   L7 - write ATRIUID in DICOM header
    -   L4 - copy processed file from s3 source bucket to s3 temp bucket, rename the file
        -   input:
            -   source file path and name
            -   target file path and name
        -   output: successful (True/False)
-   ADDITIONAL PROCESS:
    -   L8 - additional process (such as LONI process)
-   WRITE META DATA:
    -   L6 - call API to write file metadata in study DB (set file status to "in progress")
        -   input: L1 and L5 output
        -   output: successful (True/False)
-   COPY TO DESTINATION:
    -   L9 - copy file from S3 temp bucket to S3 destination bucket
        -   input:
            -   s3 temp bucket
            -   s3 destination bucket
            -   file path
            -   file name
        -   output: successful (True/False)
-   UPDATE STATUS:
    -   L10 - call API to update file status in study DB to "done"
-   ErrorSNS
    -   L3 - email/log error for Admin
        -   input: error
        -   output: "status : error"
-   PROCESS COMPLETION SNS:
    -   L11 - email / log to Admin on summary or error
        -   if error: call API to update file status in study DB to "error"
        -   input: status
        -   output: successful (True/False)
-   USER DIGEST (This has to be post pipeline processing - separate Lambda not part of this step fn):
    -   L12 - (user digest) send batch summary to users
        -   input: duration (last 24 hours etc.)
        -   output: successful (True/False)

### (AWS) Batch  [![](https://atrihub.atlassian.net/secure/viewavatar?size=xsmall&avatarId=11000&avatarType=issuetype)PIPE-4](https://atrihub.atlassian.net/browse/EDCAWS-93?src=confmacro)  IN FEATURE BRANCH

  

## Error Logs

Pipeline errors will be logged in the S3 bucket

-   bucket name: atri-edc-trc-production-_[aws account ID]_
-   source folder: image_pipelines
-   file path: image_pipelines/error_logs/[edcpipeline.code]/
-   file name: (matching closely with destination file name under quarantined folder)  
    -   [[ddcrf.name](http://ddcrf.name/)]_[[ddfile.name](http://ddfile.name/)]_[participant code]_[event code]_[[subjecteventcrf.id](http://subjecteventcrf.id/)]_[subjecteventcrffile.revisionnumber]_[[edcpipelinefile.id](http://edcpipelinefile.id/)]_[timestamp]_error_log.txt

### Error Flags in the Database

scenario 1: edcpipelinefile.status = completed, but edcpipelinefile.has_error = True

-   File is processed and is available in destination bucket quarantined folder, but some errors were generated during the process
    -   e.g. can't insert ATRIUID into the DICOM header etc.

scenario 2: edcpipelinefile.status = Failed

-   File is not available in the destination bucket quarantined folder, something more seriously wrong during the process

### Error Codes and their meaning

Please refer to  [Pipeline Dictionary](https://atrihub.atlassian.net/wiki/spaces/AWSPIPE/pages/927367226/Pipeline+Dictionary)

Q: should I promote error code to database table column? e.g. edcpipelinefileerrorlog.error_code

### Error Notification Email

![](https://atrihub.atlassian.net/wiki/download/thumbnails/920256553/image2019-3-22_0-29-26.png?version=1&modificationDate=1553239768323&cacheVersion=1&api=v2&width=580&height=400)

# Relevant Data

## ATRIUID / Batch script logic

![](https://atrihub.atlassian.net/wiki/download/thumbnails/920256553/IMG_1369.JPG?version=1&modificationDate=1549077947704&cacheVersion=1&api=v2&width=533&height=400)

## Brainstorming Drawing Board

![](https://atrihub.atlassian.net/wiki/download/thumbnails/920256553/IMG_1352.JPG?version=1&modificationDate=1547499937854&cacheVersion=1&api=v2&width=533&height=400)

[![](https://atrihub.atlassian.net/wiki/s/114275273/6452/6e72b6ef919f435036c01f9329c3844398ba52f1/1000.0.6/_/download/resources/com.atlassian.confluence.plugins.confluence-view-file-macro:view-file-macro-resources/images/placeholder-large-file.png)IMG_6233.HEIC](https://atrihub.atlassian.net/wiki/download/attachments/920256553/IMG_6233.HEIC?version=1&modificationDate=1547519251926&cacheVersion=1&api=v2)

  

## TRC Amyloid PET Scan & Data Flow

[![](https://api.media.atlassian.com/file/a01818e9-f241-413d-9e4a-58f8746242ca/image?height=320&client=2c443d3b-214d-4735-8807-9d540fa17755&token=eyJhbGciOiJIUzI1NiJ9.eyJpc3MiOiIyYzQ0M2QzYi0yMTRkLTQ3MzUtODgwNy05ZDU0MGZhMTc3NTUiLCJhY2Nlc3MiOnsidXJuOmZpbGVzdG9yZTpmaWxlOmEwMTgxOGU5LWYyNDEtNDEzZC05ZTRhLTU4Zjg3NDYyNDJjYSI6WyJyZWFkIl19LCJleHAiOjE1NTU2MTE0ODYsIm5iZiI6MTU1NTYwODEyNn0.DkS_0J9yNZYtgBB_ysp7Pwzi7zoiLny-6o-8LyfyLVI)](https://atrihub.atlassian.net/wiki/download/attachments/920256553/TRC_PET_Data_Flow.pdf?version=1&modificationDate=1547497581575&cacheVersion=1&api=v2)[Edit file](https://atrihub.atlassian.net/wiki/download/attachments/920256553/TRC_PET_Data_Flow.pdf?version=1&modificationDate=1547497581575&cacheVersion=1&api=v2)
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTI3OTQ2MTI4OCwxMzIxOTc2MjI2XX0=
-->
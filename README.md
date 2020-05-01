# Spool Tool

Helper script for uploading files from a spool directory into S3,
typically used with a data ingestion process. It runs in two phases:

1. Upload Phase. Your program places files into one or more
   SPOOL_DIRs. `spool_tool` looks for eligible files (see below for
   what that means) and uploads them. It then moves the files into the
   completed directory if the upload succeeds. If not, it leaves the
   files for the next run.

2. Cleanup Phase. Deletes files in the completed directories.

## Eligible Uploads

Your program places new files into `SPOOL_DIR/new`. Files are eligible
for uploading if they are regular files (ie, not directories or
symlinks), and have not been modified for HOLD_FOR minutes. By
default, HOLD_FOR is set to 0, which means files are immediately
eligible for upload. Ideally your ingestion process places new files
atomically, but this might not always be the case, so setting a
reasonable hold period will prevent `spool_tool` from uploading a
partial file.

## Eligible Completed Deletions

Similar to the upload holding period, `spool_tool` will not delete
completed files that are newer than DELETE_AFTER minutes, based on
last modified time. By default this is set to one hour, but you should
pick an appropriate "just in case" period based off available disk
space and rate of ingestion.

The completed directory is `SPOOL_DIR/completed` and will be created
if it doesn't already exist.

## Destinations

`spool_tool` supports a noop destination for debugging, and an S3
destination for production. The S3 destination uses boto3, so the
normal search strategy for AWS credentials applies.

The S3 URI format is as follows:
s3://BUCKET/PATH/TO/DIRECTORY. `spool_tool` forces the supplied URI to
be a directory path. If you want to prefix the files with a string,
use the `--format` option.

## Filename Formatting

Hadoop and Hive based systems expect a directory structured around
partitions, which look like specially formatted paths. For instance, a
typical directory structure might look like:

```
year=2020/month=04/day=30/data-file-1.json
year=2020/month=04/day=30/data-file-2.json
year=2020/month=05/day=01/data-file-1.json
year=2020/month=05/day=01/data-file-2.json
```

`spool_tool` allows you to specify a formatting string to make this
easier. Suppose your ingestion program outputs `data-file-1.json`,
`data-file-2.json`, etc. You could specify a format string of
`year={m_YYYY}/month={m_MM}/day={m_DD}/{fname}`, and `spool_tool` will
substitute the modified time year, month, and day strings as
appropriate to produce the above directory listings. We support the
following substitions:

- m_YYYY, m_MM, m_DD, m_hh, m_mm, m_ss - modified year, month, day,
  hour, minute, and second respectively, UTC

- c_YYYY, c_MM, c_DD, c_hh, c_mm, c_ss - created year, month, day,
  hour, minute, and second respectively, UTC

- uuid1, uuid4 - randomly generated uuids (uuid1 is derived from host
  and time, uuid4 is totally random)
  
- fname - filename

- now - Unix timestamp in integer seconds

- spool_dir - Slash stripped SPOOL_DIR for the current file

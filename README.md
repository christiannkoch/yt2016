# YouTube Video Metadata Collection (Nov/Dez 2016)

- [Introduction](#introduction)
- [Using the dump](#using-the-dump)
    - [Import dump to MongoDB](#1-import-dump-to-mongodb)
    - [Parse JSON files](#2-parse-json-files)
- [Collections](#collections)
    - [jobs](#jobs)
    - [yt-metadata](#yt-metadata)

## Introduction
This repository holds a MongoDB dump consisting of the metadata of ~1,000,000 YouTube videos. The dataset was collected by issuing random search queries of the following form to YouTube: `inurl:"watch?v=[A-Z]{5}`. The collection process was conducted during November and December 2016.

## Using the dump
There are two possibilities to use the dump

1. Import to a MongoDB instance
2. Parse the .json files

### 1) Import dump to MongoDB
The dump in this repository (`mongodb.dump.tar.gz`) was exported from a MongoDB database (v2.6.10) via the mongodump command. To import the dump, extract the `.tar.gz` file in an arbitrary folder, e.g., to `home/user` -- the dump files will be extracted in a subfolder called `dump`. To import the dumped collections, issue the following command in the directory that contains the `dump` folder:

```$ mongorestore --db youtube```

After importing, the database should be accessible via `mongo youtube`.

### 2) Parse JSON files
For convinience, the dump was additionaly exported via mongoexport in JSON. Each collection (`jobs` and `yt-metadata`) was exported in a separate file (`ytmetadata.noid.json.gz` and `jobs.noid.json.gz`) and each line in a file represents a single document in the collection. The `*.sample.json` files contain the first 500 documents in an uncompressed format.

## Collections
The database consists of two important collections:
- jobs
- yt-metadata

### jobs
The jobs collection was used in the pre-collection-phase of the video metadata. In this phase, random video ID prefixes consisting of 5 characters (`[a-z]{5}`) were generated and saved as a "job". These jobs were processed by a worker who used the YouTube search engine to submit queries in the form `inurl:"watch?v=[A-Z]{5}`. The results of these queries were appended to the job.

**Example document:**
- `"prefix" : "wxmdw"`<br>
  This was the video ID prefix that was used to generate the search query. Any videos found by the query are expected to have a video ID that starts with this prefix (case-insensitive).
- `"status" : 0`
  - Status **0**: Job was processed, at least 1 video was found
  - Status **10**: Job is pending, await processing
  - Status **20**: Too many results. This status would have been used if a search yields more than 20 results for a given prefix. However, this never happend during the collection process
  - Status **21**: The search yielded no result for the submitted query.
- `"results" : [ { "request_language" : "zh-TW", "content_language" : "TW", "via" : "tcp://60.199.198.24:8080", "videoId" : "wXMdW-Mb9k0", "inserted_at" : 1479411573 }, { "request_language" : "zh-TW", "content_language" : "TW", "via" : "tcp://60.199.198.24:8080", "videoId" : "wxmDw-q4Bjk", "inserted_at" : 1479411573 } ]`<br>
  Array of search results. In addition to the video ID of a particular entry in the list of results, it contains the proxy address via which the search query was performed, the request and content language that YouTube responded with and the unix timestamp at which this entry was inserted to the database (which corresponds to the time at which the search was performed)<br>
  Because prefixes were generated in a random fashion, some of them were generated multiple times but saved as a separate job. These jobs were later merged (i.e. the results of those jobs with a common "prefix" were aggregated into a single job). Therefore it is not reliable to count the elements in the "results" array to determine the number of search results! Furthermore, it is possible that the amount of search results for a given prefix vary, depending on the proxy that was used to submit the query. This is because some videos are geo-blocked by YouTube in specific countries. To reliably count the search results for a given prefix, only count those elements in the array that share the same inserted_at timestamp.
- `"created_at" : 1479382240`<br>
  The time at which this document was inserted to the database
- `"updated_at" : 1479411573`
  The last time that this document was updated
- `"generated_at" : 1479411509`
  This was used internaly

### yt-metadata
Once enough jobs were generated and processed, the collection-phase was initiated. In this phase, for each prefix in the job collection for which at least a single video could be found (status 0), a random video ID was selected from the list of search results.
For these videos (1,043,429), the metadata were crawled once and statistics of were observed throughout a period of 50 days on a daily basis. For this task, the YouTube API was used.

**Example document:**
- `"videoid" : "yoftU-xA4Fw"`<br>
  The video ID 
- `"parts" : "id,statistics"`<br>
  This was populated by a scheduler to indicate which parts the worker should crawl, the next time it requests the video metadata from the YouTube API
- `"status" : 0`<br>
  This is the status of the worker that was replied after fetching the metadata of this video.
  - Status **0**: OK. The YouTube API successfuly replied with the metadata for this video.
  - Status **10**: PENDING. This status was used by the scheduler to indicate that the last processing time of the video was ~24 hours ago and the worker needs to recrawl the statistics of this video.
  - Status **20**: NOT FOUND. The YouTube API did not replied with any metadata for this video. Once in this status, the worker did not tried any further attempts to collect metadata for this video.
- `"results" : [ { "contentDetails" : { "duration" : "PT42S", "dimension" : "2d", "definition" : "hd", "caption" : "false", "licensedContent" : "0", "projection" : "rectangular" }, "id" : "yoftU-xA4Fw", "snippet" : { "publishedAt" : "2015-08-26T23:00:56.000Z", "channelId" : "UCJ6u6QrV4O4aJP7JYF19ejw", "title" : "Rew. Orange Circles (Real)", "channelTitle" : "HappierChunk7", "categoryId" : "20", "liveBroadcastContent" : "none" }, "statistics" : { "viewCount" : "10", "likeCount" : "1", "dislikeCount" : "0", "favoriteCount" : "0", "commentCount" : "0" }, "status" : { "uploadStatus" : "processed", "privacyStatus" : "public", "license" : "youtube", "embeddable" : "1", "publicStatsViewable" : "1" }, "inserted_at" : 1482086167 }, ... ]`<br>
  This contains the reported data from the YouTube API. For the first crawl crawl, both static and dynamic video metadata was captured. For successive crawls only the dynamic (= statistics) video metadata was crawled.
- `"updated_at" : 1486318262`<br>
  The last time this document was updated
- `"created_at" : 1481217193`<br>
  The time this document was created
- `"reserved_at" : 1486318262`<br>
  Used internaly. The last time that the worker reserved this document

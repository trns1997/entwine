# Configuration

Entwine provides 4 sub-commands for indexing point cloud data:

| Command             | Description                                             |
|---------------------|---------------------------------------------------------|
| [build](#build)     | Generate an EPT dataset from point cloud data           |
| [info](#info)       | Gather information about point clouds before building   |
| [merge](#merge)     | Merge datasets build as subsets                         |

These commands are invoked via the command line as:

    entwine <command> <arguments>

Although most options to entwine commands are configurable via command line,
each command accepts configuration via [JSON](https://www.json.org/).  A
configuration file may be specified with the `-c` command line argument.

Command line argument settings are applied in order, so earlier settings can be
overwritten by later-specified settings.  This includes configuration file
arguments, allowing them to be used as templates for common settings that may
be overwritten with command line options.

Internally, Entwine CLI invocation builds a JSON configuration that is passed
along to the corresponding sub-command, so CLI arguments and their equivalent
JSON configuration formats will be described for each command.  For example,
with configuration file `config.json`:

```json
{
    "input": "~/data/chicago.laz",
    "output": "~/entwine/chicago"
}
```

The following Entwine invocations are equivalent:

```
entwine build -i ~/data/chicago.laz -o ~/entwine/chicago
entwine build -c config.json
```

Throughout Entwine, a wide variety of point cloud formats are supported as input
data, as any [PDAL-readable](https://pdal.io/stages/readers.html) format may be
indexed.  Paths are not required to be local filesystem paths - they may be
local, S3, GCS, Dropbox, or any other
[Arbiter-readable](https://github.com/connormanning/arbiter) format.

Each command accepts some common options, detailed at [common](#common).



## Build

The `build` command generates an [Entwine Point Tile](entwine-point-tile.md) (EPT) dataset from point cloud data.

```bash
entwine build (<options>)
```

## Options

| Key | Description |
|-----|-------------|
| [input](#input) | Input file(s) or directories to include in the build |
| [output](#output) | Output directory for the resulting EPT dataset |
| [config](#config) | Optional configuration file for templating common options |
| [tmp](#tmp) | Directory for temporary files |
| [srs](#srs) | Set the SRS metadata entry of the output |
| [reprojection](#reprojection) | Reproject input data to a different SRS |
| [hammer](#hammer) | Force use of user-supplied input SRS, overriding file headers |
| [threads](#threads) | Number of parallel threads |
| [force](#force) | Overwrite an existing build instead of continuing it |
| [dataType](#datatype) | Data encoding type for serialized output (`laszip`, `zstandard`, `binary`) |
| [span](#span) | Number of voxels in each spatial dimension for data nodes |
| [noOriginId](#nooriginid) | Disable OriginId tracking for point source files |
| [bounds](#bounds) | Explicit spatial bounds for filtering points |
| [deep](#deep) | Force full file reads during analysis instead of header-only reads |
| [absolute](#absolute) | Use absolute double-precision XYZ values instead of scaled integers |
| [scale](#scale) | Set coordinate scale factor |
| [limit](#limit) | Limit number of files to insert in this build session |
| [subset](#subset) | Specify a portion of a parallel/subset build |
| [maxNodeSize](#maxnodesize) | Maximum number of points in a node before overflow |
| [minNodeSize](#minnodesize) | Minimum number of overflowed points before new node creation |
| [cacheSize](#cachesize) | Number of nodes cached in memory before serialization |
| [hierarchyStep](#hierarchystep) | Step size for hierarchy file splitting (testing only) |
| [sleepCount](#sleepcount) | Count per thread after which idle nodes are serialized |
| [progress](#progress) | Interval (seconds) for progress logging (0 disables) |
| [laz_14](#laz_14) | Write LAZ 1.4 content encoding |
| [profile](#profile) | AWS CLI profile name for S3 access |
| [sse](#sse) | Enable AWS server-side encryption |
| [requester-pays](#requester-pays) | Enable AWS S3 requester-pays flag |
| [allow-instance-profile](#allow-instance-profile) | Allow EC2 instance profile credentials for S3 access |

### input

The point cloud data paths to be indexed.  This may be a string, as in:
```json
{ "input": "~/data/autzen.laz" }
```

This string may be:
- a file path: `~/data/autzen.laz` or `s3://entwine.io/sample-data/red-rocks.laz`
- a directory (non-recursive): `~/data` or `~/data/*`
- a recursive directory: `~/data/**`
- an info directory path: `~/entwine/info/autzen-files/`
- an info output file: `~/entwine/info/autzen-files/1.json`

This field may also be a JSON array of multiples of each of the above strings:
```json
{ "input": ["autzen.laz", "~/data/"] }
```

Paths that do not contain PDAL-readable file extensions will be silently
ignored.

### output

A directory for Entwine to write its EPT output.  May be local or remote.

### config

Path to a JSON configuration file for templating common parameters.  
Command-line arguments override configuration file values.

```bash
--config template.json -i in.laz -o out
```

### tmp

A local directory for Entwine's temporary data.

```bash
--tmp /tmp/entwine
```

### srs

Specification for the output coordinate system.  Setting this value does not
invoke a reprojection, it simply sets the `srs` field in the resulting EPT
metadata.

If input files have coordinate systems specified (and they all match), then this
will typically be inferred from the files themselves.

### reprojection

Coordinate system reprojection specification.  Specified as a JSON object with
up to 3 keys.

If only the output projection is specified, then the input coordinate system
will be inferred from the file headers.  If no coordinate system information
can be found for a given file, then this file will not be inserted.

```bash
--reprojection EPSG:3857
--reprojection EPSG:26915 EPSG:3857
```

JSON form:

```json
{ "reprojection": { "in": "EPSG:26915", "out": "EPSG:3857" } }
```

An input SRS may also be specified, which will be overridden by SRS information
determined from file headers.
```json
{
    "reprojection": {
        "in": "EPSG:26915",
        "out": "EPSG:3857"
    }
}
```

To force an input SRS that overrides any file header information, the `hammer`
key should be set to `true`.
```json
{
    "reprojection": {
        "in": "EPSG:26915",
        "out": "EPSG:3857" ,
        "hammer": true
    }
}
```

When using this option, the `output` value will be set as the coordinate system
in the resulting EPT metadata, so the `srs` option does not need to be
specified.

### threads

Number of threads for parallelization.  By default, a third of these threads
will be allocated to point insertion and the rest will perform serialization
work.

```bash
--threads 12
```

```json
{ "threads": 9 }
```

This field may also be an array of two numbers explicitly setting the number of
worker threads and serialization threads, with the worker threads specified
first.
```json
{ "threads": [2, 7] }
```

### force

By default, if an Entwine index already exists at the `output` path, any new
files from the `input` will be added to the existing index.  To force a new
index instead, this field may be set to `true`.

```bash
--force
```

```json
{ "force": true }
```

### dataType

Specification for the output storage type for point cloud data.  Currently
acceptable values are `laszip`, `zstandard`, and `binary`.  For a `binary`
selection, data is laid out according to the [schema](#schema).  Zstandard
data consists of binary data according to the [schema](#schema) that is then
compressed with [Zstandard](https://facebook.github.io/zstd/) compression.

```bash
--dataType laszip
```

```json
{ "dataType": "laszip" }
```

### span

Number of voxels in each spatial dimension which defines the grid size of the
octree.  For example, a `span` value of `256` results in a `256 * 256 * 256`
cubic resolution.

```bash
--span 128
```

### noOriginId

Disable insertion of the `OriginId` dimension, which tracks the original source file for each point.

```bash
--noOriginId
```

### bounds

Total bounds for all points to be index.  These bounds are final, in that they
may not be expanded later after indexing has begun.  Typically this field does
not need to be supplied as it will be inferred from the data itself.  This field
is specified as an array of the format `[xmin, ymin, zmin, xmax, ymax, zmax]`.

```bash
--bounds 0 0 0 100 100 100
--bounds "[0,0,0,100,100,100]"
```

```json
{ "bounds": [0, 500, 30, 800, 1300, 50] }
```

### deep

By default, file headers for point cloud formats that contain information like
number of points and bounds are considered trustworthy.  If file headers are
known to be incorrect, this value can be set to `true` to require a deep scan
of all the points in each file.

### absolute

Scaled values at a fixed precision are preferred by Entwine (and required for
the `laszip` [dataType](#dataType)).  To use absolute double-precision values
for XYZ instead, this value may be set to `true`.

### scale

A scale factor for the spatial coordinates of the output.  An offset will be
determined automatically.  May be a number like `0.01`, or a 3-length array of
numbers for non-uniform scaling.

```bash
--scale 0.1
--scale "[0.1, 0.1, 0.025]"
```

```json
{ "scale": 0.01 }
```
```json
{ "scale": [0.01, 0.01, 0.025] }
```

### limit

If a build should not run to completion of all input files, a `limit` may be
specified to run a fixed maximum number of files.  The build may be continued
by providing the same `output` value to a later build.

```bash
--limit 20
```

```json
{ "limit": 25 }
```

### subset

Entwine builds may be split into multiple subset tasks, and then be merged later
with the [merge](#merge) command.  Subset builds must contain exactly the same
configuration aside from this `subset` field.

Subsets are specified with a 1-based `id` for the task ID and an `of` key for
the total number of tasks.  The total number of tasks must be a power of 4.

```bash
--subset 1 4
```

```json
{ "subset": { "id": 1, "of": 16 } }
```

### maxNodeSize

A soft limit on the maximum number of points that may be stored in a data node.
This limit is only applicable to points that are "overflow" for a node - so
points that fit natively in the `span * span * span` grid can grow beyond this
size.

### minNodeSize

A limit on the minimum number of points that may reside in a dedicated node.
For would-be nodes containing less than this number, they will be grouped in
with their parent node.

### cacheSize

When data nodes have not been touched recently during point insertion, they are
eligible for serialization.  This parameter specifies the number of unused
nodes that may be held in memory before serialization, so that if they are used
again soon enough they won't need to be serialized and then reawakened from
remote storage.

### hierarchyStep

For large datasets with lots of data files, the
[hierarchy](entwine-point-tile.md#ept-hierarchy)
describing the octree layout is split up to avoid large downloads.  This value
describes the depth modulo at which hierarchy files are split up into child
files.  In general, this should be set only for testing purposes as Entwine will
heuristically determine a value if the output hierarchy is large enough to
warrant splitting.

### sleepCount

Serialization frequency for idle nodes (per-thread count before flushing).

### progress

Progress logging interval in seconds.  
Set to `0` to disable (default: `10`).

### laz_14

By default, laszip encoded output will be written as LAS 1.2.  Set `laz_14` to
`true` to write 1.4 data instead.

### profile

Specify an AWS CLI profile to use for S3 access.

```bash
--profile john
```

### sse

Enable AWS Server-Side Encryption (SSE) for S3 writes.

### requester-pays

Enable S3 requester-pays mode.

### allow-instance-profile

Allow EC2 instance profile credentials for S3 access.


## Info

The `info` command is used to aggregate information about unindexed point cloud
data prior to building an Entwine Point Tile dataset.

Most options here are common to `build` and perform exactly the same function in
the `info` command, aside from `output`, described below.

| Key | Description |
|-----|-------------|
| [input](#input) | Path(s) to build |
| [output](#output-info) | Output directory |
| [tmp](#tmp) | Temporary directory |
| [srs](#srs) | Output coordinate system |
| [reprojection](#reprojection) | Coordinate system reprojection |
| [threads](#threads) | Number of parallel threads |
| [deep](#deep) | Specify whether file headers are trustworthy |

### output (info)

The `output` is a directory path to write detailed per-file metadata.  This
directory may then be used as the `input` for a [build](#build) command.



## Merge

The `merge` command is used to combine [subset](#subset) builds into a full
Entwine Point Tile dataset.  All subsets must be completed.

*Note*: This command is **not** used to merge unrelated EPT datasets.


| Key | Description |
|-----|-------------|
| [output](#output-merge) | Output directory of subsets |
| [tmp](#tmp) | Temporary directory |
| [threads](#threads) | Number of parallel threads |

### output (merge)

The output path must be a directory containing `n` completed subset builds,
where `n` is the `of` value from the subset specification.






## Common

| Key | Description |
|-----|-------------|
| [verbose](#verbose) | Enable verbose output |
| [arbiter](#arbiter) | Remote file access settings for S3, GCS, Dropbox, etc. |

### verbose

Defaults to `false`, and setting to `true` will enable a more verbose output to STDOUT.

### arbiter

This value may be set to an object representing settings for remote file access.  Amazon S3, Google Cloud Storage, and Dropbox settings can be placed here to be passed along to [Arbiter](https://github.com/connormanning/arbiter).  Some examples follow.

Enable Amazon S3 server-side encryption for the default profile:
```json
{ "arbiter": {
    "s3": {
        "sse": true
    }
} }
```

Enable IO between multiple S3 buckets with different authentication settings.  Profiles other than `default` must use prefixed paths of the form `profile@s3://<path>`, for example `second@s3://lidar-data/usa`:
```json
{ "arbiter": {
    "s3": [
        {
            "profile": "default",
            "access": "<access key here>",
            "secret": "<secret key here>"
        },
        {
            "profile": "second",
            "access": "<access key here>",
            "secret": "<secret key here>",
            "region": "eu-central-1",
            "sse": true
        }
    ]
} }
```

Setting the S3 profile is also accessible via command line with `--profile <profile>`, and server-side encryption can be enabled by using `--sse`.

## Miscellaneous

### S3

Entwine can read and write S3 paths.  The simplest way to make use of this
functionality is to install [AWSCLI](https://aws.amazon.com/cli/) and run
`aws configure`, which will write credentials to `~/.aws`.

If you're using Docker, you'll need to map that directory as a volume.
Entwine's Docker container runs as user `root`, so that mapping is as simple as
adding `-v ~/.aws:/root/.aws` to your `docker run` invocation.

### Cesium

Creating 3D Tiles point cloud datasets for display in Cesium is a two-step
process.

First, an Entwine Point Tile datset must be created with an output projection of
earth-centered earth-fixed, i.e. `EPSG:4978`:

```
mkdir ~/entwine
docker run -it -v ~/entwine:/entwine connormanning/entwine build \
    -i https://entwine.io/sample-data/autzen.laz \
    -o /entwine/autzen-ecef \
    -r EPSG:4978
```

Then, `entwine convert` must be run to create a 3D Tiles tileset:

```
docker run -it -v ~/entwine:/entwine connormanning/entwine convert \
    -i /entwine/autzen-ecef \
    -o /entwine/cesium/autzen
```

Statically serve the tileset locally:

```
docker run -it -v ~/entwine/cesium:/var/www -p 8080:8080 \
    connormanning/http-server
```

And browse the tileset with
[Cesium](http://cesium.entwine.io/?url=http://localhost:8080/autzen/tileset.json).


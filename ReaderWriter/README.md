# OpenVectorFormat ReaderWriters

This folder contains tools for easy handling of OpenVectorFormat Jobs in your application.

## How to use?

### In C#

* Unified interfaces used for all readers / writers are provided in the AbstractReaderWriter project.
* For easy usage, a FileReaderWriterFactory library is provided. It offers easy access to all File-Reader and File-Writer implementations.
* If you do not need File-Storage (e.g. for intermediate jobs), the `OVFReaderWriter` offers to classes to comfortably handle a job in-memory, also with the common interface from `AbstractReaderWriter`.

Simple usage examples on how to read, write or convert a file:

```c#
using OpenVectorFormat.FileReaderWriterFactory;
using OpenVectorFormat.AbstractReaderWriter;
```

**Reading a file**
```c#
string filename = "testfile.ovf";
string fileExtension = ".ovf";
if (!FileReaderFactory.SupportedFileFormats.Contains(fileExtension))
{
    throw new Exception("Not supported")
}

// create progress object & file reader for this extension
IFileReaderWriterProgress progress = new FileReaderWriterProgress();
FileReader fileReader = FileReaderFactory.CreateNewReader(fileExtension);

// read job
await fileReader.OpenJobAsync(filename, progress);

// optional - cache the job completely to memory to have fast access
await fileReader.CacheJobToMemoryAsync();
    
// access job
// The jobshell contains the meta-data at the job level and the processing parameters.
// It does NOT contain the geometry data.
Job jobshell = fileReader.JobShell;
WorkPlane wp = await fileReader.GetWorkPlaneAsync(workplane_index); // get a complete workplane with geometry data
WorkPlane wp_shell = fileReader.GetWorkPlaneShell(workplane_index); // get a workplane without geometry data, just metadata
VectorBlock vb = await fileReader.GetVectorBlockAsync(workplane_index, vectorblock_index) // get a complete vectorblock, with geometry data

// ... do what you need

fileReader.Dispose();
```

**Writing a file**
```c#
string filename = "testfile.ovf";
string fileExtension = ".ovf";
if (!FileWriterFactory.SupportedFileFormats.Contains(fileExtension))
{
    throw new Exception("Not supported")
}

Job job = new Job();

// ... fill job with metadata & geometry if already possible - can also be done later on

// create progress object & file reader for this extension
IFileReaderWriterProgress progress = new FileReaderWriterProgress();
FileWriter fileWriter = FileWriterFactory.CreateNewWriter(fileExtension);

// Option 1: Write complete job at once (no streaming)
await fileWriter.SimpleJobWriteAsync(job, filename, progress);
fileWriter.Dispose();

// Option 2: Start partial writing and add data over time
fileWriter.StartWritePartial(job, filename, progress);

// write to job
// The jobshell contains the meta-data at the job level and the processing parameters.
// It does NOT contain the geometry data.
// acces the jobshell to add metadata, processing parametes etc.
fileWriter.JobShell.JobMetaData.Author = "Ano Nym";

// add a workplane - it may already contain vector blocks, but it is not required. more blocks can be added later.
// appending a new workplane finalizes the previous one - it cannot be altered anymore.
await fileWriter.AppendWorkPlaneAsync(workPlane);

// append vectorblock to latest workplane
await fileWriter.AppendVectorBlockAsync(vectorBlock);

// to finish & close the file
fileWriter.Dispose();
```

**To convert from one format to another**
```c#
await FileConverter.ConvertAsync(soruceFile, targetFile, progress); 
```


### Other languages
Use the standalone `FileReaderWriterFactoryGRPCWrapper` (pre-build executable is available on the release page) - it provides the functionality of the `FileReaderWriterFactory` through an gRPC Service and can be easily integrated in most progamming languages. The service definition can be found here: [gRPC Service definition](AbstractReaderWriter/grpc_reader_writer_interface.proto). For more information on gRPC, see here: [gRPC](https://grpc.io/).

To read / write a job all-at-once, call the `SimpleJobRead` or `SimpleJobWrite` procedures.
If the job is very big and you do not want to keep it in memory all the time, use the `PartialJobRead` and `PartialJobWrite` procedures to open a stream and do the reading / writing in parts.

## Supported file formats
### (.ovf) OpenVectorFormat Reference Implementation
We strongly recommend to use this reference implementation whenever possible.

It implements the common [FileReader](AbstractReaderWriter/FileReader.cs) and [FileWriter](AbstractReaderWriter/FileWriter.cs) provided by AbstractReaderWriter.
It offers
* Reading a job from and writing a job to a .ovf file
  * For big jobs, both reading and writing are streamable to minimize the memory requirements of your application
* Automatically takes care of housekeeping tasks such as incrementing the `num_work_planes` variable in a `Job` whenever a new `WorkPlane` is added.

### 3rd party formats / legacy formats
#### (.asp) ASP-geometry files
The ASP-format is a custom, very simple format developed by the Fraunhofer Institute for Laser Technology for use with laser scanning systems.

A reader and writer implemenation are available.
Important points:
* ASP has no support for structuring the geometry data in WorkPlanes or axis positions
  * Reading from an ASP file puts all data into one WorkPlane
  * Writing to an ASP file merges  all WorkPlanes into one, axis positions are lost
  * There is no support for streaming data in the reader, it is stored in memory immediatly after reading
* ASP does not support metadata. 
* Currently, only the following ASP-commands are supported:
  * LP
  * VG
  * VJ
  * ON
  * JP
  * GO

#### (.cli, .ilt)
The Common Layer Interface (CLI) is a universal format for the input of geometry data to model fabrication systems based on layer manufacturing technologies (LMT).

Development of the Common Layer Interface (CLI) originated in the Brite-EuRam Project "Rapid Prototyping Techniques". CLI has no support for any manufacturing parameters.

Documentation can be found here: https://web.archive.org/web/19970522080658/http://www.cranfield.ac.uk/aero/rapid/CLI/cli_v20.html

Integration of SLI has been started, but is not finished. SLI is a propriatary variant of CLI by EOS that additionally has a look up table for layers.

The .ilt format is a custom format developed by the Fraunhofer Institute for Laser Technology for use with laser scanning systems. It uses multiple CLI files and a text file with parameters in a ZIP archive.
It has been adopted by some commercial software, e.g. by Aconity 3D or Autodesk Netfab.

Only a reader is available.
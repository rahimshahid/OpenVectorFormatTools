# OVF File Structure
To enable selective streaming of various elements of a job from the file, the Job protobuf message cannot simply be seriallized and dumped to a file. Instead, the job is seriallized in parts and the position of the parts is indicated by look-up-tables (LUTs).

The LUTs themselves are protobuf messages, the .proto file can be found at [protobuf/ovf_lut.proto](OVFReaderWriter/protobuf/ovf_lut.proto).

All protobuf messages (LUTs, JobShell, WorkPlaneShells, VectorBlocks) are written to the file using the `WriteDelimitedTo` method and can be read using the `ParseDelimitedFrom` method in protobuf libraries.
The writeDelimited methods first read / write the length in bytes of the protobuf message from / to the stream as VarInt32 (defined by protobuf) and then parse the data. They are not available in all language implementations default libraries, reading and writing of the length might have to be implemented manually. Some help to this can be found here: https://stackoverflow.com/questions/2340730/are-there-c-equivalents-for-the-protocol-buffers-delimited-i-o-functions-in-ja/22927149#22927149

## Structure overview

| Length (bytes) | Description | Values | Annotation |
|---|---|---|---|
| 4      | Magic number | [0x4c, 0x56, 0x46, 0x21] |
| 8     | Position of Job LUT in File (Int64, LittleEndian)      |   |
| 8 | Position of LUT for Workplane 0 in file (Int64, LittleEndian) | | Start of fielpart for WorkPlane 0 |
| var | VectorBlock 0 of WorkPlane 0      |     |
| var | VectorBlock 1 of WorkPlane 0      |     |
| var | VectorBlock 2 of WorkPlane 0      |     |
| ... | ...      | ...     |
| var | WorkPlaneShell of WorkPlane 0 (WorkPlane message with empty vector-blocks array) | |
| var | WorkPlane LUT for WorkPlane 0 | |
| 8 | Position of LUT for Workplane 1 in file (Int64, LittleEndian) | |  Start of fielpart for WorkPlane 1 |
| var | VectorBlock 0 of WorkPlane 1      |     |
| var | VectorBlock 1 of WorkPlane 1      |     |
| ... | ...      | ...     |
| var | WorkPlaneShell of WorkPlane 1 (WorkPlane message with empty VectorBlocks array) | |
| var | WorkPlane LUT for WorkPlane 1 | |
| ... | ...      | ...     |
| var | JobShell (Job message with empty WorkPlane array) ||
| var | JobLUT || 

## JobLUT
The JobLUT contains 
- the position of the JobShell in the file
- an array of starting positions for each WorkPlane. At this given position in the file, there is an Int64 which in turn gives the position for the WorkPlaneLUT. 

On the first glance, it might seem counter-intutive that there is one Int64 in front of every WorkPlane to indicate the position of the WorkPlaneLUT, instead of just writing the position of the WorkPlaneLUT in the array in the JobLUT directly.

However, to speed up parsing of a job and parse multiple WorkPlanes at once, one can set up multiple sub-streams, one for each WorkPlane, and those sub-streams require the start and end of the part of the file they should work on. And by writing the starting position of the part of file for each WorkPlane into the WorkPlanePositions array in the JobLUT, the start and end position for the part of the file containing a specific WorkPlane is directly visible (see table above).

## WorkPlaneLUT
A WorkPlaneLUT contains
- the position of the WorkPlaneShell in the file
- an array of the starting positions for the VectorBlocks of this WorkPlane. 

Here, the array of starting positions direclty indicates the starting position of a VectorBlock protobuf message that can be parsed directly.

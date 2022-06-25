---
layout: post
title: "personal file naming conventions"
---

On both Windows and Linux, a "file" may contain any format of data, and this data can be absolutely anything which can be represented as a sequence of bits. Those bits can be transformed into useful information when they are passed to a file parser, and until then they are just a sequence of bits without a purpose. Unforuntately, it is very difficult to construct meaningful information with just a partial sequence of bits. For one, it is impossible to understand the entirety of a file's meaning without parsing the entirety of the file; provided that a partial sequence is a sequence which contains less data than is required by the parser, any amount of data must be non-partial for it to be parsable and therefore useful. Even if the data is usable by the parser, it may not be useful information because it lacks sorrounding data which allows people to accurately understand the information's wider purpose. As both of these facts are present within each and every possible file, it is impossible for a parser to present accurate information without absorbing the entire content of the file. This presents a problem for programs which must quickly convey information; in order to present information, the parser must transform the entire file.

The solution to this problem is metadata, which is commonly described as data about data. Metadata is just like other form of data in that it must be represented as a sequence of bits; however, the sequence of bits which represent metadata is not the entirety of the file, and in fact is usually much, much shorter than the sequence of bits which represent the actual data of the file. Some file formats automatically embed metadata into the content of the file; `mp3`, `pdf`, and `png` all embed information about the author, and EXIF data in raw images also embeds information about the camera's sensor directly into the stored file. File systems or operating systems (in some cases both) also store metadata about entire files. Arguably the most important metadata any file could have is a name. A file's name is an inherently more human-readable form of information as it is a string of text rather than a sequence of bits, and humans usually identify files by file name rather than any other form of information. 

In my opinion, file names are superior any other form of metadata for three reasons; first, since most implementations allow file names to be accessed without having to read any data from the file itself; therefore, file names be read faster than any other form of metadata; second, file names are returned by standard Linux tools (e.g. `ls`, `find`, `locate`, etc.), which allows pipelines to be created which operate on file names; and third, file names are essentially universal across operating systems, which while also true of metadata stored within a file, does not apply to metadata within a file if the data within a file is unreadable.

### stuffing metadata into a file name

Given that a file name has these advantages, it is reasonable to conclude that file names are opportune places to store a wealth of metadata about a file. Finding an effective way to encode metadata into a string representation is a challenge, but there is a viable solution which does not compromise the readability of a file name. 

In order to write metadata into a file name, the metadata must be first converted into a string representation. Luckily any data type and any structure of metadata can be represented in JSON, so the first step in solving this problem becomes translating the metadata of a file into a format which can be transformed into a JSON format. There is no universal algorithm which determines individual fields of metadata, determines the exact structure of all of the fields, and returns this structure as output; therefore, this problem must be solved for each format of metadata individually.

Once the metadata is in JSON format, the next issue which arises is converting the JSON string into a file name. JSON, while extremely explicit in defining the relationships between a fields in an object, makes use of nesting which does not translate well when condensed into a single line of text. Take the following JSON, representing some metadata about a file containing a music track, and how it appears when condensed.

```
/* original */

{
	"artists": [
		"Miss K8",
		"Angerfist"
	],
	"title": "Act on Impulse",
	"format": "mp3"
}

/* condensed */

{"artists":["Miss K8","Angerfist"],"title":"Act on Impulse","format":"mp3"}
```

The condensed version contains some extraneous information, namely the keys from the original JSON data. Assuming that the author of the file name understands some contextual information about the file, such as the possible authors and the possible file formats, the keys can be removed. Additionally, the characters `{": [,]}` do not play nice with command line interaction. The second problem can be solved by removing those pesky characters from the condensed string, which produces the following string if the labels are removed before trimming that character set. Note that space have been replaced by underscores, since spaces do delimit words in most human languages, but cause issues at the command line; underscores perform the same function of delimiting words, while also playing nice with the command line.  

```
/* condensed, keys removed */

{["Miss K8","Angerfist"],"Act on Impulse","mp3"}

/* condensed, keys removed, trimmed */

Miss_K8AngerfistAct_on_Impulsemp3

/* condensed, keys removed, trimmed, lowercased */

miss_k8angerfistact_on_impulsemp3
```

There is now a new problem; it is impossible to tell the original fields of the metadata apart since trimming that character set removed the special characters JSON used to indicated nested data and data within a shared structure. This problem can be easily resolved by adding delimiters between fields of metadata in the condensed version; personally, I prefer to use `-` (dash) as a field delimiter, and `+` (plus) to delimit metadata items which were elements in a JSON list. This step produces the final string, which is a file name friendly version of the metadata:

```
/* condensed, keys removed, trimmed, lowercased, delimited */

miss_k8+angerfist-act_on_impulse-mp3
```

### algorithm steps

1. Record file metadata in JSON format.
2. Condense JSON.
3. Remove JSON keys, keep JSON values.
4. Transform the JSON values into a command line-friendly text.
5. Delimit JSON values.
6. Join JSON values and return the final string.

### algorithm, pseudo-python

```
"""
Helper function to transform value to friendly string.
"""
def to_friendly(value):
    as_string = None
    if value is a list:
        elements = [to_friendly(element) for element in value]
        # Automatically delimits elements per Step 5
        as_string = ELEMENTS_DELIM.join(elements)
    elif value is a string:
        as_string = value
    elif value is a dict:
        values = [to_friendly(dict_value) for dict_value in value.values()]
        # Automatically delimits elements per Step 5
        as_string = ELEMENTS_DELIM.join(values)
    # Step 4
    as_string = as_string.lower().replace(" ", "_")
    return trim_all_unfriendly(as_string)

"""
Entry point to the algorithm. Steps 1 and 2 should be done before this.
"""
def to_filename(json_as_dict):
    # Step 3
    values = [to_friendly(value) for value in json_as_dict.values()]
    # Steps 5 and 6
    return FIELD_DELIM.join(values)
```

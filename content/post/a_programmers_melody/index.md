---
title: "A Programmer's Melody: Creating a Tool to Fix My Song Files"
date: 2024-03-09
image: cover.png
categories:
    - Projects
tags:
    - Python
    - Bash
---

[**View it on Github**](https://github.com/nashalexander/song_identifier)

## The Tragedy

Every data hoarder's nightmare — no, not hard drive corruption (though that's up there), but something equally as chilling: scrambled file names. Imagine your meticulously curated music collection became a cryptic mess of unidentifiable tracks!

```bash
$ ls my-lifes-mixtape/
 1027098972255028133.mp3
 1027167747674130870.mp3
 1027290277583497446.mp3
 1031191161722692691.mp3
 -1062590078858828842.mp3
 1098949889736289958.mp3
 -1131991569406517012.mp3
 1172902445231424744.mp3
 1173619643127455402.mp3
 -1205293020995236198.mp3
 ...
```

This is the horror I found myself in when revisiting my old collection of music. Somehow, the titles became unidentifiable indexed numbers. With hundreds of files like this, playing and identifying them all by hand would take an unreasonable amount of time. The only real choice to deal with this issue is to reach for the lazy programmer's playbook.

**Let's try to automate this.**

## The Lost Title Format

To begin, it's important to identify a target file format that would satisfy my data collection needs. An ideal file name format for this use case would include the title of the track as well as the artist name.

 `Track title - Artist.extension`

More song info could be collected, but with the title and artist name the files can at least be identified.

### There are a few good ways to collect track info:

- It's possible some files have their track info embedded in metadata. In this case, we can inspect the file for this metadata and determine if the track name and artist name are preserved.
- Identify the songs automatically based on audio data, this could be done with some song detection API.


## Extracting Metadata with ExifTool

By inspecting file metadata (with the tool `exiftool`), we can determine if track details are still present on any music files by parsing for Title and Artist details embedded within them.

**An example of inspecting a file:**

```bash
$ exiftool 8810117537767368468.mp3
ExifTool Version Number         : 12.40
File Name                       : 8810117537767368468.mp3
Directory                       : .
File Size                       : 7.8 MiB
File Modification Date/Time     : 2017:04:12 12:51:55-04:00
File Access Date/Time           : 2024:01:28 02:26:09-05:00
File Inode Change Date/Time     : 2017:04:12 12:51:55-04:00
# ...
Title                           : Mocki - Weekend (Jai Wolf Remix)
Artist                          : Underground Charisma
# ...
```

Here we can see an example of a file with Title and Artist details preserved. The file names for these are relatively easy to resolve, and we can setup a simple bash script to correct them.

### Parse metadata and request a rename

First let's automate retrieving our desired tags. We'll create a bash script leveraging `exiftool` to automate this.

```bash
# Just process the first command line argument as single file, and if there are
# no arguments then fail
file="${1:?}"

title=""
artist=""

if exiftool "${file}" | grep -q 'Title'; then
    title=$(exiftool -Title -m -p '${Title}' "${file}")
fi

if exiftool "${file}" | grep -q 'Artist'; then
    artist=$(exiftool -Artist -m -p '${Artist}' "${file}")
fi
```

To ensure name change actions are verified by a user directly, we can ask the user if the would like to rename the file based on the given data. Ensure the file extension is preserved using the [bash substring removal](https://tldp.org/LDP/abs/html/string-manipulation.html) pattern.

``` bash
# The expression removes everything but the file extension
file_ext="${file##*.}"

echo "${file} : ${title} - ${artist}.${file_ext}"
echo "rename file? (y/N)"

read answer
if [[ "${answer}" = "y" ]] || [[ "${answer}" = "Y" ]]; then
    mv "${file}" "${title} - ${artist}.${file_ext}"
else
    echo "${file} not renamed"
fi
```

### Edge case: Handle -filenames

Our script works - almost. There is an edge case that currently breaks it however. Files that begin with a `-`, such as `-1131991569406517012.mp3` cause an interesting error:

```bash
Invalid TAG name: "1131991569406517012.mp3"
No file specified
```
The error appears to imply `exiftool` is parsing the filename as an additional command line option due to the `-` signifying an option. To eliminate this issue, `exiftool` must be told where command line options end, and where file arguments begin. This can be accomplished by including `--` within the command call.

```bash
if exiftool -- "${file}" | grep -q 'Title'; then
    title=$(exiftool -Title -m -p '${Title}' -- "${file}")
fi
```

### Final Script

Before we finalize this script, let's add the functionality to handle all input CLI arguments as filenames. This will allow us to do simple calls such as `./rename_using_tags.sh *.mp3` within a target directory to completely repair corrupt file names!

Dry run capabilities are also included below, allowing test runs to substitute `mv` with `echo` instead. Dry run is activated by calling `-d` before including filenames. for example: `./rename_using_tags.sh -d *.mp3`. This can allow us to verify our script will function as expected before running any irreversible commands.

```bash
#!/bin/bash

MV_CMD=mv

# Dry run
if [[ "$1" = "-d" ]]; then
  MV_CMD=echo
  shift
fi

for file in "$@"; do
  title=""
  artist=""

  if exiftool -- "${file}" | grep -q 'Title'; then
    title=$(exiftool -Title -m -p '${Title}' -- "${file}")
  else
    continue
  fi

  if exiftool -- "${file}" | grep -q 'Artist'; then
    artist=$(exiftool -Artist -m -p '${Artist}' -- "${file}")
  fi

  file_ext="${file##*.}"

  echo "${file} : ${title} - ${artist}.${file_ext}"
  echo "rename file? (y/N)"
  
  read answer
  if [[ "${answer}" = "y" ]] || [[ "${answer}" = "Y" ]]; then
      file_ext="${file##*.}"
      
      $MV_CMD -- "${file}" "${title} - ${artist}.${file_ext}"
  else
      echo "${file} not renamed"
  fi

done
```

## Identifying tracks automatically with the ShazamIO API

What about the songs without any useful metadata to be found? Only inspecting the audio itself could help identify the track. For this case, an identification API such as [shazamio](https://pypi.org/project/shazamio/) can be leveraged.

### Test basic usage of ShazamIO

First let's implement a basic usage of ShazamIO (referencing their [example script](https://github.com/shazamio/ShazamIO/blob/master/examples/recognize_song.py)). With this, we can experiment and determine the functionality required to identify and retrieve our track data. ShazamIO is an asynchronous API, and thus we'll need to leverage coroutines to utilize it.

```python
import sys
import asyncio
from shazamio import Shazam, Serialize

# We'll pass in the song name via the first command line argument
song = sys.argv[1]

# Init API
shazam = Shazam()

# Generates a coroutine for us to run asynchronously
coroutine = shazam.recognize_song(song)

# Run and wait for coroutine to return
out = asyncio.run(coroutine)

# Serialize the out data buffer into a readable format
serialized = Serialize.full_track(out)

# Print to inspect available values
if serialized is None:
    print("Song ", song, " not found")
else:
    print(serialized)
```

From this quick example script, we can parse the returned data and determine the useful fields for our utility.

**Output:**
```bash
ResponseTrack(
    tag_id=UUID('18f05c79-9e5c-40ba-b2af-5207c37159be'), 
    retry_ms=None, 
    location=LocationModel(accuracy=0.01), 
    matches=[MatchModel(id='683393722', offset=92.227828125, time_skew=-0.06451672, frequency_skew=-0.009831846, channel=None)],
    timestamp=1708139612620,
    timezone='Europe/Moscow',
    track=TrackInfo(key=683393722,
    title='Let It Enfold You',
    subtitle='Senses Fail',
    artist_id='42',
    shazam_url='https://www.shazam.com/track/42',
    ...
```

Great, the title is an available field, and subtitle references the Artist of the song. We can create our parsing algorithm to retrieve this info and rename the song files.

### Encapsulate within an identify function

Let's create an identify function to retrieve the Shazam data, parse it, and return the desired file name. Making this function asynchronous will help us later.

For error scenarios, we'll use Python exceptions. These can be caught by the calling function, which can determine how to proceed with the song file. 

```python
async def identify(song_file):
    title = ""
    subtitle = ""
    song_file_extension = os.path.splitext(song_file)[1]

    shazam = Shazam()
    random.seed()

    try:
        out = await shazam.recognize_song(song_file)
    except Exception as e:
        raise Exception(f"Shazam could not recognize the song from file {song_file}")

    serialized = Serialize.full_track(out)

    if serialized.track.title is None:
        raise Exception(f"Song name of {song_file} not found")
    else:
        title = serialized.track.title

    if serialized.track.subtitle is None:
        raise Exception(f"Song artist of {song_file} not found")
    else:
        subtitle = serialized.track.subtitle
    
    # Return our desired naming format
    return title + " - " + subtitle + song_file_extension
```

### Create a process to handle the main execution logic

We'll create a main process to handle the control flow of our tool. Because the ShazamIO API takes some time to identify the song, our plan is to use several workers at the same time to quickly find and rename the many files given to our application.

Here, multiple identifier workers will be created. These workers will concurrently handle identifying all songs, and populating our async song queue when found.

For our limited amount of workers to traverse the long list of input files efficiently, a stride based setup can be leveraged. Each worker thread is given a start index and a stride length based on the total number of workers. Using this method allows for workers to process multiple songs by striding through the array without stepping on each other.

Here is example of a stride based traversal to illustrate this method:

```python
workers[3]
array[12]

# Process elements
workers[1] : 0, 3, 6, 9
workers[2] : 1, 4, 7, 10
workers[3] : 2, 5, 8, 11
```

There will be just one renamer worker however. This single worker handles taking files from the async queue and renaming them. This way, we keep the renaming part simple and sequential.

```python
async def process(file_list, num_coroutines):
    song_name_queue = asyncio.Queue()

    # Create producer tasks, each with a different start index and the same stride
    identifiers = [
        asyncio.create_task(
            identifier_coroutine(song_name_queue, file_list, i, num_coroutines)
        )
        for i in range(num_coroutines)
    ]

    renamer = asyncio.create_task(renamer_coroutine(song_name_queue))

    # Wait for all songs to be identified
    await asyncio.gather(*identifiers)

    # Wait until the queue is fully processed
    await song_name_queue.join()

    # Cancel the consumer task as it's intentionally an infinite loop
    renamer.cancel()
```

### Create coroutines to handle identifying songs and renaming files

Here we implement the identifier and renamer coroutines. As mentioned before, the `identifier_coroutine` will utilize a stride based implementation.

```python
async def identifier_coroutine(queue, data, start_index, stride):
    for i in range(start_index, len(data), stride):
        item = data[i]
        try:
            song_title = await identify(item)
            await queue.put([item, song_title])
        except Exception as e:
            print(e)

async def renamer_coroutine(queue):
    while True:
        result = await queue.get()
        file = result[0]
        new_name = result[1]

        print(f"renaming {file} to {new_name}")
        os.rename(file, new_name)

        # Notify the queue that the item has been processed
        queue.task_done()
```

### The throttling problem

The Shazam API unfortunately puts a cap on how many requests you can make in a certain amount of time. If this limit is exceeded, any additional requests will time out for a while. To avoid hitting this limit, we can make our identify function wait for a random amount of time before trying again. This way, not all of our identifier workers will send requests at the same time. Spacing out requests with this method after the timeout ends will help prevent overloading the Shazam API again with multiple requests at once.

```python
async def identify(song_file):
    title = ""
    subtitle = ""
    song_file_extension = os.path.splitext(song_file)[1]

    shazam = Shazam()
    random.seed()

    max_attempts = 20
    out = None

    # Shazam throttles song requests, so retry with sleep if exception occurs
    for attempt in range(max_attempts):
        try:
            out = await shazam.recognize_song(song_file)
            break
        except Exception as e:
            random_sleep_time = random.randint(1, 30)
            await asyncio.sleep(random_sleep_time)
    
    if out is None:
        raise Exception(f"Shazam could not recognize the song from file {song_file}")

    serialized = Serialize.full_track(out)
    ...
```

## Final Result

Some songs are still misidentified by ShazamIO, but for most cases this combo of tools fixed my music library.

```
    Fur Elise - Ludwig van Beethoven.mp3
    Sandstorm - Darude.mp3
    Canon in D - Johann Pachelbel.mp3
    Baby - Justin Bieber.mp3
    River Flows In You - Yiruma.mp3
    Never Gonna Give You Up - Rick Astley.mp3
```

## Additional Thoughts and Improvements

- These tools prompt the user before every file name change, but this could quickly become tedious. Instead, implementing a -y option to skip prompts could be effective.
- For review, adding the option to generate a log of file name changes could also help if some name change occurred that was not intended.
- Using an API like ShazamIO that connects to a commercially owned service is a weak point, and creates issues such as the throttling problem above. Using a different API, or replacing with a deep learning model that can run on local hardware or private servers would be preferred. At the time of writing this, I could not find a well developed and updated song identification model, but something like this should be feasible with current deep learning technology.
---
title: "Ripping Audio CDs"
excerpt: "Personal notes on ripping audio CDs to FLAC or MP3 on Linux using abcde.
Includes software installation, configuration, and basic usage for extracting
and encoding audio tracks from CDs."
categories:
  - Misc
tags:
  - abcde
  - audio
  - backup
---
## Introduction
In this post, I describe the steps I take to create a backup of my audio
CDs. I usually rip them either to FLAC or to 320 kbps MP3. After all
prerequisites are installed, ripping a CD is super easy and can be done by
executing a single command.

## Optical Disk Drive
I use an external optical disc drive that connects to my PC via USB. It's a
Hitachi-LG Slim Portable DVD Writer, model GP60NW60. I currently use Debian 13,
and there were no problems at all detecting this device, and I didn't need to
install any drivers manually. It works out of the box.

## Software Installation
We need to install *abcde* and at least one audio codec.

### Installing abcde
I use Debian, and on this distro, *abcde* is available through apt:
```bash
$ sudo apt install abcde
```
It should also be available in other major distros. Otherwise, you can always
download the package from [abcde downloads
page](https://abcde.einval.com/download/) and install it manually.

### Installing audio codecs
Audio encoders/decoders are not part of abcde and need to be installed
separately. Since I'm going to rip my CD into FLAC format, I need to install
the [FLAC Codec](https://xiph.org/flac/index.html):
```bash
$ sudo apt install flac
```
If I wanted to rip to MP3, I would need [LAME](https://lame.sourceforge.io/):
```bash
$ sudo apt install lame
```

## Ripping a CD
1. Insert the CD into the optical disc drive.
2. Navigate to a directory where you want the CD audio files to be stored.
   For example:
```bash
$ cd ~/Music
```
3. Run `abcde` with `-o` specifying the output format. For example, if you want
   to rip to FLAC:
```bash
$ abcde -o flac
```
4. *abcde* will now search for the album in the MusicBrainz database. After it
   finds at least one match, it will enter an interactive mode where the user
   is asked a few questions. I usually go with the first matching database record
   and choose the default options, as shown below. After that, give it a
   few minutes to grab all tracks.

```bash
Grabbing entire CD - tracks:  1 2 3 4 5 6
Retrieved 4 matches...
#1 (Musicbrainz): ---- Jean Michel Jarre / Oxygène ----
1: Oxygène, Part I
2: Oxygène, Part II
# Output removed to keep this short...
6: Oxygène, Part VI

Which entry would you like abcde to use (0 for none)? [0-4]: 1
Selected: #1 (Musicbrainz) (Jean Michel Jarre / Oxygène)
---- Jean Michel Jarre / Oxygène ----
Year: 1983
1: Oxygène, Part I
2: Oxygène, Part II
3: Oxygène, Part III
4: Oxygène, Part IV
5: Oxygène, Part V
6: Oxygène, Part VI
Edit selected CDDB data [y/N]? N
Is the CD multi-artist [y/N]? N
Grabbing track 1: Oxygène, Part I...
cdparanoia III release 10.2 (September 11, 2008)

Ripping from sector      32 (track  1 [0:00.00])
          to sector   34606 (track  1 [7:40.74])

outputting to /home/pawel/Music/abcde.4d094d06/track1.wav

 (== PROGRESS == [              | 034606 00 ] == :^D * ==)

Done.

# Grabbing the rest of the tracks...

Encoding track 6 of 6: Oxygène, Part VI...
Tagging track 6 of 6: Oxygène, Part VI...
Finished.

```

That's it! Your audio CD is now backed up!
```bash
$ ~/Music ls
Jean_Michel_Jarre-Oxygène
$ ~/Music ls -1 Jean_Michel_Jarre-Oxygène
1.Oxygène,_Part_I.flac
2.Oxygène,_Part_II.flac
3.Oxygène,_Part_III.flac
4.Oxygène,_Part_IV.flac
5.Oxygène,_Part_V.flac
6.Oxygène,_Part_VI.flac
```
## Links
* [abcde - A Better CD Encoder website](https://abcde.einval.com/wiki/FrontPage)
* [abcde(1) - Linux man page](https://linux.die.net/man/1/abcde)
* [MusicBrainz - the open music encyclopedia](https://musicbrainz.org/)


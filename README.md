# SevenZ4S

[![Maven Central](https://maven-badges.herokuapp.com/maven-central/fun.mactavish/sevenz4s/badge.svg)](https://maven-badges.herokuapp.com/maven-central/fun.mactavish/sevenz4s/)
[![Scaladoc](https://img.shields.io/badge/scaladoc-latest-blue.svg)](https://gonearewe.github.io/SevenZ4S/fun/mactavish/sevenz4s/index.html)
[![GitHub stars](https://img.shields.io/github/stars/gonearewe/SevenZ4S.svg?label=Stars)](https://github.com/gonearewe/SevenZ4S)
[![GitHub forks](https://img.shields.io/github/forks/gonearewe/SevenZ4S.svg?label=Fork)](https://github.com/gonearewe/SevenZ4S)
[![GitHub issues](https://img.shields.io/github/issues/gonearewe/SevenZ4S.svg?label=Issue)](https://github.com/gonearewe/SevenZ4S/issues)
[![license](https://img.shields.io/github/license/gonearewe/SevenZ4S.svg)](https://github.com/gonearewe/SevenZ4S/master/LICENSE)

# Introduction

This a 7Z compression library for Scala(v2.13), providing simple api to create, update
and extract archives of different formats.

This library offers compression and update abilities for 5 formats:

```
7-Zip	Zip	 GZip	Tar	 BZip2
```

As for extraction, following formats are all supported:

```
7-Zip	Zip	Rar	Tar	 Split	Lzma   Iso	HFS	 GZip
Cpio	BZip2	Z	 Arj	Chm	   Lhz	Cab	 Nsis
Ar/A/Lib/Deb	Rpm	 Wim	Udf	   Fat	Ntfs
```

Multiple platforms are also supported.

This project is based on [net.sf.sevenzipjbinding](http://sevenzipjbind.sourceforge.net/index.html),
which is a 7z binding library for java. You may acquire more info from its homepage.
Also, it's [licensed](https://github.com/borisbrodski/sevenzipjbinding/blob/master/License.txt)
 under LGPL and unRAR restriction.

# Quick Start

For creating archives, you need to first acquire an instance of concrete
`ArchiveCreator`. There're 5 kinds of `ArchiveCreator`, one for each format.
Some features and callback hooks may be optional. Archive is composed of many
entries, you need to pass some to `compress` method to finish compression.
Object `SevenZ4S` provides some utils that may help gain entries from local
file system.

```scala
  val path: Path = new File(getClass.getResource("/root").getFile).toPath

  def create7Z(): Unit = {
    val entries = SevenZ4S.get7ZEntriesFrom(path)
    new ArchiveCreator7Z()
      .towards(path.resolveSibling("root.7z"))
      // archive-relevant features
      .setLevel(5)
      .setSolid(true)
      .setHeaderEncryption(true)
      .setPassword("12345")
      .setThreadCount(3)
      .onEachEnd {
        ok =>
          if (ok)
            println("one success")
          else
            println("one failure")
      }.onProcess {
      (completed, total) =>
        println(s"$completed of $total")
    }.compress(entries)
  }
```

7Z engine offers abilities to modify, append and remove a few entries
in a more efficient way. Use `ArchiveUpdater` to update archive instead of
extract-modify-compress by yourself.

```scala
  // import ...
  import fun.mactavish.sevenz4s.Implicits._
  
  def update7Z(): Unit = {
    val replacement = path.resolveSibling("replace.txt")

    val updater = new ArchiveUpdater7Z()
      .from(path.resolveSibling("root.7z")) // update itself
      .withPassword("12345")
      .update {
        entry =>
          if (entry.path == "root\\a.txt") { // path separator is OS-relevant
            entry.copy(
              dataSize = Files.size(replacement), // remember to update size
              source = replacement, // implicit conversion happens here
              path = "root\\a replaced.txt" // file name contains space
            )
          } else {
            entry
          }
      }

    updater.removeWhere(entry => entry.path == "root\\sub\\deeper\\c.txt")
    // directory can not be deleted until its contents have all been deleted,
    // otherwise, it ignores the request silently rather than raise an exception.
    updater.removeWhere(entry => entry.path == "root\\sub\\deeper")
    updater += SevenZ4S.get7ZEntriesFrom(replacement).head
    // notice that file with the same name is allowed,
    // but may be overwritten by OS's file system during extraction
    updater += SevenZ4S.get7ZEntriesFrom(replacement).head
  }
```

Unlike `ArchiveCreator` and `ArchiveUpdater`, SevenZ4S provides a generic
`ArchiveExtractor` supporting the extraction of all formats. It can
autodetect the format of input archive.

```scala
  def extract7Z(): Unit = {
    new ArchiveExtractor()
      .from(path.resolveSibling("root.7z"))
      .withPassword("12345")
      .onEachEnd(println(_))
      .foreach { entry =>
        println(entry.path)
        if (entry.path == "root\\b.txt") {
          // extract independently
          entry.extractTo(path.resolveSibling("b extraction.txt"))
        }
      }
      // `extractTo` takes output folder `Path` as parameter
      // `onEachEnd` callback only triggers on `extractTo`
      .extractTo(path.resolveSibling("extraction"))
      .close() // ArchiveExtractor requires closing
  }
```

If all you want is simply compressing and extracting files on local file
system, the object `SevenZ4S` provides super useful utilities.

```scala
  val path: Path = new File(getClass.getResource("/root").getFile).toPath

  def test7Z(): Unit = {
    val output = path.resolveSibling("util test/7z")

    SevenZ4S.compress(ArchiveFormat.SEVEN_Z, path, output)
    val f = output.resolve("root.7z")

    SevenZ4S.extract(f, output)
  }
```

Refer to the test cases and test resources for more details.

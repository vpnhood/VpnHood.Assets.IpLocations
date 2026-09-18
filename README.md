# VpnHood.Assets.IpLocations

The **IP2Location LITE** country database as an **inert asset package**: data plus MSBuild targets,
and **no compiled code**. Nothing in your codebase can take a reference on it — it is a store of
bytes, not a library.

```xml
<PackageReference Include="VpnHood.Core.IpLocations.Assets.Ip2LocationLite" />
```

That is the whole integration. The package's targets place `IpLocations.zip` where your app's
platform keeps files, and you read it from the asset path **`iplocations/IpLocations.zip`**. That
path is the entire contract between this package and your code.

## Why it is data and not an embedded resource

.NET for Android keeps assemblies **once per CPU architecture**. A 14.6 MB database compiled into a
DLL is therefore carried once per ABI — three or four times over in a multi-ABI package. Placed as
data it is carried once, in the part of the package Android does not split by architecture.

Reading it as a stream rather than a `byte[]` also keeps those 14.6 MB out of the managed heap: the
zip's entries are **stored, not deflated**, so a reader seeks straight to the country it wants.

## Where the file lands

The targets live in `buildTransitive/`, so an app that reaches this package **through** another
package or project gets the file placed just the same, at any depth. Only an **app's** build places
it; a library that merely passes the reference along gets no 14.6 MB copy in its `bin`.

| platform | placed as | where it ends up |
| --- | --- | --- |
| Windows, Linux | `None` + copy-to-output | `iplocations/IpLocations.zip` beside the executable |
| iOS, tvOS, Mac Catalyst | `BundleResource` | `iplocations/IpLocations.zip` in the app bundle |
| Android | `AndroidAsset` | `iplocations/IpLocations.zip` inside the `.apk` |
| Browser (WASM) | nothing | no filesystem to read |

## Reading it

### Windows, Linux, iOS, tvOS, Mac Catalyst

An ordinary file, under `AppContext.BaseDirectory`:

```csharp
var path = Path.Combine(AppContext.BaseDirectory, "iplocations", "IpLocations.zip");
using var zip = new ZipArchive(File.OpenRead(path), ZipArchiveMode.Read);
```

### Android

On Android the asset is **not a file**. It is an entry of the `.apk`, reachable only through
`AssetManager`, and the stream it hands out is forward-only — which `ZipArchive` cannot use,
because it reads the central directory at the end of the file and then seeks back to each entry.

So copy it into memory and read it from there:

```csharp
using var source = context.Assets.Open("iplocations/IpLocations.zip");
var memoryStream = new MemoryStream();
source.CopyTo(memoryStream);
memoryStream.Position = 0;

using var zip = new ZipArchive(memoryStream, ZipArchiveMode.Read);
```

The copy costs the database's full size in memory for as long as you hold it, so open it when a
lookup actually needs it and dispose it afterwards, rather than keeping one alive for the life of
the app.

`AssetManager.OpenFd` will tell you the entry's byte offset and length inside the `.apk`, which
tempts you to read it in place with no copy at all. Two conditions have to hold for that to be
correct: the entry must be **stored rather than deflated**, or there is no file descriptor to open;
and the offset must index into the file you think it does, which is `ApplicationInfo.SourceDir` for
a normal install but not necessarily for one delivered in a split or an asset pack. The second is
the dangerous one — if it is wrong the read does not fail, it returns whatever bytes lie at that
offset. Weigh that before choosing it over a copy.

## What is in the zip

One entry per country, named by its lower-case ISO code (`tr.ips`), holding that country's IP
ranges already sorted and unified, plus `_checksum.txt` naming the build of the data. Every entry
is stored uncompressed.

## Licensing — read this before you ship

This package is licensed in **two parts**, because it holds two different kinds of thing.

| part | licence |
| --- | --- |
| `buildTransitive/IpLocations.zip` — the data, derived from IP2Location LITE | **CC BY-SA 4.0**, plus IP2Location's attribution requirement |
| the MSBuild targets that place it | LGPL v2.1 |

Both licence texts ship inside the package, under `licenses/`, and `LICENSE` sets out which applies
to what. The data is a converted form of the IP2Location LITE database, so anyone redistributing it
or an adaptation of it carries the **ShareAlike** obligation forward.

### The attribution you owe

IP2Location LITE requires that **all sites, advertising materials and documentation** mentioning
features or the use of this database display this acknowledgment:

> [Your site name or product name] uses the IP2Location LITE database for
> [IP geolocation](https://lite.ip2location.com).

Referencing this package does **not** discharge that for you. It lands on your product — its about
screen, its documentation, or its website — with your own name in place of the bracketed part.

## Data source

This package includes **IP geolocation data** from [IP2Location LITE](https://lite.ip2location.com).
IP2Location LITE is **Copyright (c) 2001-2024 Hexasoft Development Sdn. Bhd.** All Rights Reserved.
IP2Location is a registered trademark of Hexasoft Development Sdn Bhd.

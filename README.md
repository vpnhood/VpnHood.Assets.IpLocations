# VpnHood.Assets.IpLocations

The **IP2Location LITE** country database as an **inert asset package**: data plus MSBuild targets,
and **no compiled code**. Nothing in your codebase can take a reference on it — it is a store of
bytes, not a library.

```xml
<PackageReference Include="VpnHood.Assets.Ip2LocationLite" />
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

### Android — read it in place, do not copy it

On Android the asset is **not a file**. It is an entry of the `.apk`, and `AssetManager.Open`
returns a forward-only stream, which `ZipArchive` cannot use because it seeks.

Do **not** solve that by copying the asset to the cache folder: that spends another 14.6 MB of the
user's storage on bytes the app already carries. Read it where it lies instead. `OpenFd` gives the
entry's offset and length inside the `.apk`, and the `.apk` is an ordinary file, so a window onto
that byte range is an ordinary seekable stream:

```csharp
// 1. where the entry lies inside the package
using var descriptor = context.Assets.OpenFd("iplocations/IpLocations.zip");
var packagePath = context.ApplicationInfo.SourceDir;

// 2. a seekable window onto that range - nothing is copied
var stream = new SubStream(File.OpenRead(packagePath), descriptor.StartOffset, descriptor.Length);
using var zip = new ZipArchive(stream, ZipArchiveMode.Read);
```

`SubStream` here is any read-only, seekable window over another stream: position 0 is the window's
start, its length is the window's length, and each read is served from `offset + position` of the
source. It is about sixty lines, and it is the only code this package's Android support needs.

**The entry must be stored, not deflated**, or `OpenFd` has no file descriptor to hand out and
throws. This package's targets already arrange that by adding `.zip` to
`AndroidStoreUncompressedFileExtensions` for the consuming app, so it works out of the box; if you
place the file yourself, you have to do the same.

## What is in the zip

One entry per country, named by its lower-case ISO code (`tr.ips`), holding that country's IP
ranges already sorted and unified, plus `_checksum.txt` naming the build of the data. Every entry
is stored uncompressed.

## Data source

This package includes **IP geolocation data** from [IP2Location LITE](https://lite.ip2location.com).
IP2Location LITE is **Copyright (c) Hexasoft Development Sdn. Bhd.** All Rights Reserved.

## License & attribution

This package **requires attribution** under IP2Location LITE's license. If you use it, you must
acknowledge IP2Location LITE as follows:

> This nuget uses the IP2Location LITE database for [IP geolocation](https://lite.ip2location.com).

For more details, visit the [IP2Location LITE license](https://lite.ip2location.com).

# [Wintun Network Adapter](https://www.wintun.net/)
### TUN Device Driver for Windows

This is a layer 3 TUN driver for Windows 7, 8, 8.1, and 10. Originally created for [WireGuard](https://www.wireguard.com/), it is intended to be useful to a wide variety of projects that require layer 3 tunneling devices with implementations primarily in userspace.

## Build Requirements

- [Visual Studio 2017](https://visualstudio.microsoft.com/downloads/)
- [Windows Driver Kit for Windows 10](https://docs.microsoft.com/en-us/windows-hardware/drivers/download-the-wdk)
- [WiX Toolset 3.11.1](http://wixtoolset.org/releases/)


## Digital Signing

Digital signing is integral part of the build process. By default, the driver will be test-signed using a certificate that the WDK should automatically generate. To subsequently load the driver, you will need to put your computer into test mode by executing as Administrator `bcdedit /set testsigning on`.

If you possess an EV certificate for kernel mode code signing you should switch TUN driver digital signing from test-signing to production-signing by authoring your `wintun.vcxproj.user` file to look something like this:

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project ToolsVersion="15.0" xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <PropertyGroup>
    <SignMode>ProductionSign</SignMode>
    <CrossCertificateFile>$(WDKContentRoot)CrossCertificates\DigiCert_High_Assurance_EV_Root_CA.crt</CrossCertificateFile>
    <ProductionCertificate>DF98E075A012ED8C86FBCF14854B8F9555CB3D45</ProductionCertificate>
    <TimestampServer>http://timestamp.digicert.com</TimestampServer>
  </PropertyGroup>
</Project>
```

Modify the `<CrossCertificateFile>` to contain the full path to the cross-signing certificate of CA that issued your certificate. You should be able to find its `.crt` file in `C:\Program Files (x86)\Windows Kits\10\CrossCertificates`. Note that the `$(WDKContentRoot)` expands to `C:\Program Files (x86)\Windows Kits\10\`.

If you already have `wintun.vcxproj.user` file, just add the `<PropertyGroup>` section.


## Building from Command Line

Open _Developer Command Prompt for VS 2017_ and use the `msbuild` command:

```
msbuild wintun.proj [/t:<target>]
```

### Targets

  - `Build`: Builds the driver release configurations of all supported platforms. This is the default target.

  - `Clean`: Deletes all intermediate and output files.

  - `Rebuild`: Alias for `Clean` followed by `Build`.

  - `SDV`: Runs Static Driver Verifier, which includes a clean driver build, only for AMD64 release configuration.

  - `DVL`: Runs the `SDV`, and creates a Driver Verification Log, only for AMD64 release configurations.

  - `MSM`: Builds Microsoft Installer Merge Modules in `<output folder>\wintun-<platform>-<version>.msm`. Requires WHLK signed driver (See: [Building Microsoft Installer Merge Modules]).

The driver output folders are:

Platform and Configuration | Folder
-------------------------- | --------------------
x86 Debug                  | `x86\Debug\wintun`
x86 Release                | `x86\Release\wintun`
AMD64 Debug                | `amd64\Debug\wintun`
AMD64 Release              | `amd64\Release\wintun`
ARM64 Debug                | `arm64\Debug\wintun`
ARM64 Release              | `arm64\Release\wintun`

### Building Microsoft Installer Merge Modules

1. `msbuild wintun.proj /t:DVL;Build`
2. Perform Windows Hardware Lab Kit tests; Submit submission package to Microsoft; Get WHQL signed driver.
3. Copy WHQL signed driver to `whql\x86` and `whql\amd64` subfolders.
4. `msbuild wintun.proj /t:MSM`
5. MSM files are placed in `dist` subfolder.


## Usage

After loading the driver and creating a network interface the typical way using [SetupAPI](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/setupapi), open `\\.\Device\WINTUN%d` as Local System, where `%d` is the [LUID](https://docs.microsoft.com/en-us/windows/desktop/api/ifdef/ns-ifdef-_net_luid_lh) index (`NetLuidIndex` member) of the network device. You may then [`ReadFile`](https://docs.microsoft.com/en-us/windows/desktop/api/fileapi/nf-fileapi-readfile) and [`WriteFile`](https://docs.microsoft.com/en-us/windows/desktop/api/fileapi/nf-fileapi-writefile) bundles of packets of the following format:

```
+------------------------------+
| size_0                       |
|   4 bytes, native endian     |
+------------------------------+
|                              |
| padding                      |
|   12 bytes, all zero         |
|                              |
+------------------------------+
|                              |
| packet_0                     |
|   size_0 bytes               |
|                              |
~                              ~
|                              |
+------------------------------+
| padding                      |
|   16-((4+size_0)&15) bytes,  |
|   all zero                   |
+------------------------------+
| size_1                       |
|   4 bytes, native endian     |
+------------------------------+
|                              |
| padding                      |
|   12 bytes, all zero         |
|                              |
+------------------------------+
|                              |
| packet_1                     |
|   size_1 bytes               |
|                              |
~                              ~
```

Each packet segment should contain a layer 3 IPv4 or IPv6 packet. Up to 256 packets may be read or written during each call to `ReadFile` or `WriteFile`.

It is advisable to use [overlapped I/O](https://docs.microsoft.com/en-us/windows/desktop/sync/synchronization-and-overlapped-input-and-output) for this. If using blocking I/O instead, it may be desirable to open separate handles for reading and writing.

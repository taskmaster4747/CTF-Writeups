# HTB Sherlock: Frosted — Hard
## Digital Forensics and Incident Response Investigation

**Platform:** Hack The Box  
**Challenge:** Sherlock — Frosted  
**Difficulty:** Hard  
**Category:** Digital Forensics and Incident Response (DFIR)  
**Completion date:** 5 November 2026  
**Solver position:** 146th  
**Status:** Completed — 10/10 tasks

---

# 1. Executive Summary

This report documents my investigation of **Frosted**, a Hard-level Digital Forensics and Incident Response Sherlock challenge on Hack The Box.

The investigation involved analyzing a compromised container environment, examining memory artifacts, investigating suspicious Java code, identifying attacker infrastructure, examining Kubernetes-related artifacts, and reconstructing malicious activity.

The investigation was performed using Linux command-line tools and forensic artifacts supplied with the challenge.

The key areas investigated included:

- Memory-image analysis and string extraction.
- Container filesystem and file-descriptor examination.
- Recovery and analysis of a suspicious Java class.
- Investigation of Kubernetes service-account artifacts.
- Network indicator identification.
- Detection of suspicious process and thread activity.
- Identification of attacker-created files and data staging.
- Correlation of evidence to answer all ten Sherlock questions.

I successfully completed all ten tasks and achieved the **146th solve position**. This was my first completed Hard-level Sherlock.

# 2. Investigation Objectives

The main objectives were:

1. Identify the initial attacker infrastructure.
2. Locate artifacts associated with the attacker's covert activity.
3. Determine relevant filesystem metadata.
4. Identify suspicious external communication endpoints.
5. Recover attacker-related configuration or credentials.
6. Identify the initial compromise source.
7. Investigate abuse of a trusted reporting workflow.
8. Identify Kubernetes service-account activity.
9. Identify suspicious Java runtime activity.
10. Locate the archive associated with data staging or exfiltration.

# 3. Investigation Environment

## 3.1 Working environment

The investigation was performed in a Linux environment.

Working directory:

```bash
~/Desktop/HTB/Frosted/checkpoint-report-pod
```

## 3.2 Evidence inventory

The investigation involved the following artifacts:

| Artifact | Purpose |
|---|---|
| `checkpoint/pages-1.img` | Memory-page image used for forensic examination |
| `checkpoint/pages-1_strings.txt` | Extracted readable strings from memory |
| `checkpoint/files.json` | File-descriptor and filesystem metadata |
| `checkpoint/ghost-file-1.img` | Recovered deleted-file artifact |
| `checkpoint/tcp-stream-*.img` | Network stream artifacts |
| `checkpoint/memory_strings.txt` | Additional extracted memory strings |
| `container.log` | Container-related event information |
| `dump.log` | Dump and artifact-recovery information |
| `rootfs-diff.tar` | Container filesystem differences |
| `rootfs-diff/` | Extracted filesystem differences |
| `rootfs_extracted/` | Extracted filesystem contents |
| `extracted/local/lab/Stager.class` | Recovered suspicious Java class |

# 4. Investigation Methodology

I approached the investigation by examining different evidence sources and correlating their findings.

The general workflow was:

1. Inspect the available challenge artifacts.
2. Extract readable strings from the memory image.
3. Search for suspicious paths, IP addresses, domains, report identifiers, and process names.
4. Examine container filesystem metadata.
5. Recover and inspect suspicious Java class data.
6. Correlate findings with network and container artifacts.
7. Verify the findings against the Sherlock questions.

An important lesson from this investigation was that a single artifact rarely provides the entire answer. Findings become more useful when correlated with other evidence.

# 5. Investigation and Technical Findings

## 5.1 Memory-image investigation

The memory image was one of the most important sources of information.

I used the extracted strings file to search for readable information without repeatedly examining the entire binary image manually.

The file was:

```text
checkpoint/pages-1_strings.txt
```

I searched for relevant terms associated with reports, processes, network communication, suspicious paths, and Java functionality.

Example investigation commands:

```bash
grep -Ein 'report|scheduler|export|generate|document|template|download|attachment|artifact|upload' checkpoint/pages-1_strings.txt
```

I also searched for suspicious file paths and application-related references:

```bash
grep -Ein 'filename|fileName|file_path|filepath|outputFile|createTempFile|Files\.write|FileOutputStream|\.pdf|\.csv|\.xlsx|\.json' checkpoint/pages-1_strings.txt
```

This helped narrow down the investigation to relevant portions of the extracted memory strings.

**Important finding:** The memory artifacts contained information associated with the reporting application, the trusted scheduler workflow, and the suspicious Java stager.

## 5.2 Container filesystem investigation

I examined the extracted filesystem differences to identify files created or modified during the incident.

Command:

```bash
find rootfs-diff -type f -printf '%s bytes  %p\n' | sort -nr | head -100
```

The notable results included:

```text
32768 bytes  rootfs-diff/tmp/hsperfdata_ubuntu/1
6078 bytes   rootfs-diff/tmp/hsperfdata_ubuntu/1_string.txt
```

I also searched the extracted filesystem for report-related names and suspicious paths:

```bash
grep -RInaE 'report|scheduler|Ad Spend|Conversion Funnel|Invoice Aging|filename|output' rootfs_extracted 2>/dev/null | head -100
```

This search did not directly reveal the trusted workflow's generated report artifact.

That was an important investigative lesson: an artifact may not remain in the final filesystem snapshot. It may need to be identified through memory, logs, or other evidence sources.

## 5.3 File-descriptor investigation

I examined:

```text
checkpoint/files.json
```

The file-descriptor information included references to:

```text
/app/app.jar
/tmp/.x
/app
/
```

The application JAR appeared in two file descriptors:

- FD 27 — `/app/app.jar`
- FD 28 — `/app/app.jar`

Another relevant entry was:

```text
/tmp/.x
```

The investigation also identified a deleted-file reference associated with this path in `dump.log`.

I examined the recovered deleted-file artifact:

```text
checkpoint/ghost-file-1.img
```

Its readable contents included:

```text
campaign=RAKSHASA
operator=bhairav
worker=worker-03
```

These contents were useful for understanding the attacker's filesystem activity.

**Forensic distinction:** `/tmp/.x` is the path shown in the recovered container artifact. The answer recorded for Task 2 is `/tmp/x`. These should not be treated as interchangeable without checking the original task wording and supporting evidence.

## 5.4 Java stager investigation

One of the most interesting parts of Frosted was the recovery of a suspicious Java class.

Recovered artifact:

```text
extracted/local/lab/Stager.class
```

Class information:

- Class: `local.lab.Stager`
- Source file: `Stager.java`
- Java class-file version: 61.0

The recovered class contained methods including:

```text
<init>
run
buildReconBlob
q
readFirstLine
stratumSession
send
```

I examined the class's strings and references to understand its functionality.

Notable references included:

```text
/var/run/secrets/kubernetes.io/serviceaccount/token
/tmp/.x
/app/.cache
/app/.cache/.mid
/bin/sh
```

The class also referenced Java functionality associated with:

- `Paths.get`
- `Files.readAllBytes`
- `RandomAccessFile`
- `Files.write`
- `ProcessBuilder.start`
- File deletion and directory creation.

These references indicated filesystem operations, process execution, and access to Kubernetes service-account material.

### Suspicious command

The recovered class contained the following shell command:

```bash
mkdir -p /dev/shm/.X11-unix && {
    id
    uname -a
    echo BHAIRAV-WAS-HERE
} > /tmp/r.txt &&
tar -czf /dev/shm/.X11-unix/.cache.tar.gz -C /tmp r.txt &&
rm -f /tmp/r.txt
```

This command performs several operations:

1. Creates a directory under `/dev/shm`.
2. Collects basic system information.
3. Writes the collected information to `/tmp/r.txt`.
4. Compresses the file into an archive.
5. Removes the temporary text file.

The resulting archive path was:

```text
/dev/shm/.X11-unix/.cache.tar.gz
```

This was an important indicator when investigating the data-staging activity.

### Java stager interpretation

The recovered class demonstrated why Java memory and class-file analysis can be valuable during container investigations.

A suspicious Java class may contain important functionality even when the corresponding source file is unavailable in the filesystem.

## 5.5 Trusted reporting workflow investigation

Frosted involved a reporting application and an internal scheduler.

One relevant HTTP request contained:

```text
POST /api/v1/report
```

The request included:

```text
user-agent: analytics-scheduler/1.4 (+internal)
```

The request body contained:

```text
title=Ad+Spend+ROI+2026&owner=growth
```

The associated request ID was:

```text
a501e4b068dbc76dd
```

The request also contained serialized Java-related data, including references to:

```text
Stager.java
SourceFile
```

This was an important clue because it connected the reporting workflow to the suspicious Java class.

### Task 7 — Trusted workflow artifact

The final answer was:

```text
Report ID: RPT-1057
Title: Untitled
```

This task was particularly interesting because identifying the report required investigating the reporting workflow rather than simply looking for obvious malware filenames.

During the investigation, I also examined other report identifiers and titles. Some were misleading or unrelated to the final answer.

The important lesson was to avoid assuming that a report created around the same time as suspicious activity was automatically the correct artifact.

**Evidence locator to complete before publication:** Add the exact memory-string line, log entry, or other artifact that established `RPT-1057` and `Untitled`. The full surrounding output was not preserved in my current notes.

## 5.6 Network investigation

The investigation included identifying suspicious network endpoints and correlating them with the recovered Java stager and other artifacts.

The recovered stager contained references to an external IP address:

```text
185.220.101.47
```

It also contained mining-related configuration and a Stratum communication method.

The investigation identified the following network indicators:

```text
185.220.101.47:14444
xmrp.eternal-hashes.net:55555
```

These indicators were relevant to the covert communication and mining-related activity identified during the investigation.

The network findings should be supported in the final report with the exact TCP stream, memory offset, or log location where each indicator was recovered.

## 5.7 Initial compromise source

The investigation identified the following IP address as the initial compromise source:

```text
27.34.64.82
```

This was one of the key indicators required to reconstruct the incident.

For the final report, include the original evidence location and the relevant request or connection context that establishes its role.

## 5.8 Kubernetes service-account investigation

The recovered Java stager referenced:

```text
/var/run/secrets/kubernetes.io/serviceaccount/token
```

This is the conventional location of a Kubernetes service-account token inside a container.

The class also referenced environment variables including:

```text
HOSTNAME
KUBERNETES_SERVICE_PORT_HTTPS
JAVA_HOME
KUBERNETES_PORT
AWSACCESSKEY
AWSSECRETKEY
IAMROLE
CLUSTERNAME
SERVICEACCOUNT
```

These references indicated that the stager was designed to collect information about its execution environment and access relevant service-account material.

The investigation also identified the service-account authentication expiry:

```text
2027-06-29 03:29:09
```

This finding was important when examining the Kubernetes-related portion of the incident.

## 5.9 Suspicious Java runtime thread

Another finding was the suspicious thread name:

```text
C2 CompilerThread9
```

This was the answer to Task 9.

The significance of this finding was that runtime thread information can provide useful forensic context when investigating suspicious Java execution.

A thread name alone should not be treated as proof of malicious activity. Its significance comes from correlating it with the other recovered artifacts and the challenge's evidence.

## 5.10 Data staging and exfiltration archive

The recovered Java class contained functionality associated with collecting system information and creating a compressed archive.

The identified archive path was:

```text
/dev/shm/.X11-unix/.cache.tar.gz
```

The command found in the class showed that `/tmp/r.txt` was archived and then deleted.

This is an example of why examining only the final filesystem can miss important evidence: temporary files can be removed after their contents have been collected.

# 6. Final Sherlock Answers

The following table records my completed answers.

| Task | Answer |
|---:|---|
| 1 | `185.220.101.47:14444` |
| 2 | `/tmp/x` |
| 3 | `323271` |
| 4 | `xmrp.eternal-hashes.net:55555` |
| 5 | Recovered credential — retain the exact value in the private evidence record |
| 6 | `27.34.64.82` |
| 7 | `RPT-1057` — `Untitled` |
| 8 | `2027-06-29 03:29:09` |
| 9 | `C2 CompilerThread9` |
| 10 | `/dev/shm/.X11-unix/.cache.tar.gz` |

**Completion:** 10/10 tasks solved.

**Solver position:** 146th.

# 7. Investigation Challenges

Several parts of the investigation required more than simple keyword searches.

### 7.1 Large memory artifacts

The memory strings file contained a large amount of Java, Spring Framework, and application-related data.

Searching for generic terms generated many irrelevant results.

I had to narrow searches using more specific terms and examine surrounding context.

### 7.2 Distinguishing application artifacts from attacker artifacts

Not every suspicious file was the answer to a Sherlock task.

For example, `/tmp/.x` and `/app/.cache/.mid` were relevant to the recovered stager, but their presence did not automatically establish that they were the trusted workflow's generated artifact.

### 7.3 Investigating the reporting workflow

The report-related HTTP requests contained multiple identifiers and titles.

It was necessary to distinguish the relevant workflow artifact from other reporting activity.

The final Task 7 answer was `RPT-1057 — Untitled`.

### 7.4 Correlating different evidence sources

The Java class, memory strings, file-descriptor information, network artifacts, and container logs provided different pieces of the investigation.

Understanding how these artifacts related to each other was more useful than analyzing them independently.

# 8. Lessons Learned

Completing Frosted helped me develop a better understanding of several cybersecurity and forensic concepts.

1. **Memory forensics:** Readable strings and recovered class data can expose evidence that is not available in the filesystem.
2. **Container forensics:** Container snapshots and filesystem differences do not necessarily preserve every temporary or deleted artifact.
3. **Java analysis:** Examining class files can reveal suspicious methods, filesystem operations, and process execution.
4. **Kubernetes security:** Service-account tokens and environment information can be valuable targets during container compromise.
5. **Network forensics:** Correlating IP addresses, domains, ports, and application behavior helps reconstruct attacker activity.
6. **Evidence correlation:** A finding becomes stronger when multiple independent artifacts support it.
7. **Analytical discipline:** A plausible answer is not necessarily a correct answer. Findings need evidence.

# 9. Conclusion

Frosted was my first Hard-level Sherlock challenge on Hack The Box.

The investigation required me to work with memory artifacts, container filesystem data, Java class files, Kubernetes-related information, and network indicators.

Completing all ten tasks was a significant milestone in my cybersecurity learning journey.

I also learned that DFIR is not simply about finding suspicious strings or identifying malware. It is about understanding what happened, identifying the evidence that supports each conclusion, and explaining the investigation in a reproducible way.

**Final result: 10/10 tasks solved — 146th solver.**

---

# Appendix A — Commands Used During Investigation

The following commands are taken from my investigation notes.

### Search memory strings

```bash
grep -Ein 'report|scheduler|export|generate|document|template|download|attachment|artifact|upload' checkpoint/pages-1_strings.txt
```

### Search for suspicious file-generation references

```bash
grep -Ein 'filename|fileName|file_path|filepath|outputFile|createTempFile|Files\.write|FileOutputStream|\.pdf|\.csv|\.xlsx|\.json' checkpoint/pages-1_strings.txt
```

### Inspect extracted filesystem differences

```bash
find rootfs-diff -type f -printf '%s bytes  %p\n' | sort -nr | head -100
```

### Search extracted filesystem

```bash
grep -RInaE 'report|scheduler|Ad Spend|Conversion Funnel|Invoice Aging|filename|output' rootfs_extracted 2>/dev/null | head -100
```

### Inspect relevant memory strings

```bash
sed -n '550220,550270p' checkpoint/pages-1_strings.txt
```

### Search for application-related class names

```bash
grep -E '^[a-zA-Z0-9_$/.-]*(Controller|Service|Repository|Scheduler|Report|Job|Task|Generator)[a-zA-Z0-9_$/.-]*$' checkpoint/pages-1_strings.txt | sort -u | head -200
```

### Inspect file-descriptor metadata

```bash
python3 - <<'PY'
import json

d = json.load(open("checkpoint/files.json"))

def walk(x):
    if isinstance(x, dict):
        if "name" in x and isinstance(x["name"], str):
            print(x["name"])
        for v in x.values():
            walk(v)
    elif isinstance(x, list):
        for v in x:
            walk(v)

walk(d)
PY
```

### Search for Task 7 report evidence

```bash
rg -n -a -i -C 12 'RPT-1057|Untitled' checkpoint/pages-1_strings.txt container.log dump.log
```

**Note:** This appendix records commands visible in my investigation notes. It is not claimed to be a complete command history.

# Appendix B — Publication and Evidence Notes

Before making this report public:

- Add exact evidence line numbers and memory offsets for every answer.
- Add the full Task 5 answer to the private version; redact credentials from the public version.
- Add the exact evidence supporting Task 7.
- Add screenshots of important terminal results.
- Verify the chronology against original timestamps.
- Verify Hack The Box's Sherlock publication rules.
- Do not upload original challenge evidence or potentially sensitive credentials to a public repository.

# PHP2Shell (CVE-2026-109618 / CVE-2026-109619)

This repository contains a Python Proof of Concept (PoC) executed in an isolated Docker-based lab environment for reproducing and analyzing the PHP2Shell vulnerability.

The vulnerability is a pre-authentication Remote Code Execution (RCE) flaw in PHP resulting from Out-of-Memory (OOM) bailout state corruption. The exploit specifically targets the Apache `mod_php` SAPI and abuses the `session.upload_progress` feature to trigger a Use-After-Free (UAF), bypass ASLR, and execute arbitrary commands.

> **Disclaimer:** This project is intended exclusively for educational research, vulnerability analysis, and authorized security assessments conducted within closed, isolated laboratory environments. Do not execute this PoC against systems without explicit authorization.

## Lab Prerequisites

The following components are required:

- Docker
- Docker Compose
- Python 3.10 or later
- Kali Linux

## Execution Steps

### 1. Start the Target Environment

Navigate to the lab directory and start the Docker container.

The environment builds the pinned PHP 8.4.26 configuration and exposes the target application on port `8088`.

```bash
cd lab
docker compose up -d
```
<img width="457" height="61" alt="image" src="https://github.com/user-attachments/assets/4ce36667-064f-444d-b414-e16e28cc9570" />

### 2. Verify the Server Config

```bash
docker compose exec target apache2ctl -M | grep php
```
<img width="616" height="85" alt="image" src="https://github.com/user-attachments/assets/67d5cf8f-2ad2-449e-b571-20580b66f5c7" />

### 3. Executing Exploit

The exploit can either be run separately or launched using an .sh file.

```bash
python3 php2shell_poc.py --target http://127.0.0.1:8088/
```
```bash
./run-linux.sh
```
<img width="620" height="397" alt="Screenshot 2026-11-03 203310" src="https://github.com/user-attachments/assets/89383acf-dbf0-4d18-a7a2-3369d6392028" />

The PoC handles the exploitation process, including heap grooming, memory leak extraction for ASLR bypass, and payload delivery.

### 4. Verification of Exploitation Success

If the exploit succeeds the status page is redirected to another page with a custom php file which is uploded after RCE is achieved. The status page keeps checking if the exploit executes and then redirects it to the uploads/p.php.

<img width="1275" height="613" alt="image" src="https://github.com/user-attachments/assets/fc0bdbf4-d2b1-447a-9950-f0e8dbff4f36" />

<img width="1862" height="928" alt="Screenshot 2026-11-03 203956" src="https://github.com/user-attachments/assets/82fc6131-2d7a-47ca-9906-817611d94135" />

## Technical Writeup

Detailed explanation of the vulnerability and its exploitation mechanics is below.

PHP2Shell is a pre-authentication Remote Code Execution (RCE) vulnerability affecting PHP, caused by improper handling of Out-of-Memory (OOM) conditions during multipart HTTP request processing. The vulnerability originates from the PHP interpreter's nonlocal bailout mechanism, which uses `longjmp` to interrupt normal execution when a fatal error occurs, such as exceeding the configured memory limit. Under certain conditions, this abrupt termination bypasses normal cleanup and rollback operations, leaving stale pointers and partially initialized structures within the request state.

The exploitation scenario specifically targets Apache `mod_php` and abuses the `session.upload_progress` functionality. An important distinction exists between Apache `mod_php` and PHP-FPM: PHP-FPM terminates the affected worker following certain fatal startup errors, whereas the Apache `mod_php` execution path can leave the Apache child process alive after request memory has been released. This creates an opportunity for subsequent requests to interact with stale session pointers.

## Vulnerability Mechanism and Exploitation

The exploit begins by manipulating multipart HTTP request processing to trigger an OOM condition at a carefully selected point. PHP performs an initial `post_max_size` check against the declared `Content-Length`, but the demonstrated attack uses a multipart request whose consumed body exceeds the configured `memory_limit`. During file-upload processing, the `session.upload_progress` functionality initializes session state and stores pointers within the request heap. When the memory limit is exceeded, PHP performs a nonlocal bailout, interrupting normal execution and leaving session-related pointers referencing memory that is subsequently freed.

A second multipart request is then used to reach the upload-progress cleanup path without including an actual file-upload part. This causes PHP to process the previously retained session state and access freed memory, resulting in a Use-After-Free (UAF) condition. Through controlled heap manipulation, the Proof of Concept (PoC) attempts to reuse the freed memory and obtain memory-address disclosures through PHP error responses. These disclosures provide the heap and library addresses required to bypass Address Space Layout Randomization (ASLR). With the relevant addresses established, the exploit proceeds to manipulate a function pointer associated with `zif_system`, ultimately achieving controlled command execution.

The reproduction was performed against a pinned `php:8.4.26-apache` Docker image in an isolated laboratory environment. The Python PoC, `Exploit.py`, was calibrated against this specific build using hardcoded memory offsets. The laboratory configuration enabled error disclosure to demonstrate the memory-leak stage, while Apache logs provided additional evidence of heap corruption and worker termination.

The PoC demonstrated the exploitation process through three stages: obtaining the PHP heap address, extracting the PHP library base address, and delivering the final payload. The fixed demonstration command, `cp /a /u/p.php`, was executed within the container to establish proof of code execution. Subsequent HTTP requests confirmed that the resulting PHP file was processed by the server rather than returned as plain text. Apache error logs also recorded Zend Memory Manager corruption messages and worker termination events during the exploitation process.

The reproduction demonstrates the `session.upload_progress` dangling-state vulnerability, an in-band memory-disclosure-based ASLR bypass, and controlled command execution. It does not demonstrate the alternative `_zend_hash_str_add_or_update_i` interruption path, the PHP-FPM stale unserializer path, or a generic reverse-shell payload. Furthermore, the demonstrated ASLR bypass depends on error disclosure being enabled (`display_errors=1`), meaning the same leakage technique would not directly apply to hardened deployments configured with `display_errors=0`.

## References

- Calif PHP2Shell Research: https://calif.io/php2shell/
- NIST CVE-2026-109618: https://nvd.nist.gov/vuln/detail/cve-2026-109618
- Upstream PoC: https://github.com/eness/php2shell-CVE-2026-109618-CVE-2026-109619-PoC
- SAHMSEC Classroom Lab: https://github.com/sahmsec/CVE-2026-109618-PHP2SHELL
- LiveOverflow: https://www.youtube.com/watch?v=7CiZmq2llDg


All reproduction, testing, and validation activities documented in this repository were conducted within a closed and isolated laboratory environment. The project is intended to support vulnerability understanding, exploit analysis, and controlled security research.


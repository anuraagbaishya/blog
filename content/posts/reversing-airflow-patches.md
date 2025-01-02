---
title: "Unpatching Airflow: Turning CVE Fixes into Exploitable PoCs"
date: 2024-10-29T00:08:05-07:00
draft: true
toc: true
images:
tags:
  - cve-analysis
  - python
---
I have been reading quite a few blogs about analyzing patches for CVEs and generating exploits based on the diff. One of my favorites is [this blog](https://www.assetnote.io/resources/research/two-bytes-is-plenty-fortigate-rce-with-cve-2024-21762) from the folks at Assetnote about a CVE in Fortigate. While this level of reverse engineering is waaaay beyond what I can even think of attempting, I also read articles about simpler exercises, such as [this](https://blog.securelayer7.net/arbitrary-code-execution-in-apache-airflow/) for a CVE in Apache Airflow - which is a direct inspiration for this blog.

[Apache Airflow](https://airflow.apache.org/) is an open-source platform for creating, scheduling, and managing complex workflows and data pipelines programmatically. Apache Airflow uses [Directed Acyclic Graphs](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html) (DAGs) to define and schedule workflows as a series of interdependent tasks, ensuring a clear execution order without cycles. DAGs are Python-based, allowing dynamic task definitions and flexible scheduling. DAGs are defined in `$AIRFLOW_HOME/dags` folder. Airflow also has a plugin manager built-in that can integrate external features to its core by adding files in the `$AIRFLOW_HOME/plugins` folder.

Looking at the list of Airflow CVEs, I settled for 2 CVEs to explore: 
* [CVE-2024-45034](https://www.cvedetails.com/cve/CVE-2024-45034/) - A arbitrary code execution issue.
* [CVE-2024-41937](https://www.cvedetails.com/cve/CVE-2024-41937/) - A XSS issue.

While I was able to exploit CVE-2024-45034, I was not able to exploit CVE-2024-41937. There seemed to be additional controls that prevented the exploitation of CVE-2024-41937 (or I don't know how to exploit XSS - which, honestly, is quite probable - given complex XSS payloads can get). I will still share my learnings as this blog serves as a structured version of my notes from my ongoing journey in security research.

## CVE-2024-45034
[CVE-2024-45034]is an arbitrary code execution vulnerability which affects Airflow < 2.10.1. The description states the CVE "allows DAG authors to add local settings to the DAG folder and get it executed by the scheduler, where the scheduler is not supposed to execute code submitted by the DAG author". The patch for this CVE is added in [this pull request](https://github.com/apache/airflow/pull/41672). The changes are made to [settings.py](https://github.com/apache/airflow/blob/2.9.3/airflow/settings.py). The patch appears quite simple: 
* In the vulnerable version, the `DAGS_FOLDER` and the `PLUGINS_FOLDER` are added to `sys.path` before executing `import_local_settings`.
* In the patched version, the `DAGS_FOLDER` and the `PLUGINS_FOLDER` folder are added to `sys.path` after executing `import_local_settings`.

It can be guessed that `$AIRFLOW_HOME/dags` maps to `DAGS_FOLDER` and `$AIRFLOW_HOME/plugins` maps to `PLUGINS_FOLDER`. It can be inferred from the patch that something interesting happens in `import_local_settings` which deserves deeper investigation. Looking at [`import_local_settings`](https://github.com/apache/airflow/blob/81845de9d95a733b4eb7826aaabe23ba9813eba3/airflow/settings.py#L474) we see this code:
```Python
  def import_local_settings():
    """Import airflow_local_settings.py files to allow overriding any configs in settings.py file."""
      try:
          import airflow_local_settings
      ...
```
In the vulnerable version, it can be seen that since the `DAGS_FOLDER` and the `PLUGINS_FOLDER` are appended to `sys.path` before executing `import_local_settings`. Therefore, adding a file named `airflow_local_settings.py` to one of these folders will lead to it being executed by `import_local_settings`. This would lead to arbitrary code execution. The `DAGS_FOLDER` and the `PLUGINS_FOLDER` are meant to be writable by the users - thereby allowing any user to exploit this vulnerability by adding malicious code to a file named `airflow_local_settings.py` in either the `DAGS_FOLDER` or the `PLUGINS_FOLDER`

I created a simple `airflow_local_settings.py` file in my `$AIRFLOW_HOME/dags` folder with this line `print("This code runs when the module is imported.")`. I then restarted Airflow. Looking at the logs, I could see that this line was printed when the containers started. The CVE description states that the code will be executed by the scheduler. However, I noticed it was executed by the webserver, the worker, and the triggerer as well.

![local-settings-imported](/airflow-rev-1.png)

![code-exec-1](/airflow-rev-2.png)

![code-exec-2](/airflow-rev-3.png)
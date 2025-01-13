---
title: "Not a CVE: Using Airflow via Docker Can Lead to Arbitrary Code Execution"
date: 2025-01-12T00:08:05-07:00
draft: false
toc: true
images:
tags:
  - cve-analysis
  - python
---
I have been reading quite a few blogs about analyzing patches for CVEs and generating exploits based on the difference in code between the vulnerable and the fixed versions (`diff`). One of my favorites is [this blog](https://www.assetnote.io/resources/research/two-bytes-is-plenty-fortigate-rce-with-cve-2024-21762) from the folks at Assetnote about a CVE in Fortigate. While this level of reverse engineering is waaaay beyond what I can even think of attempting, I also read articles about simpler exercises, such as [this](https://blog.securelayer7.net/arbitrary-code-execution-in-apache-airflow/) for a CVE in Apache Airflow - which is a direct inspiration for this blog.

[Apache Airflow](https://airflow.apache.org/) is an open-source platform for creating, scheduling, and managing complex workflows and data pipelines programmatically. Apache Airflow uses [Directed Acyclic Graphs](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html) (DAGs) to define and schedule workflows as a series of interdependent tasks, ensuring a clear execution order without cycles. DAGs are Python-based, allowing dynamic task definitions and flexible scheduling. DAGs are defined in `$AIRFLOW_HOME/dags` folder. Airflow also has a plugin manager built-in that can integrate external features to its core by adding files in the `$AIRFLOW_HOME/plugins` folder.

Looking at the list of Airflow CVEs, I settled on exploring [CVE-2024-45034](https://www.cvedetails.com/cve/CVE-2024-45034/) which is an arbitrary code execution issue.

## CVE-2024-45034
[CVE-2024-45034](https://www.cvedetails.com/cve/CVE-2024-45034/) is an arbitrary code execution vulnerability which affects Airflow < 2.10.1. The description states the CVE "allows DAG authors to add local settings to the DAG folder and get it executed by the scheduler, where the scheduler is not supposed to execute code submitted by the DAG author". The patch for this CVE is added in [this pull request](https://github.com/apache/airflow/pull/41672). The changes are made to [settings.py](https://github.com/apache/airflow/blob/2.9.3/airflow/settings.py). The patch appears quite simple: 
* In the vulnerable version, the `DAGS_FOLDER` and the `PLUGINS_FOLDER` are added to `sys.path` before executing `import_local_settings`.
* In the patched version, the `DAGS_FOLDER` and the `PLUGINS_FOLDER` folder are added to `sys.path` after executing `import_local_settings`.

One can make an educated guess that `$AIRFLOW_HOME/dags` maps to `DAGS_FOLDER` and `$AIRFLOW_HOME/plugins` maps to `PLUGINS_FOLDER`. It can be inferred from the patch that something interesting happens in `import_local_settings` which deserves deeper investigation. Looking at [`import_local_settings`](https://github.com/apache/airflow/blob/81845de9d95a733b4eb7826aaabe23ba9813eba3/airflow/settings.py#L474) we see this code:
```Python
  def import_local_settings():
    """Import airflow_local_settings.py files to allow overriding any configs in settings.py file."""
      try:
          import airflow_local_settings
      ...
```
In the vulnerable version, it can be seen that since the `DAGS_FOLDER` and the `PLUGINS_FOLDER` are appended to `sys.path` before executing `import_local_settings`. Therefore, adding a file named `airflow_local_settings.py` to one of these folders will lead to it being executed by `import_local_settings`. This would lead to arbitrary code execution. The `DAGS_FOLDER` and the `PLUGINS_FOLDER` are meant to be writable by the users - thereby allowing any user to exploit this vulnerability by adding malicious code to a file named `airflow_local_settings.py` in either the `DAGS_FOLDER` or the `PLUGINS_FOLDER`

I created a simple `airflow_local_settings.py` file in my `$AIRFLOW_HOME/dags` folder with this line `print("This code runs when the module is imported.")`. I then restarted Airflow. Looking at the logs, I could see that this line was printed when the containers started. The CVE description states that the code will be executed by the scheduler. However, I noticed it was executed by the webserver, the worker, and the triggerer as well. More on this later.

![local-settings-imported](/airflow-rev-1.png)

![code-exec-1](/airflow-rev-2.png)

![code-exec-2](/airflow-rev-3.png)

## Testing on a fixed version
After reproducing the CVE in Airflow 2.9, I decided to see what happens if I try the same thing on a patched version. So I set up Airflow 2.10.1 using [the Docker setup instructions](https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html). Then I performed the same steps I tried with 2.9. I observed that the code from `import_local_settings` is still executed by the scheduler. It was however not run from the webserver, the worker, and the triggerer as with 2.9.

As the CVE specifically mentioned the scheduler as the component that runs the arbitrary code and this also happened in the fixed version, I thought the CVE was not completely fixed. Therefore I reported this to Apache Airflow security.

## Response from Apache
The Apache security team was very prompt in their response. They let me know within a few days that my finding was invalid. They told me in polite words to RTFM. In my excitement to report my findings, I had neglected to read the [security model documentation for Apache Airflow](https://airflow.apache.org/docs/apache-airflow/stable/security/security_model.html), specifically this line "In case of Local Executor and DAG File Processor running as part of the Scheduler, DAG authors can execute arbitrary code on the machine where scheduler is running". As I was running my Airflow through Docker, this statement applies to my setup. This was also why I noticed that for me the arbitrary code was running on all containers and not just the scheduler.

Between the time of me reporting this and now, it appears that [the Docker setup instructions](https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html) page has been updated to show a warning at the top stating that this way of using Airflow provides no security guarantees and therefore should not be used in production. They recommend using a [Kubernetes setup](https://airflow.apache.org/docs/helm-chart/stable/index.html) instead.

## Conclusion
This was an entertaining exercise. Reverse engineering a patch was quite fun. The added excitement that I may have found another CVE was quite a highlight, albeit only for a few days. However the main lesson I learnt from this excursion was that I should read the documentation properly before reporting clearly documented security risks as new issues to PSIRTs. I myself work on PSIRT and encounter reports quite often where I have to inform reporters what they are describing is a misconfiguration or a known documented risk. So it is ironic that I don't follow my own advice. Oh well, you live and you learn.

Thank you for reading, and Happy New Year!
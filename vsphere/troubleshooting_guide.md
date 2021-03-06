---
title: Troubleshooting Guide and Common Issues
---

This guide describes strategies for troubleshooting vSphere Cloud Foundry
deployments and lists a few common deployment issues.

## <a id="prerequisite"></a>Prerequisites ##
* Cloud Foundry is being deployed by BOSH
* Knowledge of the `bosh ssh` command. See the [BOSH CLI](/bosh/bosh-cli.html) documentation for more information.

## <a id="configuration"></a>Configuration Files ##
When BOSH deploys each Cloud Foundry job, it populates configuration files that live on the job's VM.
BOSH populates the configuration files based on information from your deployment manifest file.

We recommend checking these configuration files if you see an error during
the Cloud Foundry deployment.

To access the job VM, use `bosh ssh`.

The job configuration files are located at `/var/vcap/data/jobs/<job>/<number>/config/`,
where `<job>` and `<number>` are the associated values from the deployment
manifest (e.g. `cloud_controller_ng` and `0`).

Notably, the `<job>.yml` file will hold the information from your
deployment manifest. Check to see that all of these values have the proper
format and content to match your target environment.

## <a id="logs"></a>Log Files ##
Each Cloud Foundry job has a set of log files that the job constantly updates.

These log files are located on every job VM at `/var/vcap/sys/log/<job>/`.
The primary log file is named `<job>.log`.

To see when and where errors are occurring, run the following command while deploying Cloud
Foundry:
<pre class='terminal'>
$ tail -F `<job>.log`
</pre>
This will follow the log file and display new lines as the job updates the file.

## <a id="issues"></a>Common Issues ##
This is a list of common errors during Cloud Foundry deployments, along with some
steps to help resolve the errors.

**Issue:** During deployment, the following error is displayed:
<pre class='terminal'>
Temporary issue resolving host abc.domain.com
</pre>
This is a DNS issue stating that BOSH cannot connect to the listed host.

**Troubleshooting:**

* Check that all of the jobs are being deployed to the same network
* Check that the VMs each have access to the DNS (pings are successful)
* Check that BOSH has access to the DNS
* Check that the DNS is working correctly
	* Contact your DNS administrator
<hr>

**Issue:** During deployment, the following error is displayed:
<pre class='terminal'>
`<job>/0` is not running after update
</pre>
This means that BOSH did not get a reply from the job after sending the 'start'
command.

**Troubleshooting:**

* Sometimes jobs take longer than allowed to start. If this is the case, try running 'bosh deploy' again. The issue may resolve itself.
* Check that there is a wildcard entry in the DNS for the Cloud Foundry domain. For example, there should be an entry `*.cloudfoundry.local` that points to the router's IP address.
* Check the job's configuration files and logs. The job could be getting configured incorrectly. See the above section for instructions on accessing log and configuration files.
<hr>

**Issue:** Before a deployment, the following error is displayed:
<pre class='terminal'>
Error 100: Bosh::Director::Lock::TimeoutError
</pre>
BOSH creates a lock so that only one process may modify a Cloud Foundry
deployment at a time. This error means that BOSH is busy doing something else.

**Troubleshooting:**

* Wait one minute or until the current job finishes. The lock will automatically be removed when the current job finishes.
* Kill the BOSH process that created the lock. This is only recommended if the lock should not exist.

To kill the BOSH process:
<pre class='terminal'>
$ bosh tasks
$ bosh cancel task [task_number]
</pre>
The first command lists the tasks that BOSH is currently performing, along
with numbers associated with each task.
The second command kills the selected task once the task reaches a
checkpoint. It does not revert changes.
<hr>
# Dataproc for Data Scientists: Spark + Zeppelin + Jupyter = <3

## TOC

### Requirements
- Active Google Cloud Platform Subscription (Free Trial)
- Installed and authenticated `gcloud` command line tool

### Get a Dataproc cluster up and running
- [ ] Google Cloud SDK `gcloud` 
- [x] Cloud Shell

```
#! /bin/bash
gcloud dataproc --region {CLUSTERREGION} clusters create {CLUSTERNAME} \
--subnet default --zone {CLUSTERZONE} --master-machine-type {MASTERMACHINETYPE} \ 
--master-boot-disk-size 500 --num-workers {CLUSTERWORKERSNUMBER} \ 
--worker-machine-type {WORKERMACHINETYPE} --worker-boot-disk-size 500 \
--project {PROJECTID} \
--initialization-actions \ <!-- Initialization Scripts -->
'gs://dataproc-initialization-actions/zeppelin\zeppelin.sh', \
'gs://dataproc-initialization-actions/jupyter/jupyter.sh'

```

#### Quick word on --initialization-actions
The `--initialization-actions` flag is used to define a list of Google Cloud Storage link that refers to shell files containing instructions on how to setup Masters and workers. This is very similar to a Docker configuration file. When creating the Cluster you can pass more than one link to the flag, this is useful if, like in our case you want to setup more than one service beyond the ones provided by default by Dataproc. In the script above we pass both the Jupyter and Zeppelin instructions, they are taken from the ones made publicly available by Google itself. You can find more info on --initialization-actions [here](https://cloud.google.com/dataproc/docs/concepts/configuring-clusters/init-actions).

### Connecting to the cluster
#### NOTE: Behind a Corporate Firewall

Since connecting to the service requires a functioning SSH tunnel, you will thus need to perform some more step.

- Reserve one static IP for each clusters you plan on frequently deploy
- Ask your IT department to add the static IP to the Firewall whitelist enabling SSH connectivity between your local machine and the cluster's master node.
- Assign the static IP to the master machine via the Compute Engine panel.

Once the IP is assigned and the Firewall has been opened you can move to the next step.

#### Of Proxies and Socks

*The Cloud Shell supports standard SSH tunneling, however in our experience the services running on the cluster, like Zeppelin and Jupyter if used with tunneling cannot be reached when connecting. It is thus recommended to connect via the SOCKS protocol as described below. - Michele "Ubik" De Simoni - 09/11/2017*

>
To connect to the web interfaces, the best practice is to use an SSH tunnel to create a secure connection to your master node. The SSH tunnel supports traffic proxying using the SOCKS protocol. This means that you can send network requests through your SSH tunnel in any browser that supports the SOCKS protocol. This method allows you to transfer all of your browser data over SSH, eliminating the need to open firewall ports to access the web interfaces.

>
Connecting to the web interfaces with SSH and SOCKS is a two-step process:

>
- Create an SSH tunnel. Use an SSH client or utility to create the SSH tunnel. Use the SSH tunnel to securely transfer web traffic data from your computer's web browser to the Cloud Dataproc cluster.
- Use a SOCKS proxy to connect with your browser. Configure your browser to use the SOCKS proxy. The SOCKS proxy routes data intended for the Cloud Dataproc cluster through the SSH tunnel.

##### Step 1 - Create an SSH tunnel
- [x] Google Cloud SDK `gcloud` 
- [ ] Cloud Shell
>
We recommend using the Google Cloud SDK `gcloud compute ssh` command to create an SSH tunnel with arguments that enable dynamic port forwarding.

>
Run the following command to set up an SSH tunnel to the Hadoop master instance on port 1080 of your local machine. Note that 1080 is an arbitrary but typical choice since it is likely to be open on your local machine. Replace master-host-name with the name of the master node in your Cloud Dataproc cluster and master-host-zone with the zone of your Cloud Dataproc cluster.

```
gcloud compute ssh --zone=master-host-zone master-host-name -- -D 1080 -N -n
```
>
The -- separator allows you to add SSH arguments to the gcloud compute ssh command, as follows:

>
-Dspecifies dynamic application-level port forwarding. Port 1080 is shown in the example, but another available port on your local machine can be used.
-N instructs gcloud not to open a remote shell.
-n instructs gcloud not to read from stdin.

*Apparently the -n flag is not supported when SSHing from Windows using PuTTY, you can safely remove it. - Michele "Ubik" De Simoni - 09/11/2017*

>
This command creates an SSH tunnel that operates independently from other SSH shell sessions, keeps tunnel-related errors out of the shell output, and helps prevent inadvertent closures of the tunnel.

**NOTE:** It is okay if you do not see any output in the SSH console. That is the expected behavior, you can move to Step 2.

##### Step 2 - Connect with your web browser
>
Your SSH tunnel supports traffic proxying using the SOCKS protocol. You must configure your browser to use the proxy when connecting to your cluster.

>
The application (executable) location of your browser on your machine/device depends on its operating system. The following are standard Google Chrome application locations for popular operating systems:


|Operating System|Google Chrome Executable Path |
|----------------|:-----------------------------|
|Linux           |/usr/bin/google-chrome        |
|Windows         |C:\Program Files (x86)\Google\Chrome\Application\chrome.exe|
|Mac OS X	     |/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome|

>
To configure your browser, start a new browser session with proxy server parameters. Here's an example that uses the Google Chrome browser:

**LINUX**
```
/usr/bin/google-chrome \
  --proxy-server="socks5://localhost:1080" \
  --host-resolver-rules="MAP * 0.0.0.0 , EXCLUDE localhost" \
  --user-data-dir=/tmp/master-host-name
```

**WINDOWS**
```
"C:\Program Files (x86)\Google\Chrome\Application\chrome.exe" \
  --proxy-server="socks5://localhost:1080" \
  --host-resolver-rules="MAP * 0.0.0.0 , EXCLUDE localhost" \
  --user-data-dir="./tmp/master-host-name"
```

This command uses the following Google Chrome flags:

- `--proxy-server="socks5://localhost:1080"` tells Chrome to send all http:// and https:// URL requests through the SOCKS proxy server localhost:1080, using version 5 of the SOCKS protocol. Hostnames for these URLs are resolved by the proxy server, not locally by Chrome.
- `--host-resolver-rules="MAP * 0.0.0.0` , EXCLUDE localhost" prevents Chrome from sending any DNS requests over the network.
- `--user-data-dir=/tmp/host-master-name` forces Chrome to open a new window that is not tied to an existing Chrome session. Without this flag, Chrome may open a new window attached to an existing Chrome session, ignoring your --proxy-server setting. The value set for --user-data-dir can be any nonexistent path. If you stumble upon an error where Chrome cannot create the target directory than try changing it. Personally, both on Linux and on Windows, I use the same value: `./dataproc-chrome-proxies-tmp/{MASTER-HOST-NAME}` 

### Dataproc Web UI

Once you are connected to via the SOCKS Proxy, you will be able to access each service exposing a web UI on Dataproc by going to `http://master-host-name:SERVICE-PORT`

|Web UI					|PORT| 
|-----------------------|:---|
|YARN ResourceManager	|8088|
|HDFS NameNode			|9870|
|Jupyter				|8123|
|Zeppelin				|8080|


### Resources
- [Initialization actions](https://cloud.google.com/dataproc/docs/concepts/configuring-clusters/init-actions)
- [Cluster web interfaces](https://cloud.google.com/dataproc/docs/concepts/accessing/cluster-web-interfaces)
- [Google Cloud Platform for data scientists: using Jupyter Notebooks with Apache Spark on Google Cloud](https://cloud.google.com/blog/big-data/2017/02/google-cloud-platform-for-data-scientists-using-jupyter-notebooks-with-apache-spark-on-google-cloud)
- [Introduction to Apache Spark and Zeppelin on Google Cloud Dataproc](https://cloudacademy.com/blog/big-data-using-apache-spark-and-zeppelin-on-google-cloud-dataproc/)
# Working with Red Hat Subscription Data

## 1.How to configure the collection tool – CRHC-cli:

```
# Git the code!
$ git clone https://github.com/C-RH-C/crhc-cli.git
 
# Make sure your version is 3.8 or 3.9. I had issues with 3.10
$ python3 -V
 
# Create a python virtual environment to store our required python modules
$ python3 -m venv ~/.virtualenv/crhc-cli
 
# Start the virtual environment
$ source ~/.virtualenv/crhc-cli/bin/activate
$ cd crhc-cli
 
# Update pip
$ pip install --upgrade pip
 
# Install the required python modules
$ pip install -r requirements.txt
 
# We can now run the crhc.py script!
# Leave this shell window open as we will return here to run the script after we complete a
# few more steps.
```
 ## 2. Create an Offline Access Token for Authentication

Newer releases of the CRHC-cli tool no longer support password authentication, and an offline access token must be used to connect to the portal.  This provides a nicer interface for applications to connect and access the portal programmatically.  The instructions below walk you through the process of creating an offline access token.
* NOTE:  
  * Your portal account must have subscription view/renew rights in order to generate an access token.  If you are unsure what this is please reach out to one of your Red Hat administrators and have them check your portal account.  If you are unsure who your Red Hat administrator is please reach out to your Red Hat account team as we can help to identify a person on your team to assist.
  * Offline access tokens expire automatically after 30 days if they are unused.  If you cannot connect using your offline access token you might want to follow the steps below to create a new access token.
  * The access token is like a password and should be stored in a secure location.  I recommend using a password vault so you can retrieve the taken when it is needed.  Please do not share your token with anyone.

#### Creating your access token:
* Login to your portal account here to create the token: [https://console.redhat.com/openshift/token](https://console.redhat.com/openshift/token)
* Click on the "Load Token" button
* Copy your new token and save it to a safe place. It will be required to connect to the [cloud.redhat.com](cloud.redhat.com) API interface. I put my token into Lastpass for safe retrieval.
  * NOTE: This token must be secured and should not be shared. It is basically a password replacement that will connect you to the API interface for your Red Hat portal account.

## 3. Running the crhc.py script to get your data extract
Now, it is time to run the script for the first time.  We are going to return to our shell window to run the crhc.py script using our new access token and get our first data extract from the Red Hat portal.

Return to your shell windows from step 1
```
# Login to the portal with crhc.py
$ ./crhc.py login --token <Put your offline access token string here>
 
# Run the "troubleshoot match" command to get the extract files we need:
$ time ./crhc.py ts match
 
# I like to time the execution of the above command so I have an idea of the duration. This 
# command can take a considerable amount of time to run. I have seen execution times from 10 
# min to 1 hr. Be patient
 
# The output files will be dropped into /tmp
```
That is it!  The script is running, and it will extract the data from the portal, and it will also combine two of the data files into a CSV file that we can load into a Google Sheet for viewing.

## 4. What are the output files?
* ***/tmp/inventory.json*** - this is the raw export from the Insights Inventory page. We will not use this.
* ***/tmp/swatch.json*** - this is the faw export from the Insights Subscriptions app. We will not use this.
* ***/tmp/match_inv_sw.csv*** - the crhc.py script combines and flattens the above two files into a CSV file so we can view and process the data. This is our main file that will be used to create our reports.
* ***/tmp/issue_summary.log*** - this file is also very important to us. Any issues with the data are called out in this file. Duplicate entries and systems that are flagged as Satellite/Capsule/OpenShift servers with no packages installed are found in this file. A common issue that I have seen with many customers are servers that are incorrectly flagged as Satellite or Capsule servers. This can happen if the Satellite or Capsule repos are presented to servers inadvertently. In such cases, the Satellite, Capsule, or Other product certs can get installed on a system, and that system will no longer report as a RHEL server. These issues must be corrected in order to get an accurate count.

## 5. Examining the /tmp/issue_summary.log file for issues
There are several data issues that will be reported in this file.  The big ones are the following:
1. Wrong Sockets in Inventory
2. Wrong Sockets in Subscriptions
3. Duplicate FQDN
4. Duplicate Display Names
5. Different FQDN and Display Name)
6. Server with no socket_key
7. Installed Product with no Installed Package (Satellite, Capsule, OpenShift

I will only focus on three of the items above.  The two forms of duplicates and the last item, Installed Product with no Installed Packages.  Duplicates can throw off an inventory study since you may have “ghost” systems being reported.  Depending on the number of duplicates your deployment utilization may be off from a small amount to a large amount, so it is worth cleaning this up.  How do such duplicates get created?  They tend to stem from the build/provisioning process.  For example, let's say that a process is started to build a new server and register that server to the portal or Satellite.  Now, let’s imagine that a problem is discovered with the build, and a new build process is initiated without first cleaning up the portal/Satellite registration record.  When the rebuild process is complete and the registration happens again, a new ID for the server will be generated, and the box will now show up as a duplicate.  Same hostname, but two different IDs.  Once registration is complete, a decommissioning process should be followed to unregister the host to prevent duplicates.  I suggest automating such processes with the Ansible Automation Platform!

## 6. /tmp/match_inv_sw.csv

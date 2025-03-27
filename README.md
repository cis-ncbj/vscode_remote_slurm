# vscode_remote_slurm
Helper script for executing commands before connecting to vscode remote. This can be used to run vscode remote on the compute node of a slurm cluster.  
Conditionally wraps the ssh command if `salloc` is in the RemoteCommand. Passes through otherwise.  
This seems to work for Mac + Linux + Windows!

### Changelog:
2024-03-25:  
Notable changes in this version (cancel-jobs-better):  
- Whole process requires less ssh connections so connecting is quicker.  
- Cancels job on window close/disconnect (after `SCANCEL_TIMEOUT` has elapsed).  
- Reconnects to your job if it finds it running on connecting to the cluster. Say you accidentally closed your vscode window or lost connection, you have `SCANCEL_TIMEOUT` seconds to reconnect before your vscode slurm job is scancelled.  
  
### Requirements
- Key to be added to ssh-agent so agent forwarding works.
- The defined host in the ssh config must not be using Control Sockets.

### How I have been able to get this working:  
## Linux/macOS
- Put the ssh_wrapper.sh script somewhere.
- Make sure it's executable: `chmod +x ssh_wrapper.sh`
- Change vscode to run this instead of your default ssh binary (ctrl + shift + p -> Remote-SSH: Settings -> Remote.SSH: Path: `/full/path/to/ssh_wrapper.sh`)
- Create a host entry in your ssh_config (example below) with a RemoteCommand detailing your resources.
- Hope it works?

## Windows
- Ensure the Openssh Agent Service is installed and running (or use pageant), and add your key to ssh-agent: `ssh-add /path/to/your/key` (I have also had issues where the key has to be in the top 6 keys in the ssh-agent :shrug: It seems IdentitiesOnly isn't being respected???)
- Put the ssh_wrapper.ps1 script somewhere.
- Put that location in the ssh.bat file (the default location is currently set to be ~\ssh_wrapper.ps1 in the ssh.bat file)
- Change vscode to run ssh.bat instead of your default ssh binary (ctrl + shift + p -> Remote-SSH: Settings -> Remote.SSH: Path: `/full/path/to/ssh.bat`)
- Create a host entry in your ssh_config (example below) with a RemoteCommand detailing your resources.
- Hope it works?



### How it works
The script:

- Pretends to be ssh and intercepts the ssh commands sent from vscode,
- Uses salloc to reserve resources on the cluster (currently set in the RemoteCommand in the ssh_config),
- Figures out where those resources are,
- Proxyjumps through the login node and runs bash within the Slurm allocation using srun,
- Allows vscode to continue to send its commands to the bash shell on the compute node to run the remote server.

### TODO:  
- Wrap into extension so it runs this script on a button press instead of changing vscode to only use this script for ssh.
- Check for job already running and reconnect.


Notes:
I have tested this on a Mac M1 connecting to a Centos 7 Slurm Cluster. Vscode Insiders v1.83 and Remote - SSH v0.106.4.  
It hasn't been tested on anything else yet.  

These are my Remote SSH settings:
```
    "remote.SSH.connectTimeout": 60,
    "remote.SSH.logLevel": "trace",
    "remote.SSH.showLoginTerminal": true,
    "remote.SSH.useExecServer": true,
    "remote.SSH.maxReconnectionAttempts": 0,
    "remote.SSH.enableRemoteCommand": true,
    "remote.SSH.useLocalServer": true,
    "remote.SSH.localServerDownload": "off",
    "remote.SSH.configFile": "/full/path/to/vscode-config",
    "remote.SSH.path": "/full/path/to/ssh_wrapper.sh", # or ssh.bat on windows
```


Define your ssh connection in a `ssh_config` file, preferably distinct from the default one, like so with your desired slurm allocation:
```
Host slurmusrint1.cis.gov.pl
   HostName slurmusrint1.cis.gov.pl
   User username
   RequestTTY yes
   ForwardAgent yes
   GSSAPIAuthentication=yes
   GSSAPIDelegateCredentials=yes
   GSSAPITrustDNS=yes
   RemoteCommand salloc --no-shell -n 1 -c 4 -J vscode_interactive_job --time=12:00:00 --qos=interq -p INTEL_IVY,INTEL_SKYLAKE,INTEL_CASCADE,INTEL_HASWELL
```
  
I have put the jobname as `vscode_interactive_job` so that the script can find the single job to cancel if you disconnect.

Connect and hopefully it works.

### Troubleshooting:
- If you get an error about the ssh key not being found, make sure you have added it to the ssh-agent: `ssh-add /path/to/your/key`
- Check the script is executable: `chmod +x ssh_wrapper.sh`
- Check the `ssh_config` is correct and the RemoteCommand is correct.
- Remove the ~/.vscode-server on remotehost if you are having issues with the vscode server not starting.
- Kill the vscode server on remotehost if you are having issues with the vscode server not starting: `pkill -f vscode-server` (or by using ctrl + shift + p -> Remote-SSH: Kill VS Code Server on Host... -> remotehost)
- Kill the vscode server on your local machine if you are having issues with the vscode server not starting: `pkill -f vscode-server` (or by using ctrl + shift + p -> Remote-SSH: Kill VS Code Server on Host... -> local)

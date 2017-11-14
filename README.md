# WebSphere Scripts
Scripts for managing WebSphere Application Server

This repository contains my scripts to manage IBM WebSphere Application server installations.
I set it up to share my start and stop scripts that make sure the infrastructure is started up and shut down correctly on system restart but does not fail when sombody is manually starting or stopping a WebSpere Server or Node Agent.

# Why do I not uses systemd scripts directly to start or stop WebSphere
In the early days where SysVinit was the standard init-system for Linux, I just used the standard WASService command to creates servcies for all required processes (Dmgr, NodeAgent, Server, Proxy) on my system, set up the right boot order with the priority scheme and everything worked as intended.
Today, when systemd replaced the good old SysVinit, It is not that simple anymore.
There are a couple of guides out there on how ot start WebSphere using systemd unit file but none of them worked 100% for me.
## Example 1: Unit file using forking
    [Unit]
    Description=IBM WebSphere Application Server
    
    [Service]
    Type=forking
    ExecStart=/opt/IBM/WebSphere/AppServer/profiles/Dummy01/bin/startServer.sh server01
    ExecStop=/opt/IBM/WebSphere/AppServer/profiles/Dummy01/bin/stopServer.sh server01
    User=wasuser
    PIDFile=/opt/IBM/WebSphere/AppServer/profiles/Dummy01/logs/server01/server01.pid
    SuccessExitStatus=143 0
    [Install]
    WantedBy=default.target

This unit script works quite well when the service is started and stopped usind systemctl command. It detects the running WebSpehre process correctly and shuts it down when requested. (The credentials to stop the WebSphere process are in the soap.client.props so I do not need to add them here in clear text.)
This unit script fails, when a different process is starting or stopping the WebSphere process. It looses track of the process and shows it either as not started or failed. Further actions will not be possible as a not started or fails process will never run the stop operation. A start will not work when the WebSphere Server is already running as the start script will exit with a error other than 0 or 143. (usually 255 which also might indicated some other error.)

## Example 2: Unit file using oneshot
    [Unit]
    Description=IBM WebSphere Application Server
    
    [Service]
    Type=oneshot
    ExecStart=/opt/IBM/WebSphere/AppServer/profiles/Dummy01/bin/startServer.sh server01
    ExecStop=/opt/IBM/WebSphere/AppServer/profiles/Dummy01/bin/stopServer.sh server01
    User=wasuser
    [Install]
    WantedBy=default.target
This unit script works better as long as nobody is fiddling arround with the service to start or stop the WebSphere Service when it has a different state as systemd thinks, the state is. So after system start, the script is started and therefore WebSphere is stared. Now somebody stops WebSphere using the stopServer.sh script. According to sytemd the process is still running. Unsing "systemctl stop websphere" will fail as the stopServer.sh script run by systemd will fail as not WebSphere Server is running anymore. Especially when it comes to next system shutdown. When somebody stared WebSphere again but systemd still thinks, the service is not running, WebSpehre is not stopped by systemd.
The second disadvantage of this direct usage of this scripts is that you have to maintain the boot order inside the unit files. This is not funny to set up and to automate this process is also not that simple.

# The solution I choose
I decided to choose a boot system using multiple scritps that are documented in this git repository. The main advantage of this scrips are that they can detect what is installed on the system and can check the status of the WebSpehre process to act accordingly. 
1. startInfrastructure.sh / stopInfrastructure.sh
This script is the main script that coordinats the work to start / stop the infrastructure.
2. startAllNodes.sh / stopAllNodes.sh
This script starts / stopps all nodes on the system as they are registerd in the profileRegistry.xml
3. startAllServer.sh / stpoAllServers.sh
This script loops throuh all nodes on the system and starts / stopps all registered servers within this node.
4. unit file
The unit file calls the startInfrastructure.sh on startup and the stopInfrastructure.sh on shutdown.


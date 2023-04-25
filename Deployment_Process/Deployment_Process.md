# Deployment Process

## Finding the sybmolic link

- Login with putty on **in1-esg-ltmc002** server for ESGProcesses Dev Deployment with AD-DEV\a_username
- Change directory to cd */opt/data/tomcat*
    - Type `ll`
    ![cd_tomcat]
- We have to use **ESGProcesses**
- Now, change directory to cd ESGProcesses/conf/Catalina
- Type cat localhost/ESGProcesses.xml to view the xml file
![symbolic_link]
- So here the symbolic link name is **ESGProcesses-CURRENT.war**
- Symbolic link is basically the reference variable pointing to the current war in th Repo server
  
## Downloading the war and linking it

- Login with putty on **vpna-dev-rep001** Repo server with AD-DEV\a_username
- Change directory to cd */opt/data/httpd/htdocs/repository/CURRENT/GovernanceData/wars*
- Type `ll ESGP*` to view all the wars, here we can also see the the symbolic link
![wars_listing]
- Now, type `sudo -u webid -sH`, login with the same password as used for AD-DEV\a_username
- Now type, `wget -O ESGProcesses-SNAPSHOT_v2023-04-25.war https://artifactory.issgovernance.com/ui/native/iss-artifactory/GovernanceData/ESGProcesses/SNAPSHOT/ESGProcesses-SNAPSHOT.war` to download the war
    + `-O` is used to rename the war
    + Master's build is called SNAPSHOT
  
![downloading_war]

- Now, we need to unlink the previous war and link the current downloaded war
    + Type `unlink ESGProcesses-CURRENT.war`, this will unlink the war associated with the symbolic link
    + To link the current war type *ln -s (name of the war to be linked) (symbolic link name)*
    <br>
     `ln -s ESGProcesses-SNAPSHOT_v2023-04-25.war ESGProcesses-CURRENT.war`

![current_war_linked]


## Restarting Tomcat

- Login with putty on **in1-esg-ltmc002** server with AD-DEV\a_username
- Change directory to cd */opt/data/tomcat*
- Type `sudo systemctl restart tomcat@ESGProcesses`, this will restart the tomcat
    + *Note : After *@* we need to specify the folder which we use, so here it is **ESGProcesses***
- Now check the status of the server by typing, `sudo systemctl status tomcat@ESGProcesses`
![tomcat_status]



[cd_tomcat]: ./img/CD_Tomcat.png
[symbolic_link]: ./img/Symbolic_Link.png
[wars_listing]: ./img/Wars_Listing.png
[downloading_war]: ./img/Downloading_War.png
[current_war_linked]: ./img/Wars_Listing.png
[tomcat_status]: ./img/Tomcat_Status.png
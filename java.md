## Set JAVA_HOME

Run:
```
which java
```
`which java` often points to a _symlink_. To resolve the actual directory:
```
readlink -f $(which java)
```
Example output (typical for **OpenJDK images**):
```
/usr/lib/jvm/java-17-openjdk-amd64/bin/java
```
Set the value of `JAVA_HOME`: 
```
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
```
Add to `PATH` if needed:
```
export PATH=$JAVA_HOME/bin:$PATH
```
### If you want to persist JAVA_HOME in a Dockerfile

Add this to your Dockerfile:
```
ENV JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
ENV PATH="$JAVA_HOME/bin:$PATH"
```
### If you want to set and persist JAVA_HOME for all users in your system:
1. Open: 
```
vim /etc/profile
```
2. Press `i` to get in insert mode and add:
```
export JAVA_HOME="path that you found"
export PATH=$JAVA_HOME/bin:$PATH
```
3. logout and login again, reboot, or use `source /etc/profile` to apply changes immediately in your current shell.
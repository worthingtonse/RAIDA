# RAIDA
The Redundant Array of Independent Detection Agents

[Setup](README.md#setup)

[Start Up](README.md#start-up)

[Athenticating Coins](README.md#authentication-coins)

[Backups](README.md#backups)

[SkyVault](README.md#skyvault)

[Encryption](README.md#encryption)

# Setup
* Your System Administrator is named Fel and he will do any System Administratin jobs you need including deployment. 
* You may program on your computer but your repo will be put on a production server everyday. Otherwise you are welcome to program on the production server.
* The directory structure of the program can be found here: [Data Directory](https://github.com/worthingtonse/RAIDAX/blob/main/DATA%20DIRECTORY.MD#data-directory)
* Amol is the programmer who created the same code in the 'C' language. His job is to help you understand the code. 
* This is the Repo. 

# Startup
The RAIDA goes through its process of initialization This takes about 10 minutes depending on the Server's harddrive. 
1. The program starts and greets the user.
2. Configuration files are loaded: dns.bin, raida_no.txt, server.bin, shards.bin and version.txt
3. Data files are loaded for coin 0 (The identification coins). There are 1024 files that will be loaded into RAM.  each with  17 byte. You can see the structure of these [Coin Files](https://github.com/worthingtonse/RAIDAX/blob/main/DATA%20DIRECTORY.MD#0-identification-coin-backup-files). 
4. Data file are loaded for [Coin Files](https://github.com/worthingtonse/RAIDAX/blob/main/DATA%20DIRECTORY.MD#1-cloudcoin-backup-files). 
5. UDP Listner is started

# Authenticating Coins
The server listens to command that come from the client software. 
* Create the services that change ownership of coins: Detect, POWN (password own), PANG, ANPANG, Indentify.
* Create the services that syncronize the coins: Ticket

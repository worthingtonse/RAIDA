# RAIDA
The Redundant Array of Independent Detection Agents is a server that detects if CloudCoins are authentic or counterfeit. Here are the major parts of the developent of the server. The RAIDA server has already been written in ['C' Langauage](https://github.com/worthingtonse/RAIDAX/tree/main/code)

[Setup](README.md#setup) Preparing your developing environment

[Start Up](README.md#start-up) What happens when the server starts?

[Athenticating Coins](README.md#authenticating-coins) 

[Backups](README.md#backups) Backing up data that has changed.

[SkyVault](README.md#skyvault) The first Bank with Data Supremacy

[Encryption](README.md#encryption) AES

[Advanced Services](README.md#advanced-services)

# Setup
Before you can program, you will need to have your directory structures, config files, and data file ready. 
* System Administrator: Your system admin is Fel and Miroch. They will do any System Administration jobs that you need done so you can concentrate on programming.
* Repo: You may program on your computer but your repo will be put on a production server everyday. Otherwise you are welcome to program on the production server.
* Directory structure: Found here: [Data Directory](https://github.com/worthingtonse/RAIDAX/blob/main/DATA%20DIRECTORY.MD#data-directory)
* C Program: Amol is the programmer who created the same code in the ['C' Langauage](https://github.com/worthingtonse/RAIDAX/tree/main/code) His job is to help you understand the code. 
* Repo: This is the Repo that code will be stored on. It will be private.
* Testing: Testing tools have already been created and the server will be put into production so that errors can be detected. However, it is very important the the programmer be responsible for fully testing the code.

# Start Up
The RAIDA goes through its process of initialization This takes about 10 minutes depending on the Server's harddrive. 
1. Greeting: The program starts and greets the user.
2. Configuration files: Config files are loaded: dns.bin, raida_no.txt, server.bin, shards.bin and version.txt
3. Data files:  Data is loaded into RAM [Coin Files](https://github.com/worthingtonse/RAIDAX/blob/main/DATA%20DIRECTORY.MD#0-identification-coin-backup-files). 
4. RAM Tables: All data is kept in RAM except for backups. [RAM Tables](https://github.com/worthingtonse/RAIDAX/blob/main/Data_Tables.MD).


# Authenticating Coins
The server listens to command that come from the client software. 
1. UDP Listener: The server starts listening for client requests.
2. RAIDA Protocol Headers: Create the standard [request header](https://github.com/worthingtonse/RAIDAX/blob/main/Request_Response_Headers.md).
3. RAIDA Services: Create the services Echo and Version from the [Off Ledger Services](https://github.com/worthingtonse/RAIDAX/blob/main/OFF_LEDGER.md).


# Backups
The server must keep a log in RAM that tracks pages that have changed. Once enough pages have changed, the changes are written to file. 
1. Backup Pages: The data in the RAM is divided into "pages" that each have 1024 records. If on of the records is changed, the page needs to be backup to hard drive. A RAM table called "Archive" trackes which pages need to be backed up and the backups happen from time to time when needed. 

# Skyvault 
The Skyvault is a ledger-based bank that allows people to deposit, withdraw, transfer and do other bank functions. 
1. Create the [skyvault services](https://github.com/worthingtonse/RAIDAX/blob/main/Skywallet.md#all-services).
* RAM Tables: Load and backup additional tables. 

# Encryption
The program will use AES encryption. 
1. Implement the AES encryption on all services

# Advanced Services
1. Add the phase II services. 
2. Add the logs and statistics services. 


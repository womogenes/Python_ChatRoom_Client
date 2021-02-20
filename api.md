
# Python Chatroom v5.0 API 

**If you just want a list of commands, you only have to read the "Commands" section.**

Here you go! I really should have included this before. Now I've read about Semantic Versioning, so this is going to be part of the v5.0 update. This is the API documentation for chatroom v5.0. If you have no idea what an API is or you don't care about how the code works, you can safely ignore this document. However, if you are interested, I think it would be a fun thing to just read through this regardless of if you know what an API is. 

It's okay to not understand everything! We don't expect you to understand everything. We encourage you to learn more about things you don't get. Please do not feel discouraged by words or concepts you do not know. If you can't be bothered, most of the special words here aren't very important anyway, so you can just read on without looking it up. 

This API isn't meant to be official or specific enough so that it can be brought into a court of law. We don't have the time to read those regulations. This is for your convenience and for staying with semantic versioning standards. Then again, if you find anything you think we should change, please email us at pyvagroup@gmail.com so we can fix anything. 

## Notes on the Code 
First, there are a few things about the actual code that we'd like to say. 
- Most importantly, we use some `os` stuff to get the program to not run from IDLE. This improves performance a lot, but if you want to change the code, you probably want to take out the code at the bottom of the main file and replace it with `client = Client(1235)`. 

- We put a `sys.stderr = DevNull()` line to discourage errors from `tkinter` that we can't figure out how to fix. So if you're adding or removing features of the code and would like to debug, please remove that line. It's near the top. 

- The `Mining.py` file contains a function that is run as a thread every time you connect to the server. This is supposed to be used for coin mining, details of which are in the file itself, but you can put anything else in there. 
- The encryption system may or may not be secure. We don't expect it to be hacked, but then, it might be, because we put together RSA and AES to make PGP and called it a day. We don't actually live up to the standard. We don't accept responsibility for anything caused by insecure encryption methods. (Sorry about that.) 

## Encryption System
Speaking of the encryption system, we use the libraries `rsa` and `pyaes` to encrypt and decrypt messages. We're pretty sure these are legit libraries, but we're still not responsible for anything that happens as a result of these being hacked. 

As part of update v5.0, we changed the keys from 1024-bit to 256-bit, at practically no cost to security (we don't hold ourselves accountable for that) but an increased speed. Okay, now for the details. 

### Encryption
Suppose we want to send a message to the server. We generate a 16-byte random string using `secrets.token_urlsafe()` for the AES key. We then use the CTR mode of AES to encrypt the message. 
The key is then encrypted using RSA with the server's public key. This encrypted key is 32 bytes long. We append the encrypted message to the encrypted key, and send it off to the server. 

**Important:*- All messages, after encryption, must be buffered with five (5) newline bytes at the end (`b'\n'`). 

### Decryption 
Decryption works the same way. When we receive a message of bytes, we remove the first 32 bytes and decrypt that using our private RSA key to get a 16-bit AES key. We then take all the bytes after the first 32 and decrypt that with the AES key to get a plaintext message of bytes, which is then converted to  a string. 

### Hashing 
We use the SHA512 hash for everything, including password storage, mining, and password sending. Don't worry! Passwords are salted and hashed and then stored on the server. 

From this point on, we assume this system when we say "encrypt" or "decrypt" or anything like that.  Please note that Python sockets send **only*- encoded bytes strings. We use utf-8 for this. 

## Logging In 
### Authentication
You can connect to the server via the normal Python `socket` library, no special modifications or parameters required. 
When you first connect as a new user, you must send credentials to the server in plaintext along with information that you are a new user. These credentials are, in order: 

1. Your public key modulus. 
2. Your public key exponent (usually 2^16 = 65536). 
3. Your username. 
4. Your password. *
5. "`newacc`". 

\- It doesn't matter what you send to the server as your password, as long as it's consistent. The existing system is that you concatenate `username + password` in that order, then hash it (with SHA512). This has the benefit that people can't recover your original password that may be used elsewhere, you can log in from any computer, and the server doesn't have to waste time sending out encryption keys. 

These lines should be separated with `\n` newline characters, without one at the very end. If you have an existing account, you can send the same credentials without the "`newacc`" at the end. Details of whether your login was successful or not are sent to you encrypted with your public key. New usernames may not contain certain special characters. 

If the login or creation of a new account was unsuccessfully, the connection will be closed by the server regardless of the message you get or if you sent a valid public key at all. 

### Information 
1. Once you have provided correct credentials, the server sends you their public key non-encrypted in the form `[modulus]\n[exponent]`. You should use this to encrypt all messages you send to the server after this point. 
2. You then must send the server an acknowledgement. This can be anything but must be something. For example, you can send the individual byte `b' '`, no buffering required. 
3. The server then sends you information in this format: `[money]\n[cookies]\n[prev_hash]\n[hash_zeros]`, encrypted and with buffering at the end. `[money]` is your monetary balance. `[cookies]` is how many cookies you have. 
	- `[prev_hash]` and `[hash_zeros]` are necessary for mining, details of which are in `Mining.py`, but you don't have to do anything with these if you don't want. 
4. Then, you must send another acknowledgement message, and as before, it can be a single byte. 

## Commands (sent to the server) 
Once you successfully set up a connection with the server, send valid credentials, and receive the information fro the server, the server puts you into a client thread. The following list describes the valid commands, their parameters, and what you can expect to receive from the server. 

All messages are to be **encrypted*- messages with the proper buffering (5 newline bytes `b'\n'`), and you can send multiple commands by separating them with `\n` characters and then encrypting the message (and adding the buffering afterwards). 

Text in square brackets is to be interpreted as a variable. 

**`[exit]`**
- This tells the server that you wish to exit the chatroom. The server will promptly kick you off. 

**`[message]`**
- This sends a raw message to the server. If this messages does not contain commands, the server broadcasts this message to all users including you, prefixed with your username, such that the message everybody receives is `[username]> [message]`. 

**`/w [user] [message]`**
- Sends a private message to all devices with username `[user]`. Note that this may be several devices. 
- The message the other user receives is `[username]> /w [message]`, where `[username]` is your username. 
- You receive a message in the form `[username]> /w To:[user] [message]`, to notify you that you have sent a private message. 

**`/ping [data]`**
- This tests the time it takes for a message to travel from your computer to the server and back. Sending data is optional. 
- The server will send back the exact message, `/ping [data]` as quickly as possible granted the time for encryption and buffering. 

**`/active`**
- This retrieves a list of the usernames of active users. 
- The server returns `'Server> Active users: \n`, followed by the newline character `\n` separating the usernames of all active users. 

**`/a [command] [args]`**
- If you have been granted admin powers either through purchase or favorability, you can preform admin commands. Here is a list of the ones in v5.0: 
	- **`/a /kick [username]`**: This kicks a user from the chatroom on all their devices. If `[username]` is not active or does not exist, the server will tell you. 

**`/mine [string]`**
- Details of this are in the `MIning.py` file. This command is used to mine a string. 

**`/requestLeaderboard`**
- This is used to request a list of all users and the amount of coins they have. 
- See "**`/update_leaderboard`**" in the **Commands (from the server)** section. 

**`/requestHash`**
- This is used for mining. If you don't care about mining, you can ignore this command. The server will send you the message `/newHash [string]` to inform you of the hash of the most recent block in the blockchain. 

**`/cost [item]`**
- The server will send you the message `Server> [item] costs [cost] coins. `, where `[item]` costs `[cost]` coins. 

**`/buy [item] [args]`**
- You can use this to buy an item. Your purchase will be registered in the server, and the server will notify you of your change in balance by sending `/balance -[cost]`, where `[cost]` is positive. `[args]` is useful for things like buying cookies in mass. 
- Here is a list of things you can buy and their functionality.  These are to meant be interpreted as raw strings. 

|Item         |Description              |
|-------------|-------------------------|
|`adminStatus`|Grants you admin powers. |
|`cookie`     |Buys you a cookie!       |

**`/delacc [password]`**
- You need to provide your password to delete your account. This password should be the same one you send to the server during login. 
- The server will close all connections that are associated with your username, your account will no longer exist, and you will lose all your money as well as anything you bought. 

**`/changepass [oldPassword] [newPassword]`**
- You need to provide your old password in order to change your password. This should be the same one you send during login. 
- Your password will be changed, nothing else will happen to your account, and you do not have to log in again. 

## Commands (sent by the server) 
Depending on what you send, the server will send some messages back. The following list describes what each type of message means. It is **not** assumed that each message is prefixed by a "`[user]>`". When it is such the case that there is no `[user]>` prefix, it is be assumed that the message is from the server.  

**`[username]> [message]`**
- This is a plain message, and has no special meaning if `[username]` is not "`Server`". 

**`Server> [message]`**
- Usually, this has no meaning either and is used by the server to tell you a plain message UNLESS the message is: 
	- `Server> You have been kicked by [user or the Server].` This means your connection has been closed, either by the server or an admin. 
	- `Server> Your password has been successfully changed. ` Means exactly what it says. 

**`[username]> /w [message]`**
- In this case, `[username]` can be "`Server`" or any other username. The `/w` means that this is a private message, so that you are the only person who received this message. 

**`/balance [amount]`**
- This means the server has changed your balance by `[amount]`, be it you bought something, mined, or sent an incorrect hash. Remember that the server is the definitive source on your balance, and the local information is to be used for convenience only. 
- Note that `[amount]` can be negative. 

**`/cookies [amount]`**
- This means you have successfully bought cookies and that your cookie count has been changed by `[amount]` on the server. 

**`/newHash [string]`**
- This is for mining purposes only. This message tells you that somebody has successfully mined, thus a new block has been added to the blockchain. You should modify some variable so that you mine based on this new string. 

**`/update_leaderboard [data]`**
- This is to update your local copy of the leaderboard, and this message usually follows from you sending `/requestLeaderboard`. 
- The server sends you stats for each user. For every user, their stat is `[user],[coins]`. Each stat is separated by a space and appended to the string `/update_leaderboard`, with a space between the first data point and the `/update_leaderboard`. 
- For example, it might be `/update_leaderboard Joe,10 Bob,100, Smith,0`. 

**`/ping [data]`**
- This follows from when you send a ping to the server via the `/ping` command. This is used to track how much time it takes for data to travel from your computer to the server and back. See the "`/ping`" section in **Commands (to the server)**. 

**`/e delacc [message]`**
- This is used to notify you that something has gone wrong with deleting your account; usually, it means that you have sent the wrong password. 

**`/e changepass [message]`**
- Similarly to `/e delacc`, this is used to notify you that something has gone wrong with changing your password. Likely you have simply sent the wrong password, but it is also possible you did not send a new password. 

## Conclusion
That's all for the API. We're sorry if it's not detailed enough; again, if you have any questions, please email us at pyvagroup@gmail.com. We will probably respond in six to eight weeks. 

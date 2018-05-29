# Basic FTP

[![Build Status](https://travis-ci.org/patrickjuchli/basic-ftp.svg?branch=master)](https://travis-ci.org/patrickjuchli/basic-ftp) [![npm version](https://img.shields.io/npm/v/basic-ftp.svg)](https://www.npmjs.com/package/basic-ftp)

This is an FTP client for Node.js. It supports explicit FTPS over TLS, Passive Mode over IPv6, has a Promise-based API, and offers methods to operate on whole directories.

## Dependencies

Node 8.0 or later is the only dependency.

## Introduction

The following example will connect to a server, upgrade to TLS, login, get a directory listing, and upload a file. Note that the FTP protocol doesn't allow multiple requests running in parallel.

```js
const ftp = require("basic-ftp")
const fs = require("fs")

example()

async function example() {
    const client = new ftp.Client()
    try {
        await client.access({
            host: "myftpserver.com",
            user: "very",
            password: "password",
            secure: true
        })
        console.log(await client.list())
        await client.upload(fs.createReadStream("README.md"), "README.md")
    }
    catch(err) {
        console.log(err)
    }
    client.close()
}
```

The next example shows how to work with directories and their content. First, we make sure a remote path exists, creating all directories as necessary. Then, we remove any existing directories and files from it and upload the contents of a local one.

```js
await client.ensureDir("my/remote/directory")
await client.clearWorkingDir()
await client.uploadDir("my/local/directory")
```

If you encounter a problem, it can be helpful to log out all communication with the FTP server.

```js
client.ftp.verbose = true
```

## Client API

`new Client(timeout = 0)`

Create a client instance using an optional timeout in milliseconds that will be used for control and data connections. Use 0 to disable timeouts.

`close()`

Close all socket connections.

---

`access(options): Promise<Response>`

Convenience method to get access to an FTP server. This method calls *connect*, *useTLS*, *login* and *useDefaultSettings* described below. It returns the response of the initial connect command. The available options are:

- `host (string)`: Host to connect to
- `port (number)`: Port to connect to
- `user (string)`: Username for login
- `password (string)`: Password for login
- `secure (boolean)`: Use explicit FTPS over TLS
- `secureOptions`: Options for TLS, same as for [tls.connect()](https://nodejs.org/api/tls.html#tls_tls_connect_options_callback) in Node.js.

---

The following setup methods are for advanced users who want to customize an aspect of accessing an FTP server. If you use *client.access* you won't need them.

`connect(host = "localhost", port = 21): Promise<Response>`

 Connect to an FTP server.

`useTLS([options]): Promise<Response>`

Upgrade the existing control connection with TLS. You may provide options that are the same you'd use for [tls.connect()](https://nodejs.org/api/tls.html#tls_tls_connect_options_callback) in Node. Remember to upgrade before you log in. Subsequently created data connections will automatically be upgraded to TLS reusing the session negotiated by the control connection.

`login(user = "anonymous", password = "guest"): Promise<Response>`

Login with a username and a password.

`useDefaultSettings(): Promise<Response>`

Sends FTP commands to use binary mode (TYPE I) and file structure (STRU F). If TLS is enabled it will also send PBSZ 0 and PROT P. It's recommended that you call this method after upgrading to TLS and logging in.

---

`features(): Promise<Map<string, string>>`

Get a description of supported features. This will return a Map where keys correspond to FTP commands and values contain further details.

`send(command, ignoreErrorCodes = false): Promise<Response>`

Send an FTP command. You can choose to ignore error return codes. Other errors originating from the socket connections including timeouts will still reject the Promise returned.

`cd(remotePath): Promise<Response>`

Change the working directory.

`pwd(): Promise<string>`

Get the path of the current working directory.

`list(): Promise<FileInfo[]>`

List files and directories in the current working directory. Currently, this library only supports Unix- and DOS-style directory listings.

`size(filename): Promise<number>`

Get the size of a file in the working directory.

`remove(filename, ignoreErrorCodes = false): Promise<Response>`

Remove a file from the working directory.

`upload(readableStream, remoteFilename): Promise<Response>`

Upload data from a readable stream and store it as a file with a given filename in the current working directory.

`download(writableStream, remoteFilename, startAt = 0): Promise<Response>`

Download a file with a given filename from the current working directory and pipe its data to a writable stream. You may optionally start at a specific offset, for example to resume a cancelled transfer.

---

`ensureDir(remoteDirPath): Promise<void>`

Make sure that the given `remoteDirPath` exists on the server, creating all directories as necessary. The working directory is at `remoteDirPath` after calling this method.

`clearWorkingDir(): Promise<void>`

Remove all files and directories from the working directory.

`removeDir(remoteDirPath): Promise<void>`

Remove all files and directories from a given directory, including the directory itself. When this task is done, the working directory will be the parent directory of `remoteDirPath`.

`uploadDir(localDirPath, [remoteDirName]): Promise<void>`

Upload all files and directories of a local directory to the current working directory. If you specify a `remoteDirName` it will place the uploads inside a directory of the given name. This will overwrite existing files with the same names and reuse existing directories. Unrelated files and directories will remain untouched.

`downloadDir(localDirPath): Promise<void>`

Download all files and directories of the current working directory to a given local directory.

---

`trackProgress(handler)`

Report any transfer progress using the given handler function. See the next section for more details.

## Transfer Progress

Set a callback function with `client.trackProgress` to track the progress of all uploads and downloads. To disable progress reporting, call `trackProgress` with an undefined handler.

```js
// Log progress for any transfer from now on.
client.trackProgress(info => {
    console.log("File", info.name)
    console.log("Type", info.type)
    console.log("Transferred", info.bytes)
    console.log("Transferred Overall", info.bytesOverall)
})

// Transfer some data
await client.upload(someStream, "test.txt")
await client.upload(someOtherStream, "test2.txt")

// Set a new callback function which also resets the overall counter
client.trackProgress(info => console.log(info.bytesOverall))
await client.downloadDir("local/path")

// Stop logging
client.trackProgress()
```

For each transfer, the callback function will receive the filename, transfer type (upload/download) and number of bytes transferred. The function will be called at a regular interval during a transfer.

There is also a counter for all bytes transferred since the last time `trackProgress` was called. This is useful when downloading a directory with multiple files where you want to show the total bytes downloaded so far.

## Error Handling

Errors originating from a connection or described by a server response as well as timeouts will reject the associated Promise. Use a try-catch-clause when using async-await or `catch()` when using Promises directly. The error description depends on the type of error.

### Timeout

```
{
    error: "Timeout control socket"
}
```


### Connection error

```
{
    error: [Error object by Node]
}
```

### FTP response

```
{
    code: [FTP error code],
    message: [Complete FTP response including code]
}
```


## Customize

The `Client` offers extension points that allow you to change a detail while still using existing functionality like uploading a whole directory.

`get/set client.prepareTransfer` 

FTP creates a socket connection for each single data transfer. Data transfers include directory listings, file uploads and downloads. This property holds the function that prepares this connection. Currently, this library offers Passive Mode over IPv4 (PASV) and IPv6 (EPSV) but this extension point makes support for Active Mode possible. The signature of the function is `(ftp: FTPContext) => Promise<void>` and its job is to set `ftp.dataSocket`. The section below about extending functionality explains what `FTPContext` is.

`get/set client.parseList`

You can provide a custom parser to parse directory listing data. This library supports Unix and DOS formats out-of-the-box. Parsing these list responses is one of the more challenging parts of FTP because there is no standard that all servers adhere to. The signature of the function is `(rawList: string) => FileInfo[]`. `FileInfo` is also exported by the library.


## Extend

You can use `client.send` to send any FTP command and get its result. This might not be good enough, though. FTP can return multiple responses after a command and a simple command-response pattern won't work. You might also want to have access to sockets.

The `Client` described above is just a collection of convenience functions using an underlying `FTPContext`. An FTPContext provides the foundation to write an FTP client. It holds the socket connections and provides an API to handle responses and events in a simplified way. Through `client.ftp` you get access to this context.

### FTPContext API

`get/set verbose`

Set the verbosity level to optionally log out all communication between the client and the server.

`get/set encoding`

Get or set the encoding applied to all incoming and outgoing messages of the control connection. This encoding is also used when parsing a list response from a data connection. Node supports `utf8`, `latin1` and `ascii`. Default is `utf8` because it's backwards-compatible with `ascii` and many modern servers support it, some of them without mentioning it when requesting features. You can change this setting at any time.

`get/set ipFamily`

Set the preferred version of the IP stack: `4` (IPv4), `6` (IPv6) or `undefined` (Node.js default). Set to `undefined` by default.

`get/set socket`

Get or set the socket for the control connection. When setting a new socket the current one will *not* be closed because you might be just upgrading the control socket. All listeners will be removed, though.

`get/set dataSocket`

Get or set the socket for the data connection. When setting a new socket the current one will be closed and all listeners will be removed.

`handle(command, handler): Promise<Response>`

Send an FTP command and register a handler function to handle all subsequent responses and socket events until the task is rejected or resolved. `command` may be undefined. This returns a promise that is resolved/rejected when the task given to the handler is resolved/rejected. This is the central method of this library, see the example below for a more detailed explanation.

`send(command)`

Send an FTP command without waiting for or handling the response.

`log(message)`

Log a message if the client is set to be `verbose`.

### Example

The best source of examples is the implementation of the `Client` itself as it's using the same single pattern you will use. The code below shows a simplified file upload. Let's assume a transfer connection has already been established.

```js
function mySimpleUpload(ftp, readableStream, remoteFilename) {
    const command = "STOR " + remoteFilename
    return ftp.handle(command, (res, task) => {
        if (res.code === 150) { // Ready to upload
            readableStream.pipe(ftp.dataSocket)
        }
        else if (res.code === 226) { // Transfer complete
            task.resolve(res)
        }
        else if (res.code >= 400 || res.error) {
            task.reject(res)
        }
    })
}

await mySimpleUpload(client.ftp, myStream, myName)
```

This function represents an asynchronously executed task. It uses a method offered by the FTPContext: `handle(command, callback)`. This will send a command to the FTP server and register a callback that is in charge for handling all responses from now on. The callback function might be called several times as in the example above. Error and timeout events from both the control and data socket will be rerouted through this callback as well. Also, `client.handle` returns a `Promise` that is created for you and which the upload function above returns. That is why the function `myUpload` can now be used with async/await. The promise is resolved or rejected when you call `resolve` or `reject` on the `task` reference passed to you as a callback argument. The callback function will not be called anymore after resolving or rejecting the task.


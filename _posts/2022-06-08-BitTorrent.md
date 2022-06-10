---
layout: post
title: "Building a BitTorrent Client"
date: 2022-06-08 08:51:00 -0000
categories: networking c-programming
---

## Building a BitTorrent Client
This post contains all I learnt about building a BitTorrent Client with C. It's not a replacement for the BitTorrent Protocol spec. You can find the official spec [here](http://bittorrent.org/beps/bep_0003.html) though this is not as detailed and thorough as the [unofficial spec](http://wiki.theory.org/BitTorrentSpecification). I advise you to read both of them if you need a fuller understanding of the protocol. 

This post does not cover working with trackerless torrents using [peer exchange](https://www.bittorrent.org/beps/bep_0011.html) and [Distributed Hash Table protocol](https://www.bittorrent.org/beps/bep_0005.html). It only covers the original protocol which must involve a tracker. 

The client I built mainly implements the client side of the protocol and some of the server side. What this means is that it can share files if a peer requests for a file but it doesn't listen for connection. The only way a peer can communicate with this client is if this client initiates the connection.

I'll start this post by giving an overview of the Bittorrent protocol before going into details.

### The BitTorrent Protocol
The BitTorrent protocol is a peer to peer protocol that's mainly used for sharing files. What this means is that a peer can act as a client and a server in sharing a file. There is no central server that all clients connect to in order to download the files. In BitTorrent land, all clients can be servers and all servers are clients. 

Let's say you want to download David Copperfield by Charles Dickens using the BitTorrent protocol. The steps will be:

* A torrent file for David Copperfield will be downloaded from the web using your browser. This torrent file contains some info containing metadata about the book. Some of the metadata will be the file size, name of the file and may also include the hash of the file for verification purpose. The torrent file will also contain BitTorrent specific info like the trackers, file pieces hash e.t.c. Once these are parsed, connection(s) to the tracker(s) is/are initiated.

* The tracker is a server that has the ip address and ports of all the peers that are currently downloading and uploading the file. We connect to the tracker to get a list of the peers that are currently sharing and receiving the file. Once this is successfully done, our ip address and port is added to this list for future peers for a limited time.  
It's true that BitTorrent is a peer to peer protocol but there's a measure of centralisation in it. BitTorrent now supports true decentralisation by using DHT but this won't be convered here.  
By the way, I have noticed that most working decentralised protocol usually incorporates a centralised method for locating other devices before switching to the main protocol for communication kind of like the way the internet all use the [13 root name server addresses](https://en.wikipedia.org/wiki/Root_name_server) for DNS lookup before main communication starts.

* A peer is a device like yours too which makes you a peer. There are 2 types of peers though they aren't mutually exclusive. The peer who downloads is a **leecher** while the one who uploads is a **seeder**. A peer can be both. We download pieces of the file from the seeders. A piece is a segment of the file. The pieces are verified and inserted into the file in their correct position. Once all the pieces are downloaded, the file download is complete. If you remain in the network, you stop being a leecher and fully becomes a seeder.  
The protocol performs better when they are more seeders so there are mechanisms in place to encourage more seeder-like behaviour. These mechanisms won't be covered here. The client I built actually encourages leeching so your download could be slower than if you use the popular BitTorrent clients.

This concludes the overview. The next section covers technical details about the torrent file.

### The Torrent File
The torrent file is a giant dictionary that contains information on how to download the file. This dictionary is encoded using the bencode format.  

The bencode encoding transforms dictionaries, lists, strings and integers into a giant string. Each datatype has their own format. For example, the torrent file starts with _'d'_ denoting that it's a dictionary. More details about the bencode format [here](http://bittorrent.org/beps/bep_0003.html#bencoding) and [here](https://wiki.theory.org/index.php/BitTorrentSpecification#Bencoding). Note that dictionaries can contain dictionaries too.

Some of the important keys in the torrent dictionary are listed below:
* announce: This contains the url of a single tracker.
* announce-list: This contains a list of tracker urls.
* info: This is a dictionary that contains file related details. A hash of the bencoded value of this key (including the _d_ and _e_) is used by both the trackers and peers to identify the torrent file.  Some of the details are: 
    * pieces: A long binary string consisting of the concatenation of the SHA-1 hash of each piece. Each hash is 20 bytes long so this makes the length of this string a multiple of 20. We can divide this string length by 20 to get the exact number of pieces to download.
    * piece length: The size of each piece in bytes. It's possible that the last piece might not be as large as this value so this value multiplied by the number of pieces might not be the accurate size of the file(s).
    * name: The name of file/directory to download.
    * files: This key only appears in when more than one file is being downloaded. It's a dictionary that contains the _path_ and _length_ of each file.
    * length: This key only appears **directly under the info dictionary** when only one file is being downloaded. It contains the size of the file in bytes. The value of this gives the accurate size of the file. When more than one file is downloaded, this key appears under the **files** key which is defined above. In this situation, a sum of these values gives the total size of the files.  
    It is important that an accurate size of the file(s) is calculated because this makes the accurate size of the last piece to be inferred. This can be easily calculated by using the modulo function if the total size is not divisible by the **piece length**.

Note that these aren't all the keys in the torrent file. You can read about more torrent keys and details [here](https://wiki.theory.org/index.php/BitTorrentSpecification#Metainfo_File_Structure).

The urls in the **announce** and (maybe) **announce-list** are extracted and connected to in order to get the peers. The trackers' communication flow is explained in the next section.

### Tracker
Communication with the tracker is necessary in order to get the peers. The trackers can be communicated with using either the UDP or HTTP/HTTPS protocol depending on the url scheme. UDP is used for trackers whose urls begin with _udp://_ while HTTP/HTTPS is used for trackers whose urls begin with _http://_ or _https://_.   

It's very possible that some of the trackers will produce an error when connected to so getting an error from them doesn't mean that your protocol implementation is wrong. The error might be due to a myriad of reasons that are beyond your control. This is why connecting to more of them increases your chances of getting peers.  

Previously, it was http(s) that was the only protocol but due to the demand on trackers by peers and the overhead introduced by the http(s) protocol, udp was introduced. Both protocols will be covered here.

#### HTTP(s) Protocol
We won't cover https here since its only difference from http is the use of certificates. 

A GET request is sent to the tracker where the url is either the **announce** value or one of the **announce-list** values. I wrote a very simple implementation of this using the sockets library but you're free to use any http library of your choice when building your client. Some values are added as parameters to the url. Values that need to be encoded have to be encoded using the [percent encoded](https://en.wikipedia.org/wiki/Percent-encoding) method. Some of the important parameters are:
* info_hash: This is the urlencoded SHA-1 hash of the **info** key value in the bencoded torrent dictionary. The hash is 20 bytes in length. The **info** key value has to include the _d_ and _e_ characters used to delimit the start and end of the dictionary.  
It is important that the hash has to be correct because it is used by the tracker to determine the torrent file. If it's incorrect, the tracker will either produce an error or worse, give you a different set of peers.  
The hash is a binary string and therefore has to be urlencoded.
* peer_id: This is a unique id that your client generates to identify your device. The length has to be exactly 20 characters long. The characters can even be binary. Different BitTorrent clients have [different conventions](https://wiki.theory.org/index.php/BitTorrentSpecification#peer_id) for generating peer_id. This parameter value has to be urlencoded.
* port: The port your client is listening on. It's usually set to between 6881-6889. If your device is behind a NAT, you might need to use TURN/STUN methods to determine what port is set in the NAT before setting this parameter. If you choose to be solely a leecher, you can set this to 0.
* ip: The ip address of your client. If you choose to be solely a leecher, you can set this to 0. This is optional because most trackers ignore this as they can determine this from the ip address your HTTP request is coming from. 
* compact: This determines the format of the peer string in the tracker's response. Setting this to 1 ensures that the peers list is returned as a string with each peer being 6 bytes long. This means that the number of peers received is a multiple of 6. The ip address and port of each peer are 4 bytes long and 2 bytes long respectively. This is surprising because we know that ipv4 addresses in dotted format are a minimum of 7 bytes long (0.0.0.0). Well, they can also be represented as a  [decimal number](https://en.wikipedia.org/wiki/IPv4#Addressing). This number is stored in big-endian notation. The number formatted version of the ipv4 is 4 bytes. The port is 2 bytes because every port can be represented using 2 8-bit characters since the range is 0-65535. Setting this parameter to 0 ensures the peers list is returned as a bencoded dictionary.Many trackers prefer requests with this set to 1 in order to save bandwidth.  

Here's an hypothetical request for a tracker whose url is http://opentracker.bittorrent.com/announce:7456

```
    /GET /announce?info_hash=%124Vx%9A%BC%DE%F1%23Eg%89%AB%CD%EF%124Vx%9A&peer_id=-BT0001-948911116432&port=6889&uploaded=0&downloaded=0&left=489033&compact=1&event=started HTTP/1.1
    Host: opentracker.bittorrent.com:7456

```

If the info hash is correct and the tracker has some peers connected to it, the response will be returned as a bencoded dictionary. Some of the keys in the dictionary are:
* interval: The interval in seconds that the client should wait before sending regular requests to the tracker. The purpose of this interval is for the client to update their peer list. During the course of downloading the file, there is a very high likelihood that the number of peers that the client is downloading from will reduce due to peers disconnecting (it's also possible that the number of peers can drop to zero which is bad). This will affect the download speed as there is a positive correlation between the number of peers and the download speed. To prevent this, it's good for a client to regularly get new peers list from the tracker. In order to prevent clients from sending requests too frequently, trackers send this value.
* complete: Number of peers with the entire file i.e seeders.
* incomplete: Number of peers with incomplete file i.e leechers.
* peers: This contains a list of peers along with their peer id (if **compact** is set to 0), ip address (dotted if **compact** is set to 0 or decimal if set to 1) and port. This will be in a list of dictionary form if **compact** is set to 0 or just a binary string if **compact** is set to 1.

Here's an hypothetical reply for the request sent above

```
    d8:completei1e10:downloadedi0e10:incompletei1e8:intervali2988e12:min intervali1494e5:peers12:6@]-N+Nd-6%À
```

The peers in the reply translate to 54.64.93.45:20011 and 78.100.45.54:9664.

More explanation about tracker HTTP protocol can be read in the [spec](https://wiki.theory.org/index.php/BitTorrentSpecification#Tracker_HTTP.2FHTTPS_Protocol).  

#### UDP Protocol
Due to the overhead introduced by the http protocol, a udp version was introduced. Unlike the http protocol that is text-based, this is binary-based. Because of this, numbers sent and received to and from the trackers have to be in network byte order a.k.a big-endian. Some devices are little endian e.g Intel x86 machines while some are big endian e.g ARM devices. It is a good practice to use functions that can convert between endianness when sending to and receiving from the trackers. In C, the [htonl, htons, ntohl, ntohs](https://linux.die.net/man/3/htonl) functions are used. In [Python](https://docs.python.org/3/library/struct.html#struct.pack) and [PHP](https://www.php.net/manual/en/function.pack.php), the pack and unpack functions are used.  

The udp protocol is an unreliable protocol so a retry with timeout flow has to be implemented.  
Unlike the http protocol that only involves one request-response cycle, the udp protocol involves two cycles. This is to avoid UDP spoofing. The first of these two is called the **connect** cycle while the second is called the **announce** cycle. Both will be covered in this subsection.  

The connect cycle is used to get the connection id that will be used for the announce cycle. The connection id is a time-limited value that has to be sent in the announce request. This id is used by the tracker to ensure that the announce request is coming from a connected client. The connect request contains a **magic constant** (0x41727101980) which is 8 bytes, a 4 bytes **action** value (set to 0) and a 4 bytes **transaction id** (random number generated by the client). These makes the total request size 16 bytes. These numbers have to be in big endian format. On receiving this request, the tracker responds with a 4 bytes **action** value (0), the 4 bytes **transaction id** that was sent by the client and an 8 bytes **connection id** making it a total of 16 bytes for the size of the response payload. These numbers are also in big endian format. The **transaction id** is verified by the client to make sure it's equal to the one sent in the request, this ensures that the client is receiving its own response and not that of another client.

Here's an example connect request message. The device is assumed to be a big-endian one. The parameters are shown along with their hex equivalent. Note that each pair of hexadecimal values make one byte i.e a byte is between 00-FF.

```
    magic_num = 0x41727101980 => 0000041727101980 (8 bytes)
    action = 0 => 00000000 (4 bytes)
    transaction_id = 765 => 000002FD (4 bytes)
    // numbers are converted to hex and sent
    message = 000004172710198000000000000002FD (16 bytes)
```

Here's a reply message from the tracker.

```
    00000000000002FD00000003DCB35E1B (16 bytes)

    //the decomposed components of the message and their translation will be
    action = 00000000 (4 bytes) => 0
    transaction_id = 000002FD (4 bytes) => 765
    connection_id = 00000003DCB35E1B (8 bytes) => 16587644443
```

After verifying the transaction id, the client can then initiate the **announce** cycle.  

The announce cycle is where the client gets the list of peers from the tracker. The request includes the **connection id** sent by the tracker in the **connect** cycle, an **action** value (set to 1), **transaction id** and the parameters that are stated in the HTTP protocol. The **compact** parameter is not set since the udp protocol is a binary protocol anyway so all peers list received is in binary (akin to setting compact to 1). The response sent by the trackers include the **action** value (set to 1), the **transaction id** sent by the client and the values defined in the http protocol explanation above. Please note that all the numbers sent and received in this cycle are also in big-endian format.  

Here's an example announce request message. The device is assumed to be a big-endian one
```
    connection_id = 16587644443 => 00000003DCB35E1B (8 bytes)
    action = 1 => 00000001 (4 bytes)
    transaction_id = 823 => 00000337 (4 bytes)
    info_hash = 123456789ABCDEF123456789ABCDEF123456789A //already in hex (20 bytes)
    peer_id = -BT0001-948911116432 => 2D4254303030312D393438393131313136343332 (20 bytes)
    downloaded = 0 => 0000000000000000 (8 bytes)
    left = 489033 => 0000000000077649 (8 bytes)
    uploaded = 0 => 0000000000000000 (8 bytes)
    event = 0 => 00000000 (4 bytes)
    ip_address = 0 => 00000000 (4 bytes)
    key = 0 => 00000000 (4 bytes)
    num_want = -1 => FFFFFFFF (4 bytes)
    port = 6889 => 1AE9 (2 bytes)
    // numbers are converted to hex and sent
    message = 00000003DCB35E1B0000000100000337123456789ABCDEF123456789ABCDEF123456789A2D4254303030312D393438393131313136343332000000000000000000000000000776490000000000000000000000000000000000000000FFFFFFFF1AE9 (98 bytes)
```

Here's a reply message to the above request

```
    000000010000033700000BAC000000010000000136405D2D4E2B4E642D3625C0 (32 bytes)

    // the decomposed components and their translation will be
    action = 00000001 (4 bytes) => 1
    transaction_id = 00000337 (4 bytes) => 823
    interval = 00000BAC (4 bytes) => 2988
    leechers = 00000001 (4 bytes) => 1
    seeders = 00000001 (4 bytes) => 1
    first peer ip&port = 36405D2D4E2B (6 bytes) => 54.64.93.45:20011
    second peer ip&port = 4E642D3625C0 (6 bytes) => 78.100.45.54:9664
```

It is recommended that you read more about the tracker [UDP protocol](https://www.bittorrent.org/beps/bep_0015.html).  

After the client verifies that the transaction id received is equal to the one sent, the peer list can be parsed and connections to each peer are initiated using the peer message protocol.
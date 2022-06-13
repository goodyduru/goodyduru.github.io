---
layout: post
title: "Building a BitTorrent Client"
date: 2022-06-08 08:51:00 -0000
categories: networking c-programming
---

## Building a BitTorrent Client
This post contains all I learnt about building a BitTorrent Client with C. It's not a replacement for the BitTorrent Protocol spec. You can find the official spec [here](http://bittorrent.org/beps/bep_0003.html) though this is not as detailed and thorough as the [unofficial spec](http://wiki.theory.org/BitTorrentSpecification). I advise you to read both of them if you need a fuller understanding of the protocol. 

This post does not cover working with trackerless torrents using [peer exchange](https://www.bittorrent.org/beps/bep_0011.html) and [Distributed Hash Table protocol](https://www.bittorrent.org/beps/bep_0005.html). It only covers the original protocol which must involve a tracker. 

The client I built mainly implements the client side of the protocol and some of the server-side. What this means is that it can share files if a peer requests for a file but it doesn't listen for a connection. The only way a peer can communicate with this client is if this client initiates the connection.

You can find the source code of my client on [Github](https://github.com/goodyduru/simpletorrent)

I'll start this post by giving an overview of the Bittorrent protocol before going into details.

### The BitTorrent Protocol
The BitTorrent protocol is a peer-to-peer protocol that's mainly used for sharing files. What this means is that a peer can act as a client and a server in sharing a file. There is no central server that all clients connect to download the files. In BitTorrent land, all clients can be servers and all servers are clients.  

Let's say you want to download David Copperfield by Charles Dickens using the BitTorrent protocol. The steps will be:

* A torrent file for David Copperfield will be downloaded from the web using your browser. This torrent file contains some info containing metadata about the book. Some of the metadata will be the file size, and name and may also include the hash of the file for verification purposes. The torrent file will also contain BitTorrent-specific info like the trackers, file pieces hash e.t.c. Once these are parsed, connection(s) to the tracker(s) is/are initiated.

* The tracker is a server that has the IP address and ports of all the peers that are currently downloading and uploading the file. We connect to the tracker to get a list of the peers that are currently sharing and receiving the file. Once this is successfully done, our IP address and port are added to this list for future peers for a limited time.  
BitTorrent is indeed a peer-to-peer protocol but there's a measure of centralization in it. BitTorrent now supports true decentralization by using DHT but this won't be covered here.  
By the way, I have noticed that most working decentralized protocol usually incorporates a centralized method for locating other devices before switching to the main protocol for communication kind of like the way the internet all use the [13 root name server addresses](https://en.wikipedia.org/wiki/Root_name_server) for DNS lookup before the main communication starts.  

* A peer is a device like yours too which makes you a peer. There are 2 types of peers though they aren't mutually exclusive. The peer who downloads is a **leecher** while the one who uploads is a **seeder**. A peer can be both. We download pieces of the file from the seeders. A piece is a segment of the file. The pieces are verified and inserted into the file in their correct position. Once all the pieces are downloaded, the file download is complete. If you remain in the network, you stop being a leecher and fully becomes a seeder.  
The protocol performs better when they are more seeders so there are mechanisms in place to encourage more seeder-like behavior. These mechanisms won't be covered here. The client I built encourages leeching so your download could be slower than if you use the popular BitTorrent clients.

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
    // the hex values are concatenated
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
    // the hex values are concatenated together
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

A way to read and write messages like the above in C is to use the [memcpy](https://man7.org/linux/man-pages/man3/memcpy.3.html) function. Python has the [struct](https://docs.python.org/3/library/struct.html) package. PHP has the [pack](https://www.php.net/manual/en/function.pack.php) and [unpack](https://www.php.net/manual/en/function.unpack.php) for writing and reading messages respectively. Java has the [ByteBuffer](https://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html) class. You can find equivalents in other languages too.  

It is recommended that you read more about the tracker [UDP protocol](https://www.bittorrent.org/beps/bep_0015.html).  

After the client verifies that the transaction id received is equal to the one sent, the peer list can be parsed and connections to each peer are initiated using the peer message protocol. 

### Peers
Communication with other peers is done to each ip and port addresses received from the trackers via the TCP protocol. The protocol for this is a binary protocol so it has similar format as the one in the UDP protocol. Due to the fact that your client connects to more than one peer, these connections have to be handled by your application. Some of the options to do that in C are:

* The [libevent](https://libevent.org/) library.
* [select](https://man7.org/linux/man-pages/man2/select.2.html), [poll](https://man7.org/linux/man-pages/man2/poll.2.html) and equivalent functions.
* Multithreaded socket functions.

The socket descriptor has to be set to non-blocking when using non-multithread methods. It's very possible some peers might not immediately reject your connection so setting connecting with timeouts is important to avoid delay. Doing this in C is not so straightforward unlike [Python](https://docs.python.org/3/library/socket.html#socket.create_connection), a way to do this is to use the [select](https://stackoverflow.com/a/2597774) function.  

Once 2 peers are connected, the first message that is exchanged between them is the handshake message.

#### Handshake Message
This is a required message. The peer that initiated the connected has to send this message to the accepting peer. Once this message is sent by the initiating peer, the accepting peer also replies with its own handshake message.  

The structure of the message is `<pstrlen><pstr><reserved><info_hash><peer_id>`. `<pstrlen>` is always _19_, it signifies the length of `<pstr>` which is always _BitTorrent protocol_, `<reserved>` is always 0 except you want to support extensions to the protocol, `<info_hash>` is the same as the info hash that was sent to the tracker and `<peer_id>` is the peer id that was also sent to the tracker. The length of this message is 68 bytes. Here's an example.

```
    pstr_len = 19 => 13 (1 byte)
    pstr = BitTorrent protocol => 426974546F7272656E742070726F746F636F6C (19 bytes)
    reserved = 0 => 0000000000000000 (8 bytes)
    info_hash = 123456789ABCDEF123456789ABCDEF123456789A //already in hex (20 bytes)
    peer_id = -BT0001-948911116432 => 2D4254303030312D393438393131313136343332 (20 bytes)

    // The hex values are concatenated
    message = 13426974546F7272656E742070726F746F636F6C0000000000000000123456789ABCDEF123456789ABCDEF123456789A2D4254303030312D393438393131313136343332 (68 bytes)
```

The only difference between the message sent and message received should be the peer_id. All other values should be the same. It is recommended the info_hash in the received message is checked to avoid issues. 

Note that the reply to the sent handshake message may also include additional messages like the bitfield/choke/have message.

You can read more details about the handshake message [here](https://wiki.theory.org/index.php/BitTorrentSpecification#Handshake).

#### Messages
This is the core of the BitTorrent protocol. The different types of messages are the keep-alive, choke, unchoke, interested, uninterested, have, bitfield, request, piece, cancel and port messages. You can see details [here](https://wiki.theory.org/index.php/BitTorrentSpecification#Messages).

The messages all begin with the message length which has a size of 4 bytes. Messages except the _keep-alive_ message also include the id (size = 1 byte), this id is used to identify the type of message that is send and received. The other parts of the message (for some message type) are specific to the message type.

There's a possibility that a client might receive more than one message type in one segment. It is recommended that the message's _message length_ value is used when reading the segment to demarcate the different types. It is also possible that less than the full message might appear in the segment. Your implementation should have a way of handling this. For example, you could use an internal buffer when reading from the socket. 

Some of the important messages are covered below

##### Choke/Unchoke message
These messages are used to denote if a peer wants to serve files to another peer. A _choke_ message is sent by a peer to inform another peer that it doesn't want to serve file to the other peer. The structure of the message is shown below

```
    length = 1 => 00000001 (4 bytes)
    id = 0 => 00 (1 byte)
    
    The hex values are concatenated together
    message = 0000000100
```

On receiving this message, it is recommended that the receiver refrains from requesting pieces from the sender until an _unchoke_ message is sent. The _unchoke_ message gives the receiver license to request pieces from the sender. The structure of the message is shown below

```
    length = 1 => 00000001 (4 bytes)
    id = 1 => 01 (1 byte)
    
    The hex values are concatenated together
    message = 0000000101
```

A _choke_ message can be sent at anytime so your implementation has to look out for it. A _choke_ message can also be sent alongside the _handshake_ message by the accepting peer to prevent an immediate request from the initiating peer.

##### Interested/Uninterested message
These messages are used to denote if a peer wants to receive files from another peer. An _interested_ message is sent by a peer to inform the receiver that it wants to receive a file from the it. On receiving the message, the recipient could choose to send a _choke_ message to prevent the sender from requesting for pieces from it. It could also send an _unchoke_ message to a choked sender. This is entirely based on the choking algorithm that the peer adopts. Note that the _interested_ message doesn't require a reply.  

The structure of an _interested_ message is shown below

```
    length = 1 => 00000001 (4 bytes)
    id = 2 => 02 (1 byte)
    
    The hex values are concatenated together
    message = 0000000102
```

The _uninterested_ message is sent by a peer to another peer if the receiver doesn't have the pieces the sender wants or its upload speed is too low for the sender. The recipient could choose to choke the sender or ignore the message. 

The structure of an _uninterested_ message is shown below

```
    length = 1 => 00000001 (4 bytes)
    id = 3 => 03 (1 byte)
    
    The hex values are concatenated together
    message = 0000000103
```

##### Have/Bitfield message
A peer has to signal the pieces it has to another peer. This is usually done in the reply to a handshake message (appended to the reply handshake message). They are 2 message types that can do this. One type could be used or a combination of both.  

The _have_ message is sent by a peer to signal that it has a piece. It's very possible for a peer to send several of them to the other peer. For example, if a file has 16 pieces and a peer has 14 of them, the peer could send 14 _have_ messages to the other peer each containing the different piece index. It is expected that a client has a way to keep record of pieces that each peers it's connected to has.  

The structure of a _have_ message is shown below

```
    length = 5 => 00000005 (4 bytes)
    id = 4 => 04 (1 byte)
    piece_index = 10 => 0000000A (4 bytes)
    
    The hex values are concatenated together
    message = 00000001040000000A
```

 The piece_index is the index of the piece denoted in the message. It is zero-based i.e first piece will be referenced as 0. 

Another way that a peer can signal the pieces it has is to send them all as a _bitfield_ message. A data structure that a peer can use to store the pieces it has is as a **bitarray**. This is an array that just stores bits. The length of the array is the number of pieces in the shared file(s). Each item in the array is either 1 or 0 if the piece is present or absent respectively. From the example above, the peer having 14 pieces might not have piece 10 and 13 so the bitarray would be '1111111110110111'. C has no native structure for bitarrays so an array of characters can be used where each character represents 8 pieces, this means the array for this would have _number of pieces/8_ items. Note that this length is rounded up to the nearest integer. Any spare bits at the end of the last character is set to 0. 

The structure of a _bitfield_ message is shown below

```
    length = 3 => 00000003 (4 bytes)
    id = 5 => 05 (1 byte)
    bitfield = 1111111110110111 => FFB7 (2 bytes)
    
    The hex values are concatenated together
    message = 0000000105FFB7
```

A peer can send a _bitfield_ or _have_ message once it has downloaded a complete piece to all the connected peers if it wants to share files.

##### Request message
The _request_ message is sent by a peer to download a piece from another peer. A piece is usually too large to download at a single time so it's advisable that the pieces are also divided into sub-pieces or blocks. According to the [spec](https://wiki.theory.org/index.php/BitTorrentSpecification#request:_.3Clen.3D0013.3E.3Cid.3D6.3E.3Cindex.3E.3Cbegin.3E.3Clength.3E), the recommended size should be set to 2^14 bytes (16 KB). If the piece is not evenly divisible by the block size, the last block would have a smaller size. A block should also have an offset attribute, this attribute signifies where it is located within the piece.

When requesting a block from another peer, it is recommended that the peer has the piece containing the block. The order of downloading a block is up to the client. It could be in order, random or any fancy algo out there. During downloads, it is recommended that a check is made for outstanding downloads that take more than a set time. These delayed blocks should be re-requested to speed up the entire process.

The structure of the _request_ message for a block in piece index 13 (zero-based) is shown

```
    length = 13 => 0000000D (4 bytes)
    id = 6 => 06 (1 byte)
    index = 13 => 0000000D (4 bytes)
    begin = 49152 => 0000C000 ( 4 bytes )
    block_length = 16384 => 00004000 (4 bytes)
    
    The hex values are concatenated together
    message = 0000000D060000000D0000C00000004000
```

The **index** is the index of the piece that contains the block. The **begin** value signifies the offset in bytes that the requested block is located within the piece i.e the block starts from the 49152th byte. From the structure above, the block is the 4th block in the piece. The **block_length** is the size of the block requested in bytes.

##### Piece message
This message is a response to the _request_ message. It contains the block data whose length is equal to the length requested. 

The structure of the _piece_ message for a block in piece index 13 (zero-based) is shown

```
    length = 9 => 00000009 (4 bytes)
    id = 7 => 07 (1 byte)
    index = 13 => 0000000D (4 bytes)
    begin = 49152 => 0000C000 ( 4 bytes )
    block = ADB40212..... (16384 bytes)
    
    The hex values are concatenated together
    message = 00000009070000000D0000C000ADB40212.....
```

The **index** is the index of the piece that contains the block. The **begin** is the offset in bytes that the requested block is located within the piece. In the example, the block starts from the 49152th byte. The **block** is the block data itself, the pseudocode shows the truncated form. The block data in the example above is 16KiB in size.  

When all the blocks of a piece has been downloaded and merged, a hash of the piece should be checked against the piece hash contained within the **pieces** element of the info dictionary of the torrent metafile. If the hash does not match, the piece should be discarded and its blocks should be requested for again.

##### Cancel/Port message
The _cancel_ message is mainly sent if the peer executes the [End Game](https://wiki.theory.org/index.php/BitTorrentSpecification#End_Game) algorithm. The message is sent as a form of politeness and to save bandwidth. I didn't implement the algorithm so there's no need for my client to send the  _cancel_ message.  

The structure of the message is the same as that of the _request_ message.

```
    length = 13 => 0000000D (4 bytes)
    id = 8 => 08 (1 byte)
    index = 13 => 0000000D (4 bytes)
    begin = 49152 => 0000C000 ( 4 bytes )
    block_length = 16384 => 00004000 (4 bytes)
    
    The hex values are concatenated together
    message = 0000000D080000000D0000C00000004000
```

The only difference between the _cancel_ message and the _request_ message is the id i.e 8 for the _cancel_ message type against 6 for the _request_ message.  

The _port_ message is only needed for clients that implements a DHT tracker. Like the _cancel_ message, I didn't implement DHT so there's no need for miy client to send the _port_ message.

It is sent to the receivers to notify them of the port the sender DHT's node is listening on. The structure of the message is shown here

```
    length = 3 => 00000003 (4 bytes)
    id = 9 => 09 (1 byte)
    port = 48879 => BEEF (2 bytes)
    
    The hex values are concatenated together
    message = 0000000109BEEF
```

### Writing a File
When a piece is verified, it is still in the RAM. Keeping all the pieces in memory can use up a lot of memory. For example, a 4gb file would use up ~4gb of memory if the pieces are all kept in memory. It is recommended that once a piece is verified, it should be written to a file except if the peer is currently sharing that piece.

If the torrent contains only one file, writing a piece to the file is a trivial matter of using the piece index to calculate the offset within the file and writing the piece from that offset. A lot of programming languages provide the mechanism for that.  

In the case of a torrent containing more than one file, it can be tricky. It's tricky because a piece might belong to more than one file and a file might contain more than one piece. If all the pieces are merged, they will be written to the files according to how the file names are arranged in the **files** list of the info dictionary. You should use the **length** element of each file to guide you on how much data has to be written to that file.

Your piece structure implementation should account for the files that each piece is written to along with the offset and length within each file.  

### Resources
* The [official](http://bittorrent.org/beps/bep_0003.html) and [unofficial](https://wiki.theory.org/index.php/BitTorrentSpecification) specifications are a must-read.
* Some of the BEPs found on this [page](http://bittorrent.org/beps/bep_0000.html) should also be read depending on your goals.
* Part [one](https://www.kristenwidman.com/blog/33/how-to-write-a-bittorrent-client-part-1) and [two](http://www.kristenwidman.com/blog/71/how-to-write-a-bittorrent-client-part-2/) of Kristen Widman's article on writing a bittorrent client are great to read too.
* My implementation in C can be found on [Github](https://github.com/goodyduru/simpletorrent).

Happy coding!
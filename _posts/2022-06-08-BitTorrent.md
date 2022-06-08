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
* info: This is a dictionary that contains file related details. Some of the details are: 
    * pieces: A long binary string consisting of the concatenation of the SHA-1 hash of each piece. Each hash is 20 bytes long so this makes the length of this string a multiple of 20. We can divide this string length by 20 to get the exact number of pieces to download.
    * piece length: The size of each piece in bytes. It's possible that the last piece might not be as large as this value so this value multiplied by the number of pieces might not be the accurate size of the file(s).
    * name: The name of file/directory to download.
    * files: This key only appears in when more than one file is being downloaded. It's a dictionary that contains the _path_ and _length_ of each file.
    * length: This key only appears **directly under the info dictionary** when only one file is being downloaded. It contains the size of the file in bytes. The value of this gives the accurate size of the file. When more than one file is downloaded, this key appears under the **files** key which is defined above. In this situation, a sum of these values gives the total size of the files.  
    It is important that an accurate size of the file(s) is calculated because this makes the accurate size of the last piece to be inferred. This can be easily calculated by using the modulo function if the total size is not divisible by the **piece length**.

Note that these aren't all the keys in the torrent file. You can read about more torrent keys and details [here](https://wiki.theory.org/index.php/BitTorrentSpecification#Metainfo_File_Structure).
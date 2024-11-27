+++
title = 'My Projects'
+++

I mainly build stuff to try out different tools and technologies. The following projects were made for this purpose.

### [A browser-only RSS Reader](https://offlinerss.goodyduru.com/)
I love RSS! In other to understand it better, I decided to build a client. This client works entirely in the browser. You can even read already-fetched articles offline.  The app's server component exists only as a CORS proxy. All the main processing happens on the client. This project gave me an opportunity to explore Conditional GETs, IndexedDB, Service Worker, and Go programming language. Check out the [source code](https://github.com/goodyduru/offline-rss).

### [An Icon Font Subsetter](https://iconsubset.goodyduru.com/)
In the course of improving Offline RSS perf, I wanted my font awesome font file to include only the icons I used. There were no browser tools for that, and so I built this. It works entirely in the browser. It uses the excellent [fonttools](https://github.com/fonttools/fonttools) written in Python. This gave me an opportunity to explore Web Assembly using [Pyodide](https://pyodide.org) and Web Workers. You can find the source code [here](https://github.com/goodyduru/font-subset).

### [Nitrows](https://github.com/goodyduru/nitrows)
I have always found Websockets to be a fascinating and yet obsure communication protocol. What better way to understand it than building one. In addition, I wanted to apply some of the lessons I learnt from a [Performance Engineering class](https://youtube.com/playlist?list=PLUl4u3cNGP63VIBQVWguXxZZi0566y7Wf&si=OBikOBBBstQ_GpdO) I took. I built this one in C. Check out the [source code](https://github.com/goodyduru/nitrows).

### [Simpletorrent](https://github.com/goodyduru/simpletorrent)
Built this for the same reason as Nitrows. Bittorrent felt like magic, and I wanted to rectify that. This project was that rectification. Loved this project because it gave me an opportunity to play around with my newly acquired C programming language skills and BSD sockets. There are lots of embarassing lines of code in there, but still feel proud of it. Here's the [source code](https://github.com/goodyduru/simpletorrent).

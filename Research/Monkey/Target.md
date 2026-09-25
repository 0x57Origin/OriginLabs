# Target: https://github.com/monkey/monkey

---

## Lab 1 - Pick and Pin the Target

Repo: https://github.com/monkey/monkey  
Why this one: small C HTTP server, real project, fits a research loop  
Commit: e257e6383585ac281ab7de2797fbc5eca64f804f  
Path: /home/kali/Desktop/monkey  
Protocol: HTTP/1.1  
Listen: 127.0.0.1:2001 (binary bound 0.0.0.0:2001)  
Build: cmake -S . -B build && cmake --build build  
Run: ./build/bin/monkey -c build/conf -p 2001  
Proof: curl -v http://127.0.0.1:2001/ returned an HTML page and left the connection intact  

---

## What is Monkey?

Small web server written in C for Linux. If you start it, it will wait on a port. A browser or curl command sends GET /. Monkey then reads the request and goes to find the file like the HTML in htdocs; thus it sends us back. This is a website. It is written to stay small and not eat up RAM. People use that kind of server on little devices or as a piece inside another program.

Structure:

- mk_core/ - helper tools (memory, files, events)
- mk_server/ - the actual HTTP brain
- plugins/ - optional extras
- htdocs/ - the demo website we already loaded
- fuzz/ - extra programs for later crash-testing

Summary: A small server written in C to serve files and it uses HTTP.

## Clone the project

    git clone https://github.com/monkey/monkey.git
    cd monkey
    git rev-parse HEAD

git rev-parse HEAD = This will print out the ID for the code version we have right now.

    cmake -S . -B build

-S is the source folder. -B build is the folder CMake writes build files into.

    cmake --build build

This command compiles the Monkey repo using the CMake files already put in build.

Now before we continue there was an error while running with CMake. -o means use this folder as the website. htdocs is the folder in our Monkey repo. It holds the HTML/JS/CSS the server sends when you open /.

So now ./build/bin/monkey -o htdocs means run Monkey and serve the files in the htdocs folder, but it still failed because Monkey also needs a config file and -o does not point it at build/conf.

So here is how we fix it:

    cd ~/Desktop/monkey
    ls build/conf
    ./build/bin/monkey -c build/conf -p 2001

Result:

    CMakeFiles  cmake_install.cmake  Makefile  monkey.conf  monkey.mime  plugins.load  sites  tls.conf
    Monkey HTTP Server v1.8.10
    Built : Sep 24 2026 21:12:41 (/usr/bin/cc 15.2.0)
    Home  : https://monkeywebserver.com
    [+] Process ID is 153446
    [+] Server listening on 0.0.0.0:2001
    [+] 4 threads, may handle up to 1024 client connections
    [+] Loaded Plugins:
    [+] Linux Features: TCP_FASTOPEN SO_REUSEPORT

Breakdown:

- It gives us a process ID which is 153446.
- Server listening on 0.0.0.0:2001 = It is accepting TCP connections on port 2001 on every interface, not just localhost. 0.0.0.0 means all NICs.
- 4 threads, may handle up to 1024 client connections = 4 worker threads. About 1024 clients max with this config.
- Loaded Plugins = Nothing extra is loaded right now.
- Linux Features: TCP_FASTOPEN SO_REUSEPORT = Kernel socket options it turned on. Faster/smarter accept on Linux. It is not a bug.

# Curl Command

    curl -v http://127.0.0.1:2001/

    *   Trying 127.0.0.1:2001...
    * Established connection to 127.0.0.1 (127.0.0.1 port 2001) from 127.0.0.1 port 46636
    * using HTTP/1.x
    > GET / HTTP/1.1
    > Host: 127.0.0.1:2001
    > User-Agent: curl/8.20.0
    > Accept: */*
    >
    * Request completely sent off
    < HTTP/1.1 200 OK
    < Server: Monkey/1.8.10
    < Date: Fri, 25 Sep 2026 01:13:57 GMT
    < Last-Modified: Fri, 25 Sep 2026 01:10:15 GMT
    < Content-Type: text/html
    < ETag: "6ab5c9f7-2254"
    < Content-Length: 8788
    <
    <!DOCTYPE html>

## TCP

My Kali opened a connection to Monkey. 46636 is curl's temporary port. Same machine, 2 ports.

## What curl command sent

    GET / HTTP/1.1
    Host: 127.0.0.1:2001
    User-Agent: curl/8.20.0
    Accept: */*

Plain HTTP/1.1:

- GET / : Give me the homepage
- Host: Which site is it
- User-Agent: I am curl
- Accept: */* : I take anything

Later on we will find out where in C code does it land. A parser is a computer program that takes a sentence or chunk of data and breaks it down so the computer can understand its meaning. Monkey has a parser too. That text is what Monkey's parser reads.

## What Monkey sent back

    HTTP/1.1 200 OK
    Server: Monkey/1.8.10
    Date: ...
    Last-Modified: ...
    Content-Type: text/html
    ETag: "6ab5c9f7-2254"
    Content-Length: 8788

- 200 = Found the file and here it is
- Server = Gives us the exact version
- Content-Type = this is HTML
- Content-Length: 8788 = body is exactly 8788 bytes
- ETag / Last-Modified = caching info for that file

The HTML is in htdocs/index.html which is the default site.

Connection #0 to host 127.0.0.1:2001 left intact = This means it did not slam the socket shut; keepalive is on.

What does this tell us so far? Monkey speaks HTTP/1.1, serves a static file, answers 200 OK, body size matches Content-Length. That is enough information.

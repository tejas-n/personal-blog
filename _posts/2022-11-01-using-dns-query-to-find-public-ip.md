---
layout: post
title: 'Using DNS Queries to Find the Public IP Address of a Machine'
date: '2022-11-01 13:00:00 +1100'
categories: java
---
We were recently implementing a feature at our company which required our custom Android devices to periodically report their public IPv4 addresses to our back end. We wanted a solution which was fast and reliable. The easiest solution to solve this was to use some existing web service like <https://ifconfig.so/> and all you had to do was to make an HTTP GET request and it would return your public IPv4 address as a response. This solution isn't reliable though since it isn't owned by us and could go away in future. It is a fast solution but still not the fastest and there's still scope for improvement!

So how can we improve this solution? An obvious way might be to host your own web service. This will make it:
    
1. More reliable since we'd own it. There's no risk of the service going away unexpectedly.

2. Faster since only the devices owned by us would ping the web service. Less network traffic to the web server would mean better response time and server uptime(all things being comparable to the other public web services).

I was having a quick chat about it with my boss who pointed out to me that there was a faster way to get the public IPv4 address using DNS servers.

```
dig +short myip.opendns.com @resolver1.opendns.com -4
```
You can achieve the same thing on a Windows machine with Powershell
```
Resolve-DnsName -Name myip.opendns.com -Server resolver1.opendns.com
```

So what's going under the hood? The [OpenDNS](https://www.opendns.com/) servers have been programmed to return the public IPv4 address of the machine which tries to query for the A record of `resolver1.opendns.com`. I decided to implement this as a Java library which is now hosted at <https://github.com/tejas-n/echo-my-ip>. I'll explain how the library works but before we need to understand how DNS queries work.

# Format of DNS query
A DNS packet has the following structure

|: Header :||||||||||||||||
|: Question :||||||||||||||||
|: Answer :||||||||||||||||
|: Authority :||||||||||||||||
|: Additional :||||||||||||||||

We can ignore the "Authority" and "Additional" sections since they are irrelevant here. The DNS query that we'll send will contain the Header and the Question section while the response to the query will consist of the Header, Question, and Answer sections.

The structure of the header:

|15|14|13|12|11|10|9|8|7|6|5|4|3|2|1|0|
| : ID     :               ||||||||||||||||
| :QR: | : Opcode : ||||AA|TC|RD|RA| : Z : ||| RCODE ||||
| : QDCOUNT :       ||||||||||||||||
| : ANCOUNT :       ||||||||||||||||
| : NSCOUNT :       ||||||||||||||||
| : ARCOUNT :       ||||||||||||||||


Now let's try to understand what the fields mean and construct the header of our query along the way:

|**Bit**|**Meaning**|**Description**|**Value we'll set in our query**|
|:ID:| :Identifier:| A 16-bit unique identifier assigned by the client to the query. The DNS server will use the same ID in the header of it's corresponding reply which helps the clients to match a response with a query.|:0x0001:|
|:QR:| :Query or Response:| A 1-bit field which identifies if the message is a query(0) or a response(1).| :0: |
|:Opcode:| :Operation Code: | A 4-bit field that specifies the type of query in the message. The possible values are 0 for a standard query, 1 for an inverse query, and 2 for a server status request.|:0x0000:|
|:AA:|:Authorative Answer:| A 1-bit field which specified if the responding name server is an authority for the domain name in the question section. This field is only valid in responses.|:0:|
|:TC:|:Truncation:| A 1-bit field which denotes if the response is truncated.|:0:|
|:RD:|:Recursion Desired:| A 1-bit field which signals to the name server if it should resolve the query recursively|:0:|
|:RA:|:Recursion Available:| A 1-bit field which specifies if recursive query support is available on the name server. This is valid only in responses|:0:|
|:Z:|:Reserved:|Reserved for future use.|:0:|
|:RCODE:|:Response Code:| A 4-bit field which denotes the response code. The meaning of the codes is out of the scope of this article since we are going to ignore it anyway|:0x000:|
|:QDCOUNT:|:Question Count:| A-16 bit field specifying the number of entries in the question section|:0x0001:|
|:ANCOUNT:|:Answer Count:| A 16-bit field specifying the number of resource records in the answer section.|:0x0000:|
|:NSCOUNT:|:Authority Records Count:| A 16-bit field specifying the number of resource records in the authority records section|:0x0000:|
|:ARCOUNT:|:Additional Records Section:| A 16-bit field specifying the number of resource records in the additional records section|:0x0000:|

Now let's look at the "Question Section" structure:

|**Field**|**Description**|**Length in bytes**|
| NAME |Name of the requested resource|Variable|
| TYPE |Type of requested resource(A, AAAA, etc)|2|
| CLASS |Class Code|2|


Combining all the sections that we've defined above, this is how I've defined the request packet in my library:

```
 private static final byte[] requestPacket = {
            // Transaction ID: 0x0001
            0x00, 0x1,
            // Flags: 0x0000 (Standard query without recursion)
            0x00, 0x00,
            // Questions: 1
            0x00, 0x01,
            // Answers: 0
            0x00, 0x00,
            // Authority Records: 0
            0x00, 0x00,
            // Additional Records: 0
            0x00, 0x00,
            // Query for myip.opendns.com
            0x04, 0x6d, 0x79, 0x69, 0x70, 0x07, 0x6f, 0x70, 0x65, 0x6e, 
            0x64, 0x6e, 0x73, 0x03, 0x63, 0x6f, 0x6d, 0x00,
            // Type A query
            0x00, 0x01,
            // IN
            0x00, 0x01
    };
```

Now let us look at the code which sends a Type A query to OpenDNS Servers to find out its public IP

```
 public String myIPv4Address() throws IOException {
        byte[] mBuffer = new byte[50];
        DatagramPacket sendPacket =
                new DatagramPacket(requestPacket, requestPacket.length, InetAddress.getByName(OPENDNS_SERVER_IP), 53);
        DatagramPacket packet = new DatagramPacket(mBuffer, mBuffer.length);
        DatagramSocket socket = new DatagramSocket();
        socket.send(sendPacket);
        socket.receive(packet);
        StringBuilder ip4StringBuilder = new StringBuilder();
        for (int i = 46; i < 50; i++) {
            ip4StringBuilder.append(mBuffer[i] & 0xFF);
            if (i != 49) {
                ip4StringBuilder.append(".");
            }
        }
        String ipv4Address = ip4StringBuilder.toString();
        if (!ipv4Address.equals("0.0.0.0")) {
            return ipv4Address;
        } else {
            return null;
        }
    }
 ```

I think the first few lines of the code are self-explanatory. We send the request packet that we've already defined to the OpenDNS server on port 53 over UDP. Now let's try to understand why we are ignoring the first 46 bytes of the response. The answer section of the response looks like this:

|**Field**|**Description**|**Length in bytes**|
|NAME|Name of the requested resource or pointer to the resource name if it was defined before this|2 bytes if pointer, otherwise variable|
|TYPE|Type of resource(A, AAAA, etc)|2|
|CLASS|Class Code|2|
|TTL|Count of seconds that the requested resource stays valid|4|
|RDLENGTH|Length of `RDATA` field in bytes|2|
|RDATA|Record data|Defined by `RDLENGTH`|

We want to find out how many bytes in the response we need to skip to reach the `RDATA` field which should have the IP address that we need. Let's find out how many bytes the `NAME` field of the answer section is first. DNS packets support compression that let you define a pointer to the resource name if it was defined before this. In our case, the domain name already appears in the `NAME` field of the question section, so the `NAME` field of our answer section will contain a 2 bytes pointer to that field.

As mentioned at the beginning of the post, the response will consist of a header, the question section that we sent in our request, and the answer section described above. To reach `RDATA` of the answer section we can skip 12 bytes of the header section, 22 bytes of the question section, 2 bytes of the `NAME` field of the answer section, 2 bytes of `TYPE`, 2 bytes of `CLASS`, 4 bytes of `TTL`, and 2 bytes of `RDLENGTH` i.e. 46 bytes in total. Our IP address will be present from byte 46 to byte 49.

# Benchmarks

The latency was measured by running the tools ten times and calculating an average of the latency.

|Service|Latency|
|[echo-my-ip library](https://github.com/tejas-n/echo-my-ip) which is based on the DNS query method described in the article |15 ms|
|[Ipify](https://api.ipify.org) REST API call|315 ms|
|<https://ifconfig.so> REST API call|328 ms|

# Conclusion

In the article, we saw how OpenDNS servers could be used to find out the public IPv4 address of the machine. This method is faster than using a dedicated web service for echoing back the IP. Now the question is if it's worth it. Like everything in software development, it depends. For the majority of use cases, it won't matter what you use, but if latency is critical for your use case, this method surely does the job better than using a web service.

# References

<https://en.wikipedia.org/wiki/Domain_Name_System>

<https://mislove.org/teaching/cs4700/spring11/handouts/project1-primer.pdf>

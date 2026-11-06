# Wireshark SOC Lab – Investigation Notes

This document records the investigation procedures used in six guided Wireshark labs. Each section explains how the relevant traffic was located, which Wireshark features and display filters were used, and how packet evidence supports the finding.

The examples and observations are from the NetworkWalks practice captures. IP addresses and activity described here belong to those lab environments.

---

## 1. Starting a capture and reading the Wireshark interface

### Starting a capture

1. Open Wireshark.
2. Select the active network interface. The interface activity graph helps identify which interface is carrying traffic.
3. Double-click the interface to start capturing packets.
4. Enter a display filter in the filter bar to narrow the packet list.
5. Select a packet to inspect its fields in the packet-details pane.
6. Check the status bar for the total captured packet count and the number of packets currently displayed.

### Packet-list columns

| Column | Information |
|---|---|
| No. | Packet number in the capture |
| Time | Time at which the packet was captured |
| Source | Address of the packet sender |
| Destination | Address of the packet recipient |
| Protocol | Protocol identified by Wireshark |
| Length | Packet size in bytes |
| Info | Summary of packet details, such as TCP flags, ports, or an application request |

The packet list gives a quick overview. The packet-details pane provides protocol fields for the selected packet, while the packet-bytes pane shows the packet data in hexadecimal and ASCII where available.

### Display filters

The labs use display filters to focus on relevant packets. A display filter changes which packets are shown; it does not remove other packets from the capture.

---

## 2. Lab 3 – Identify the attacker's IP address

**Scenario:** Unusual traffic is directed at a company web server. The investigation identifies the source host, target host, probed ports, server responses, and likely attack type.

### Step 1: Show TCP SYN packets

Apply:

```wireshark
tcp.flags.syn == 1
```

A SYN flag is used to initiate a TCP connection. The filter narrows the packet list to packets with the SYN flag set. It can include both initial SYN packets and SYN-ACK replies, so packet direction and the other TCP flags still need to be checked.

### Step 2: Identify the source generating repeated attempts

Inspect the **Source** column and the **Time** column. Most hosts send only a small number of connection-initiation packets in this capture. One source produces a long sequence of SYN packets within a short period.

The repeated source address is the candidate scanning host. The sequence should be checked against the Destination column to confirm that the packets are directed toward the same target.

### Step 3: Identify the target

Inspect the **Destination** column for the packets in the repeated sequence. The destination address identifies the server receiving the connection attempts.

### Step 4: Compare the TCP ports

The Info column displays the port pair in this format:

```text
source-port → destination-port [SYN]
```

The number before the arrow is the source port. The number after the arrow is the destination port.

In this capture, the source port remains the same while the destination port changes. This indicates that the source is attempting connections to multiple services on the target.

### Step 5: Count the ports and inspect server replies

Open **Statistics → Conversations**, then select the **TCP** tab. Locate the conversations between the suspicious source and the target. Each distinct destination-port conversation represents a port attempt in this lab.

To focus on packets involving the suspected source, apply:

```wireshark
ip.addr == <ip-address>
```

Inspect the server's replies in the Info column:

- `[SYN, ACK]` is the response expected when a TCP port accepts the connection attempt.
- `[RST, ACK]` indicates that the server reset the connection attempt; in this lab, it is the response associated with closed ports.

### Finding

The capture shows one host attempting connections to many different destination ports in a short period. The server responds differently depending on whether the probed port is open or closed. This pattern is consistent with a **port scan**.

A SYN packet by itself does not prove malicious activity. The conclusion comes from the repeated attempts, changing destination ports, common target, and server responses.

---

## 3. Lab 4 – Follow a TCP stream

**Scenario:** An office chat application sends messages in plain text. The investigation locates the chat conversation and inspects the exchanged messages.

### Step 1: Locate the unusual TCP conversation

Open **Statistics → Conversations** and select the **TCP** tab. Compare the ports used by the conversations. Most web traffic in this capture uses ports 80 or 443. A separate conversation uses TCP port 5000, which is the office chat traffic in this lab.

### Step 2: Filter the chat traffic

Apply:

```wireshark
tcp.port == 5000
```

Inspect the first packets in the filtered list. The first SYN packet identifies the host that initiated the TCP connection. Its Source column shows the client address, and its Destination column shows the server address.

### Step 3: Reassemble the conversation

1. Select a packet that contains application data (a packet with a non-zero TCP payload length).
2. Right-click the packet.
3. Choose **Follow → TCP Stream**.
4. Read the reassembled conversation.
5. Check the stream number in the Follow window title and the conversation size shown at the bottom.

The Follow TCP Stream view combines the data exchanged in the selected TCP conversation. It is easier to understand a chat in this view than by reading individual packets separately.

### Step 4: Identify the direction of messages

The Follow TCP Stream window uses different colours for the two directions of the conversation. In this lab, client messages appear in red and server messages appear in blue. This makes it possible to distinguish which endpoint sent each message.

### Finding

The reassembled stream contains the plain-text office chat and information exchanged between the employee and IT. The packet direction and endpoint addresses help establish who sent each message.

Plain-text application traffic can expose internal communications and sensitive information to anyone able to capture the traffic. An encrypted communication channel should be used for this type of application.

---

## 4. Lab 5 – Find information inside HTTP streams

**Scenario:** A staff member visits several intranet pages. One page contains a note that is not visible in the normal browser view.

### Step 1: Filter HTTP traffic

Apply:

```wireshark
http
```

Inspect the HTTP requests in the Info column and locate the request for the relevant page.

### Step 2: Follow the page's TCP stream

Right-click the relevant HTTP packet and choose **Follow → TCP Stream**. The reassembled stream contains the HTTP request and the server response.

Inspect the HTML response for comment markers:

```html
<!-- hidden comment -->
```

Text between `<!--` and `-->` is an HTML comment. A browser does not display it as normal page content, but it remains part of the HTML source and can be visible in an unencrypted HTTP response.

### Step 3: Identify the stream and response direction

The Follow TCP Stream window shows the stream number. The server response is displayed separately from the client request. In this lab, the hidden note is in the server-to-client part of the conversation.

The HTTP response status line can also be inspected. For example:

```http
HTTP/1.1 200 OK
```

The status code `200` indicates that the request succeeded.

### Step 4: Locate a downloaded file

Apply:

```wireshark
http.request.uri contains "download"
```

Inspect the matching GET request in the Info column to identify the requested file. Follow the corresponding TCP stream to inspect the returned text content and determine what information the file contains.

### Finding

Information hidden from the normal browser display is not necessarily hidden from packet inspection. HTML comments and files transferred over unencrypted HTTP can expose internal information. Secrets and access details should not be placed in HTML comments or unprotected resources.

---

## 5. Lab 6 – HTTP plain-text credentials

**Scenario:** A staff member signs in to an older web portal. The investigation identifies the protocol, the form submission method, the client, and whether the submitted information is protected.

### Step 1: Identify HTTP traffic

Apply:

```wireshark
http
```

The login request appears as an HTTP POST request in the packet list. A POST request is commonly used to submit form data to a server.

To show only POST requests, apply:

```wireshark
http.request.method == "POST"
```

### Step 2: Identify the login request and client

Inspect the Info column for the request path, such as `POST /login`. The Source column identifies the client that sent the request. The request method and URI can also be inspected in the packet details.

### Step 3: Inspect submitted form fields

1. Select a POST packet.
2. Expand the packet details.
3. Locate **HTML Form URL Encoded**.
4. Inspect the form fields shown in that section.

When the form data is sent without encryption, the submitted fields can be readable in the packet details. If the capture contains more than one POST request, inspect the relevant requests individually to compare the attempts and identify which one corresponds to the successful sign-in.

### Step 4: Count the form submissions

Keep the POST filter applied and count the matching login submissions. The displayed packet count in the Wireshark status bar can help when the filter is limited to the relevant login requests.

### Finding

HTTP does not encrypt the request body. If a login form submits credentials over plain HTTP, someone with access to the captured traffic may be able to read the submitted values.

HTTPS protects data in transit through encryption. Login pages and other sensitive web traffic should use HTTPS rather than plain HTTP.

---

## 6. Lab 7 – Detect FTP brute-force activity

**Scenario:** An FTP server owner reports repeated failed sign-ins. The investigation identifies the protocol, source host, target server, attempted username, failed responses, and whether a login eventually succeeded.

### Step 1: Identify the protocol

Open **Statistics → Protocol Hierarchy**. Expand the protocol tree and inspect the protocols carried over TCP. FTP traffic indicates that the login attempts use the File Transfer Protocol.

FTP authentication uses commands such as:

- `USER` – submits the username.
- `PASS` – submits the password.

### Step 2: Filter FTP traffic

Apply:

```wireshark
ftp
```

Inspect the FTP requests and server responses. The Source and Destination columns help distinguish the client sending login attempts from the FTP server responding.

### Step 3: Inspect username and password commands

To display username requests:

```wireshark
ftp.request.command == "USER"
```

To display password requests:

```wireshark
ftp.request.command == "PASS"
```

Inspect the request details and Info column to see the authentication command sequence. Repeated attempts against the same username are relevant evidence when investigating possible brute-force activity.

### Step 4: Identify failed and successful responses

FTP response codes provide evidence of the authentication result.

| Response code | Meaning in this exercise |
|---|---|
| `530` | Login incorrect |
| `230` | Login successful |

Show failed-login responses:

```wireshark
ftp.response.code == 530
```

Show successful-login responses:

```wireshark
ftp.response.code == 230
```

The displayed packet count for the `530` filter can be used to count the failed responses in the capture.

### Step 5: Correlate the successful response with the stream

Select the successful response and choose **Follow → TCP Stream**. Inspect the preceding FTP commands and responses to understand the sequence leading to the successful login.

### Finding

Repeated password attempts against one account followed by a successful authentication are consistent with a **brute-force attack** in this lab. The investigation correlates the source host, target server, username requests, failed response count, and successful response.

FTP traffic without encryption can expose authentication information. Secure file-transfer methods should be used, and credentials found in captures should not be copied into public reports.

---

## 7. Lab 8 – Investigate a VoIP call

**Scenario:** A call is made through a company VoIP system. The investigation identifies the call-signalling protocol, voice-media protocol, caller details, codec, answering party, and the security issue demonstrated by the call.

### Step 1: Identify the call setup protocol

Apply:

```wireshark
sip
```

Inspect the Info column for SIP messages such as:

- `INVITE` – starts the call setup.
- `180 Ringing` – indicates that the called endpoint is ringing.
- `200 OK` – indicates that the request was accepted.

These messages show that **SIP (Session Initiation Protocol)** handles call signalling and setup.

### Step 2: Identify the protocol carrying the voice

Apply:

```wireshark
rtp
```

The capture contains many small RTP packets sent at regular intervals. **RTP (Real-time Transport Protocol)** carries the voice media itself, while SIP handles the signalling.

### Step 3: Inspect the caller extension and name

To locate the call initiation packet, apply:

```wireshark
sip.Method == "INVITE"
```

Select the INVITE packet and expand:

**Session Initiation Protocol → Message Header**

Inspect the **From** field. It contains the caller's display name and SIP address, from which the caller name and extension can be identified.

### Step 4: Identify the codec

Apply the RTP filter:

```wireshark
rtp
```

Select an RTP packet and expand **Real-Time Transport Protocol**. Inspect the **Payload type** field. In this capture, the payload type identifies **ITU-T G.711 PCMU**, also referred to as PCMU.

A codec is used to encode and decode the audio carried in the media stream.

### Step 5: Identify who answered and inspect the audio

1. Open **Telephony → VoIP Calls**.
2. Select the call in the list.
3. Choose **Play Streams**.
4. Press Play to listen to the call.
5. Use the first spoken response to identify the answering party.

The To field in the SIP INVITE can also provide a name associated with the called endpoint. The audio playback provides an additional way to verify who answered.

### Step 6: Understand the security issue and mitigation

The lab demonstrates that voice carried in unencrypted RTP can be reconstructed and listened to by someone with access to the capture. Sensitive information spoken during a call can therefore be exposed.

**SRTP (Secure RTP)** protects the voice-media packets. TLS can be used to protect SIP signalling where supported. Encrypting both the media and signalling helps prevent captured traffic from exposing call contents and signalling information.

### Finding

SIP establishes and manages the call, while RTP carries the voice. The RTP payload type identifies the codec used in the capture. The VoIP Calls window and Play Streams feature make it possible to inspect the call and verify the answering party. The security issue is the exposure of voice media when RTP is not encrypted.

---

## 8. Useful Wireshark display filters

| Filter | Purpose |
|---|---|
| `tcp.flags.syn == 1` | Display TCP packets with the SYN flag set |
| `ip.addr == <ip-address>` | Display packets where the IP is either source or destination |
| `tcp.port == <port>` | Display TCP packets involving a particular port |
| `http` | Display HTTP traffic |
| `http.request.method == "POST"` | Display HTTP POST requests |
| `http.request.uri contains "download"` | Find HTTP requests whose URI contains `download` |
| `ftp` | Display FTP traffic |
| `ftp.request.command == "USER"` | Display FTP username commands |
| `ftp.request.command == "PASS"` | Display FTP password commands |
| `ftp.response.code == 530` | Display FTP failed-login responses |
| `ftp.response.code == 230` | Display FTP successful-login responses |
| `sip` | Display SIP signalling |
| `sip.Method == "INVITE"` | Display SIP INVITE requests |
| `rtp` | Display RTP media packets |

Replace placeholders such as `<ip-address>` and `<port>` with the value relevant to the investigation.

---

## 9. Investigation workflow

1. Read the scenario and identify the activity being investigated.
2. Start the capture on the active network interface.
3. Apply a display filter related to the protocol, flag, host, or port in question.
4. Compare the Source, Destination, Protocol, Length, and Info columns.
5. Open packet details to inspect fields that are not visible in the packet list.
6. Use **Statistics → Conversations** to examine communication between endpoints.
7. Use **Statistics → Protocol Hierarchy** to understand which protocols are present.
8. Use **Follow → TCP Stream** to reconstruct a TCP conversation when application data needs to be reviewed.
9. Correlate requests and responses before deciding what happened.
10. Record the evidence supporting the finding and exclude credentials or other sensitive values from public documentation.

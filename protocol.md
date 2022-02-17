# What is this document?

**THIS IS NOT A SPECIFICATION, IS LIKELY LINKING TO OUTDATED/INCORRECT LINKS, THERE WILL BE TYPOS.
THIS IS AN EXERCISE IN BOTH ORGANIZING INFORMATION AT A VERY HIGH LEVEL (e.g., within a conceptual model like the OSI framework)
TO PREPARE AND ARTICULATE THE BIG PICTURE. There is some details from other documents that are probably out of the scope
of this document but are kept here just to not lose track of them.**

This is an overview of the APRS protocol as it generally is implemented at the beginning of 2022. 
The goal is to unwind and connect v1.0, v1.1, proposed v1.2 that is implemented in practice,
and other useful information on [aprs.org](http://aprs.org) in perparation for future
APRS development.

Please comment/make pull requests/add info as we try to piece together all the documentation.
Link to the documents on aprs.org and work to present the info in a concise way.

# What is APRS?

APRS is an entire ecosystem of interconnected technologies supporting digital data messaging over amateur radio.
Among the many things that are a part of the APRS network include
compatible radio transmit and receive hardware,
signal decoding hardware & software (based on the APRS protocol),
internet servers for collecting and aggregating data on the radio network,
and software & hardware for displaying and analyzing the data (e.g., mobile apps, desktop apps, web apps, smart displays).

# APRS in the OSI Layer Framework

To understand all the components of APRS, it's helpful to see where it fits in an industry standard
framework. Here, we'll use the [OSI model](https://en.wikipedia.org/wiki/OSI_model)

> The Open Systems Interconnection model (OSI model) is a conceptual model that characterises
and standardises the communication functions of a telecommunication or computing system
without regard to its underlying internal structure and technology.
Its goal is the interoperability of diverse communication systems with standard communication protocols.

This allows us to think of APRS is an open, modular way that allows for networ compatibillity as it expands
to new technology and applications.

## OSI Layers

### Layer 1: Phyiscal Layer

APRS has been implemented in several different ways on the physical layer.
Most commonly it is implemented as [Bell 202 AFSK](https://en.wikipedia.org/wiki/Bell_202_modem) at VHF frequencies (e.g., 144.390MHz in the US).
Other physical layer implementations include HF (e.g., 30m and 40m bands) using 300 Baud AFSK (1600Hz/1800Hz) using SSB,
or [other digital modes HF such as PSK](http://wa8lmf.net/APRS_PSK63/index.htm).


### Layer 2: Data Link Layer

APRS most commonly uses [AX.25 v2.2](http://www.ax25.net/AX25.2.2-Jul%2098-2.pdf) for the data link layer.
Frames are transmitted as unnumbered information (UI) frames, as described by the spec.

Additionally, it is common practice to use [p-persistance CSMA](https://en.wikipedia.org/wiki/Carrier-sense_multiple_access) at the data link layer.


### Layer 3: Network Layer

APRS most commonly uses digipeating to repeat packets between stations to get to their destination.
This addressing usually takes place in the Data Link Layer (AX.25), which AX.25 v2.2 identifies as possibly not theoretically ideal.

A common method of networking is to use the [New n-N Paradigm](http://aprs.org/fix14439.html).
The New n-N Paradaigm definition isn't easily found on aprs.org (it's kind of [here](http://aprs.org/newN/relayFIX.txt)), requires knowledge of the older TNC firmwares (UIDIGI, UITRACE, UIFLOOD),
so it can be very frustrating for someone making new hardware to figure out.
Here's an attempt at a summary, but probably inaccurate due to a lack of clear info:
- Permanent, fixed, digipeaters with good coverage recognize the alias WIDE* (where * indicates a wildcard) as its digipeating call sign.
- Mobile, temporary digipeaters "filling in" gaps in coverage only recognize WIDE1 (i.e., ignore WIDE2) as its digipeating call sign.
- When digipeating, digipeaters change the digipeater list before digipeating. First, they decriment the SSID of the WIDE alias they recognized.
Second, they prepend their actually callsign before the alias with the "has been repeated" bit set for their call sign.
Third, if the SSID of the WIDE alias is decrimented to zero, it sets the "has been repeated" bit for the wide alias.

Example:
If KD9PDP-0 is fixed a digipeater, and it sees the digipeater path `WIDE1-1, WIDE2-1`, it would replace it with `KD9PDP-0, WIDE1-0, WIDE2-1`
while setting `KD9PDP-0` and `WIDE1-0` "has been sent" bits.

For the originating station that first sets the digipeater path, it is suggested that the use `WIDE1-1, WIDE2-1` (at max, 2-hop) in dense settings
and `WIDE1-1, WIDE2-2` (at max, 3-hop) when remote.

### Layer 6: Presentation Layer

APRS encodes and compresses data using a variety of methods, outlined in the APRS spec.
The spec is spread out across different documents at [aprs.org](http://aprs.org/).
Among the interesting encoding include optionally using base91 for encoding,
conventions for encoding time and date,
conventions for identifying the station type ([symbols](http://www.aprs.org/symbols.html)),
and several application-specific compression methods.

### Layer 7: Application Layer

A wide variety of application data can be transmitted via APRS, and is described in detail in the spec.


## Deviations from the OSI Model

Just as a note, APRS commonly deviates from a clean OSI model by mixing high layers in the data link layer:
- network addressing is done in the data link layer (see Layer 3, below)
- encoding of the software/hardware method used to create the packet is included in the data link layer
- and mic-e encoding uses the data link layer for application layer information


# APRS over AX.25 Protocol Overview

When APRS uses AX.25 for the data link layer, the spec defines some additional details, conventions, and nuances.

[WORK IN PRFORESS. Starting with things that aren't easily found in the spec]

## Destination Address Encoding
APRS follows AX.25 spec for destination address encoding.
In APRS, the destination field set to an alias for the APRS network or to an alias for an altnet.
If set to an alias for the APRS network, frames will be digipeated and heard by others listening on the APRS network.
If set to an altnet alias, messages may only be received by other stations monitoring for that altnet.
There are many acceptable aliases for the APRS network, 
the meaning of the destination call sign alias is maintained [here](http://aprs.org/aprs11/tocalls.txt) (and possibly moving to a new open maintenance structure).


AX.25 allows the use of an SSID (integer from 0 to 15) to be appended to a cal sign to distinguish between two stations with the same call sign.
APRS sets the SSID for destinations to `0`, but allows for non-zero SSIDs to indicate routing.
**QUESTION FOR THE COMMUNITY: DOES ANYONE ACTUALLY USE NON-ZERO DEST SSIDs?**


The AX.25 spec encodes additional information in to the SSID byte to include additional functionality (i.e., the `C` and `RR` bits in AX.25 v2.2).
These bits are set in APRS as follows:
- **C (1 bit):** AX.25 spec shows that UI frames are [command frames](http://aprs.org/aprs11/C-bits-SSID.txt), so set to `1` in the destination address.
- **RR (2 bits):** not specified yet, set to `11` ([proposal](http://aprs.org/aprs12/RR-bits.txt))
- **SSID (4bits):** Set to `0` unless used for routing information **(this shouldn't be used in the New n-N Paradigm?)**

## Source address encoding
APRS follows AX.25 spec for source address encoding.
The source address is the call sign of the transmitting station with an optional SSID.
An SSID can be used to distinguish between two different stations with the same call sign.
[APRS uses this link for its SSID convention](http://aprs.org/aprs11/SSIDs.txt).
If an SSID is not used, the SSID of `0` is assumed.

The AX.25 spec encodes additional information in to the SSID byte to include additional functionality (i.e., the `C` and `RR` bits in AX.25 v2.2).
These bits are set in APRS as follows:

- **C (1 bit):** AX.25 spec shows that UI frames are [command frames](http://aprs.org/aprs11/C-bits-SSID.txt), so set to `0` for in the source address.
- **RR (2 bits):** not specified yet, set to `11` ([proposal](http://aprs.org/aprs12/RR-bits.txt))
- **SSID (4bits):** [Conventions](http://aprs.org/aprs11/SSIDs.txt)

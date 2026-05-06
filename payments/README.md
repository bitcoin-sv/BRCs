# Payments

This directory contains standards relating to protocols for exchanging payments in various contexts.

Current orientation:

- [BRC-105](./0105.md) is the main BSV-native HTTP monetization framework in this repository.
- [BRC-118](./0118.md) extends BRC-105 with multipart transport and does not deprecate header transport.
- [BRC-120](./0120.md) assigns a BRC identifier to the frozen external x402 v1.0 specification rather than restating that spec inline.
- [BRC-121](./0121.md) defines a smaller BSV-specific 402 payment profile.

HTTP `402 Payment Required` remains a reserved status code in RFC 9110; these BRCs define ecosystem conventions layered on top of that reserved code.

BRC | Standard
-----|------------------
27   | [Direct Payment Protocol (DPP)](./0027.md)
28   | [Paymail Payment Destinations](./0028.md)
29   | [Simple Authenticated BSV P2PKH Payment Protocol](./0029.md)
41   | [PacketPay HTTP Payment Mechanism](./0041.md)
54   | [Hybrid Payment Mode for DPP](./0054.md)
55   | [HTTPS Transport Mechanism for DPP](./0055.md)
70   | [Paymail BEEF Transaction](./0070.md)
105  | [HTTP Service Monetization Framework](./0105.md)
118  | [Multipart Body Transport for BRC-105 Payments](./0118.md)
120  | [x402 Stateless Settlement-Gated HTTP Protocol](./0120.md)
121  | [Simple 402 Payments](./0121.md)
125  | [PeerPay URI Scheme for BRC-29 Payments](./0125.md)

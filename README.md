# BGP-GR-Extension-RFC-8538-Design
Design for Notification Message Support for BGP Graceful Restart-RFC-8538


1. Problem Statement:

BGP GR implementation in FRR is as per RFC 4724 and limits the usage of BGP Graceful Restart to BGP messages only. 
For many classes of errors, BGP must send a NOTIFICATION message and reset the peering session to handle the error 
condition.  The BGP Graceful Restart mechanism defined in RFC 4724 requires that normal BGP procedures defined in 
RFC 4271 be followed when a NOTIFICATION message is sent or received, there by loosing all the routes and flapping 
reachability.

2. Objective:

* The primary objective is to enhance the Helper capability for BGP speaking routes. This is achieved by allowing
the BGP speaker to avoid flapping reachability and continue forwarding while the BGP speaker restarts the session
to handle errors detected in BGP.

* RFC 8538 updates  RFC 4724 by defining an extension that permits the Graceful Restart procedures to be performed 
when the BGP speaker receives a BGP NOTIFICATION message or the Hold Time expires. RFC 8538 defines a new subcode
for BGP Cease NOTIFICATION messages. This  new subcode is needed to requests a full session restart instead of a 
Graceful Restart.

* When a BGP session is reset, both speakers operate as "Receiving Speakers", and they retain each other’s routes.

* When a BGP session HOLDTIME expires, both speakers operate as "Receiving Speakers", and they retain each other’s 
routes.

* The functionality can be defeated by sending a BGP Cease NOTIFICATION message with the Hard Reset subcode. If a 
Hard Reset is used, a full session reset is performed for that neighbor.

3. Solution:

3.1. High Level Design:

* The new BGP GR capability, the "N" bit much be sent in the BGP open message from the sending router and the receiving
router must be capability of processing the new capability.  So this involves changes in both sender and receiving router.

* Both the sender and the receiver must fall back to the procedure mention in RFC 4724, when the above capability RFC 8538 
is administrative disable. This involves introducing a CLI knob to control this feature, both at the sender and receiver 
router.

* When any established BGP session under goes reset, the router must follow bgp graceful restart procedure of "Receiving Speaker", this includes both the sending and receiving router. So when a router sends or receives a notification message,  all notification except "hard reset", qualifies for "Graceful Cease" i.e. follow the procedure of  "Receiving Speaker". However whenever, any "Hard Reset" notification send or received, the router must follow normal procedure of reseting the session as mention in RFC 4271. This involves changes in both at the sender and the receiver side.

* Similar functionality changes will have to be done for HOLDTIME expiry i.e. when HOLDTIME expires, both speakers operate as "Receiving Speakers", and they retain each other’s routes.

* BGP GR Peer Down Detection Flow would under go some change in-order to implement the above points.



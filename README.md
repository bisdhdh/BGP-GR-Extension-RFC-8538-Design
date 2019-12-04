# BGP-GR-Extension-RFC-8538-Design
Design for Notification Message Support for BGP Graceful Restart-RFC-8538


1. Problem Statement:

	BGP GR implementation in FRR is as per [RFC 4724](https://tools.ietf.org/html/rfc4724) and limits the usage of BGP Graceful Restart to BGP messages only.  For many classes of errors, BGP must send a NOTIFICATION message and reset the peering session to handle the error condition.  The BGP Graceful Restart mechanism defined in RFC 4724 requires that normal BGP procedures defined in [RFC 4724](https://tools.ietf.org/html/rfc4724) be followed when a NOTIFICATION message is sent or received, there by loosing all the routes and flapping 
reachability.

2. Objective:

	* The primary objective is to enhance the Helper capability for BGP speaking routes. This is achieved by allowing
the BGP speaker to avoid flapping reachability and continue forwarding while the BGP speaker restarts the session
to handle errors detected in BGP.

	* [RFC 8538](https://tools.ietf.org/html/rfc8538) updates  [RFC 4724](https://tools.ietf.org/html/rfc4724) by defining an extension that permits the Graceful Restart procedures to be performed when the BGP speaker receives a BGP NOTIFICATION message or the Hold Time expires. [RFC 8538](https://tools.ietf.org/html/rfc8538) defines a new subcode for BGP Cease NOTIFICATION messages. This  new subcode is needed to requests a full session restart instead of a Graceful Restart.

	* When a BGP session is reset, both speakers operate as "Receiving Speakers", and they retain each other’s routes.

	* When a BGP session HOLDTIME expires, both speakers operate as "Receiving Speakers", and they retain each other’s 
routes.

	* The functionality can be defeated by sending a BGP Cease NOTIFICATION message with the Hard Reset subcode. If a 
Hard Reset is used, a full session reset is performed for that neighbor.

3. Solution:

	3.1. High Level Design:

	* The new BGP GR capability, the "N" bit much be sent in the BGP open message from the sending router and the receiving
router must be capability of processing the new capability.  So this involves changes in both sender and receiving router.

	* Both the sender and the receiver must fall back to the procedure mention in RFC 4724, when the above capability [RFC 8538](https://tools.ietf.org/html/rfc8538) is administrative disable. This involves introducing a CLI knob to control this feature, both at the sender and receiver 
router.

	* When any established BGP session under goes reset, the router must follow bgp graceful restart procedure of "Receiving Speaker", this includes both the sending and receiving router. So when a router sends or receives a notification message,  all notification except "hard reset", qualifies for "Graceful Cease" i.e. follow the procedure of  "Receiving Speaker". However whenever, any "Hard Reset" notification send or received, the router must follow normal procedure of reseting the session as mention in RFC 4271. This involves changes in both at the sender and the receiver side.

	* Similar functionality changes will have to be done for HOLDTIME expiry i.e. when HOLDTIME expires, both speakers operate as "Receiving Speakers", and they retain each other’s routes.

	* BGP GR Peer Down Detection Flow would under go some change in-order to implement the above points.

		Peer down, is primary detected by BGP IO module form TCP Session close. If GR is enabled then it lets the close reason to PEER_DOWN_NSF_CLOSE_SESSION and set PEER_STATUS_NSF_WAIT bit in  peer sflags, there by marking the start of BGP GR. In this flow it generates TCP_connection_closed or TCP_fatal_error BGP event. 

		If  event raised is TCP_connection_closed, the action function that is called is bgp_stop() or bgp_ignore(), depending on the present state.

		if event raised is TCP_fatal_error, the action function that is called is bgp_stop or bgp_connect_fail or bgp_fsm_exeption or bgp_ignore, depending on the present state. bgp_fsm_exeption() internally calls bgp_stop().  bgp_connect_fail () also calls bgp_stop() it also calls peer_delete() if it is dynamic neighbour.

		So the bottom line is, bgp_stop() would be evenbgp_clear_stale_routetually get called with PEER_STATUS_NSF_WAIT set in this flow. All the house keeping is done in this flow. However if BGP GR is enabled, it does some special processing like start the restart and the stale path timer.

		Stale marking flow :

		The routes learned by this peer is marked STALE (BGP_PATH_STALE) in bgp_clear_route_node()

		* bgp_read( ) → bgp_event( ) → bgp_event_update( TCP_fatal_error/TCP_connection_closed ) → Next-state of this peer is "Clearing" → bgp_fsm_change_status(Clearing)

		* bgp_fsm_change_status(struct peer *peer, int status) - if the status is clearing or Deleted , it would call bgp_clear_route_all( )

		* bgp_clear_route_all() → bgp_clear_route() → bgp_clear_route_table() → work_queue_add(peer->clear_node_queue, cnq)

		* bgp_clear_route_node() sets the stale mark on the route on bgp_path_info→flags for that bgp_node only if PEER_STATUS_NSF_WAIT is set and the route is not already marked as stale.

		So the bottom line is, bgp_stop() would be evenbgp_clear_stale_routetually get called with PEER_STATUS_NSF_WAIT set in this flow. All the house keeping is done in this flow. However if BGP GR is enabled, it does some special processing like start the restart and the stale path timer.

	* Define new Hard Reset Notification. This involves writing new encoder and decoder functionality of the new Hard Reset Notification.
	* As suggested in [RFC 8538](https://tools.ietf.org/html/rfc8538), hard reset notification should be sent from all the places listed below in the "Low Level Design" section.

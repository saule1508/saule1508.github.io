---
layout: post
title: Netbird scaling
published: false
---

Issuing running Netbird on a self-hosted with very large of networks and policies

I had been using NetBird for months, with a limited group of users, and it was performing very well. But when we started to migrate away from Fortinet into Netbird, as the load increased, we hit some scalability issues related to our setup.
<!--more-->

## Intro

Netbird is a very cool project that makes it easy to set-up a private network. Netbird is open source friendly and they provide a set-up script that lets you set-up netbird on a self-hosted infrastructure very quickly, so that you can play with it. Later you can decide to use the cloud version or continue using self-hosted but with support and enterprise features. Or simply continue with the free self-hosted solution.

Althought netbird was working very well with a limited group of users, once we started to migrate users from Fortinet to Netbird we hit some performance issue, namely the cpu usage on the mgmt server went out of control. Thanks to a golang CPU profile, it appeared that the CPU was burned by the OS version and NB version posture checks, so that disabling those posture checks and at the same time increasing the number of CPU gave me time to get more deeply in the issue without stopping the migration.

## The problem

### First issue: login expiration job kicking out users when they are loggin into.

With 70 concurrent users connected to Netbird, A lot of users were experiencing difficulties to log in to Netbird, a general pattern reported by users was

* login to Netbird -> SSO login -> Success
* immediately after: message received "Your login has expired"
* login again to Netbird -> SSO login -> Success
* immediately after: "Your login has expired"

this was always during peak hour, it could happen one, twice or three times in a row. At first I shrugged off the issue, because it never happened to me (because I was outside peak hour, being in a different timezone than most of the users), but at the end it happened once to me and - after much debugging and code inspection - I could link the issue to the expiration job which was incredibly aggressive and contained a flaw. The job fetch all the peers from the database, more than 1000, iterate on them to check the last_login and if older than the setting expires them. The flaw here is that it does that even for peer that are already expired, so on a buzy system, the grpc login request is racing agains the expiry job. Disabling the expiration job (and running it manually outside business hours) made a big difference, after that I improved the code and will make a github issue

### Second issue: load increasing exponentially with peers

Another concerning issue was the CPU load increasing, which was correlated with a high metric for the login and the UpadeAccountPeers function. The UpdateAccountPeer is a function triggered by concurrent grpc requests (peer login IN, peer login Out, expiry, API calls,..) which recalculates the network map of all peers that are connected, so that Netbird will send the updated network map via the grpc channel specific to each user. 

UpdateAccountPeer is protected against concurrent execution but the throttling is not very well implemented and the function can be fired in very rapid succession.

![netbird performance issue]({{ site.url }}/images/nbperf_sep01.png)

The CPU profile showed that all the CPU was burned by the OS version check and the NB version check, so an immediate relief in order to not stop the migration was to disable those two checks which we did not really need.

![netbird CPU profile]({{ site.url }}/images/nbperf_cpuprofile_sep.png)

After having changed that and continuing the migration, we reached 400 concurrent peers and the load started again to become a worry. At that time I kept taking profiles, and all of them were pointing to the Network Map Calculation, the function GetPeerConnectionResources and more precisely the GetAllPeersFromGroup which happened to be very expensive in our set-up.

UpdateAccountPeers -> iterate over all connected peers (10 by 10). For each peer, it calls GetPeerNetworkMap which is basically the info that the control plane sends to each peer in order for netbird on the client peer to configure the wireguard interface and the firewall rules.

In GetPeerNetworkMap the bulk of the work is the GetPeerConnectionResources [GetPeerConnectionResources](
https://github.com/netbirdio/netbird/blob/96f71ff1e15b50eff87a7052432537dad2718b20/management/server/types/account.go#L274) which returns the list of peers and the firewall rules relevant to that peer.

But in our Netbird set-up, we have thousands of peers (our users) that connects to routing peers and only to routing peers, we are basically replacing Fortinet. So that in this set-up almost all our peers don't need to receive the list of peers to connect (other than for the routing peers and dns, ..) and so this calculation can be completly skipped !!

And secondly, if the GetPeerConnectionResources is so expensive for us, it is because we have a staggering amount of network and groups (one group per network) and policies (one policy per network), and this number of policies is the key to the performance issue we have.

The high number of groups is due to the design of the internal cloud in the company, we have "tenants" which are kind of business units, and each tenant is isolated from each other. The seggregation at network level is a key principle. And on top of that for each tenant there are 3 plateforms (production, preproduction, development), so each combination tenant/plateforme is a group. Idea is to segregate completly tenants and plateforms, a user's peer can have access to only to his tenant and not necessarily all plateform (sometimes only dev, sometimes prod, ..) (edited) 

Our ACL model fitted nicely with the one of Netbird and this was the main reason why Netbird was such a good match, it was very easy to map that model to the ACL of netbird (it is almost a one to one match with netbird). But of course I did not predict that kind of performance issue...

So looking at what the GetPeerConnectionResources is doing:

- Loop on each policy, this is the huge cost for use because there are 480
- For each policy rule, get the source groups and the posture checks attached to the policy, and then iterate on each peer in the source to execute the posture checks. This returns a boolean indicating if the peer (the one we are computing the network map) is in the source groups or not. If it is not in the source groups and also not in the destination, then we have computed all the posture checks for nothing because the policy is not relevant to this peer. Note that in this evaluation of the posture checks the expired peers (which are not connected) are not excluded. That's why I could see in the logs the same IOS peer repeatedly logging an error about an unsupported OS !! But this IOS peer did not connect to the network since 2 months.

This evaluation of the posture checks, even though the only one left was the process check (which is very efficient), was a huge hotspot for us. Especially the logging inside this hotstop, even when running in Info mode the Debug log were causing huge cpu cost.

To solve this issue with a very low risk change (because functionally nothing is changed), I first evaluate if the rule is relevant to the peer or not, by looking if this peer is in any source or destination group. If not, and if it is not a routing peer, this rule is not relevant for the peer and all the posture checks are not needed

```
// GetPeerConnectionResources for a given peer
//
// This function returns the list of peers and firewall rules that are applicable to a given peer.
func (a *Account) GetPeerConnectionResources(ctx context.Context, peer *nbpeer.Peer, validatedPeersMap map[string]struct{}) ([]*nbpeer.Peer, []*FirewallRule) {
	generateResources, getAccumulatedResources := a.connResourcesGenerator(ctx, peer)

	for _, policy := range a.Policies {
		if !policy.Enabled {
			continue
		}

		for _, rule := range policy.Rules {
			if !rule.Enabled {
				continue
			}

			var sourcePeers, destinationPeers []*nbpeer.Peer
			var peerInSources, peerInDestinations bool
			if earlyCheckPeerConnectionResources {
				// this is an optimization for setup with a large number of rules
				// we do an early check to see if the peer is in sources or destinations, if not the rule can be skipped
				// before doing the more expensive getAllPeersFromGroups calls (which involve posture checks etc)
				var earlyCheckIsPeerInSources bool
				var earlyCheckIsPeerInDestinations bool
				if rule.SourceResource.Type == ResourceTypePeer && rule.SourceResource.ID != "" {
					_, earlyCheckIsPeerInSources = a.getPeerFromResource(rule.SourceResource, peer.ID)
				} else {
					earlyCheckIsPeerInSources = a.getPeerMembershipFast(rule.Sources, peer)
				}

				if rule.DestinationResource.Type == ResourceTypePeer && rule.DestinationResource.ID != "" {
					_, earlyCheckIsPeerInDestinations = a.getPeerFromResource(rule.DestinationResource, peer.ID)
				} else {
					earlyCheckIsPeerInDestinations = a.getPeerMembershipFast(rule.Destinations, peer)
				}
				if !earlyCheckIsPeerInSources && !earlyCheckIsPeerInDestinations {
					// early check says we can skip this rule
					continue
				}
			}
			// this is the original code
			if rule.SourceResource.Type == ResourceTypePeer && rule.SourceResource.ID != "" {
				sourcePeers, peerInSources = a.getPeerFromResource(rule.SourceResource, peer.ID)
			} else {
				sourcePeers, peerInSources = a.getAllPeersFromGroups(ctx, rule.Sources, peer.ID, policy.SourcePostureChecks, validatedPeersMap)
			}

			if rule.DestinationResource.Type == ResourceTypePeer && rule.DestinationResource.ID != "" {
				destinationPeers, peerInDestinations = a.getPeerFromResource(rule.DestinationResource, peer.ID)
			} else {
				destinationPeers, peerInDestinations = a.getAllPeersFromGroups(ctx, rule.Destinations, peer.ID, nil, validatedPeersMap)
			}
			if rule.Bidirectional {
				if peerInSources {
					generateResources(rule, destinationPeers, FirewallRuleDirectionIN)
				}
				if peerInDestinations {
					generateResources(rule, sourcePeers, FirewallRuleDirectionOUT)
				}
			}

			if peerInSources {
				generateResources(rule, destinationPeers, FirewallRuleDirectionOUT)
			}

			if peerInDestinations {
				generateResources(rule, sourcePeers, FirewallRuleDirectionIN)
			}
		}
	}
	return getAccumulatedResources()
}

```

I believe this code optimization, which is behing a feature flag, could be incorporated in the upstream project, because it will make sense of all set-up where the number of policies is big.

Another massive optimization, but this one is probably very specific to the use case of using netbird with peer to routing pers connection only (as opposed to user peers to user peers). The optimization here is to decide upfront, before generating the ACL peers list and firewall rule, if it is needed. Because if the peer is not in any destination and not in any sources (for the sources I am not sure, I believe this condition is even not necessary), then the acl Peers list and the firewall rules are always empty

```
// In the PeerNetworkMap generation process, we need to get the aclPeers and firewall rules
// however, if the peer for which the network map is being generated is not involved in any
// relevant policy_rules, we can skip the calculation. This saves a lot of resources.
// The rule is:
//   - if it is a router, we cannot skip the calculation
//   - check all the policy rules where the destination is a peer group (as opposed to resources/subnets). For those rules, check if the peer is member of a source group
//     or a member of the destination groups. If so, we need to run the calculation.
//     If not the peer is not involved in any peer-to-peer rule and the aclPeers and fwRules are empty.
//
// This check is especially relevant for large setup with lot of rules but without peer-to-peer connections, a set-up based mostly on routing peers.
func (a *Account) shouldRunConnectionResources(peer *nbpeer.Peer, isRouter bool) bool {
	if isRouter {
		return true
	}

	// 2. Iterate through all Policies / Rules, we can discard rules where destination is a group of resources.
	for _, policy := range a.Policies {
		if !policy.Enabled {
			continue
		}

		for _, rule := range policy.Rules {
			if !rule.Enabled {
				continue
			}
			// --- STEP A: Filter by Destination Type (Big Time Saver for models based on routing peers) ---
			isDestinationPeerGroup := false

			// Check if destination is a single peer resource (peer-to-peer)
			if rule.DestinationResource.Type == ResourceTypePeer && rule.DestinationResource.ID != "" {
				isDestinationPeerGroup = true
			} else {
				// Check if any destination group is a Peer Group
				// We do that by looking each group in the Destinations. If the group as resources, it is not a group of peers.
				// Note: this causes an issue if there is a policy with no resources and no peers (invalid configuration)
				for _, destGroupID := range rule.Destinations {
					group := a.GetGroup(destGroupID)
					if group != nil && len(group.Resources) == 0 {
						isDestinationPeerGroup = true
						break
					}
				}
			}

			// If the destination is NOT a peer group, this rule only involves resources/subnets. Skip it.
			if !isDestinationPeerGroup {
				continue
			}

			// If we reach here, the rule's destination is a peer group. Now check if the current peer is a source.

			// Check if source is a single peer resource (peer-to-peer)
			if rule.SourceResource.Type == ResourceTypePeer && rule.SourceResource.ID == peer.ID {
				return true // Found a relevant peer-to-peer rule, cannot skip GetPeerConnectionResources
			}

			// Check if peer is a member of the source groups
			if a.getPeerMembershipFast(rule.Sources, peer) {
				return true // Found a relevant peer-to-peer rule, might not be able to skip (TODO: validate this assumption)
			}
			// Check if peer is a member of the destination groups
			if a.getPeerMembershipFast(rule.Destinations, peer) {
				return true // Found a relevant peer-to-peer rule, cannot skip
			}
		}
	}
	// If the loop completes, the peer is not the source of any rule
	// whose destination is another peer group.
	return false // Safe to skip GetPeerConnectionResources
}

// lookup the routers map to see if the peer is a router
func isPeerRouter(peer *nbpeer.Peer, routers map[string]map[string]*routerTypes.NetworkRouter) bool {
	for _, router := range routers {
		if _, ok := router[peer.ID]; ok {
			return true
		}
	}
	return false
}
```

## Conclusion

When hitting performance issue with Netbird the go CPU profiler is very usefull. I did a lot of iteration, each time taking a new profile. What I learned

* You cannot focus on the CPU usage only, because some optimizations caused the CPU usage to increase ! Just because I was removing contention and so the program was doing more work. It is better to focus on the load, the 1min, 5min and 15min. As long as the 5min is below the number of cores the situation is ok.
* During the debugging part, I added log statement. But a log statement in the hot path has a huge impact on CPU. With our setup with 480 rules, I was having rates of log of thousands per seconds. Even when the log level is above the log line, it has a very significant impact on the CPU. So the trick is to guard the logs in the hot path, like this

```
				if log.IsLevelEnabled(log.DebugLevel) {
					log.WithContext(ctx).Debugf("an error occurred check %s: on peer: %s :%s", check.Name(), peer.ID, err.Error())
				}
```
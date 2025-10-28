---
layout: post
title: Netbird scaling
published: false
---

Running Netbird on self-hosted with very large mumber of groups, networks and policies

I had been using NetBird for months, with a limited group of users, and it was performing very well. But when we started to migrate away from Fortinet into Netbird, as the load increased, we hit some scalability issues related to our setup which is very large and probably very atypical compared to normal netbird user.
<!--more-->

## Intro

Netbird is a very cool project that makes it easy to set-up a private network. Netbird is open source friendly and they provide a set-up script that lets you set-up netbird on a self-hosted infrastructure very quickly, so that you can play with it. Later you can decide to use the cloud version or continue using self-hosted but with support and enterprise features. Or simply continue with the free self-hosted solution.

Althought netbird was working very well with a limited group of users, once we started to migrate users from Fortinet to Netbird we hit some performance issues, which appeared to be link with the very large number of networks and groups we are managing.

## The problem

We are using Netbird to replace Fortinet. Fortinet was plagued with a constant stream of vulnerability. The director of infra, and the network guys, had all heard about wireguard but nobody had the time to look into it, so I seized this opportunity to do something cool with my team.

The company is operating an internal cloud, fully automated, in which each "customer" (internal customer) is fully segreggated from the others. We call those "customers" tenant, we can think of it as a business-unit in the company. Furthermore each tenant has access to 3 platforms, totally segregated as well : production, pre-production and development. So each tenant is fully isolated from the others and is offered 3 plateforms also seggregated, this is a core principle of the internal cloud. A user can have access to only to his tenant (sometimes multiple tenants) and not necessarily all plateform (sometimes only dev, sometimes prod, ..)

With Fortinet each user had access to the subnets of his tenant(s), on the 3 platforms. So we wanted to do the same with wireguard and it turned out netbird was a perfect fit for that. We netbird we can:

* group the resources of a tenant in a network
* create a corresponding group of peers that have access to those resources
* create a policy to give access to those resources to member of this group
* and because we wanted to restrict to production to "certified" device, we also added a posture check for each production subnets. This was not possible with Fortinet and is a significant improvement of the new solution.

But as a result we end-up with a staggering amount of groups, networks and policies, for which the implementation of part of the code in netbird was not adapted. We have namely 160 tenants (and it is increasing as more and more of the internal customers want to move away from public cloud), which means 480 networks (one per platform) and 480 groups and policies. 

This was challenging for netbird, which was probably not designed at the begining for that kind of very large set-up (at the begining of the project, the database was just a json file !)


### First issue: login expiration job kicking out users when they are loggin into.

Early in the migration, when we had 70 concurrent peers using the VPN, some users were experiencing difficulties to log in to Netbird, a general pattern reported was

* login to Netbird -> SSO login -> Success
* immediately after: message received "Your login has expired"
* login again to Netbird -> SSO login -> Success
* immediately after: "Your login has expired"

this was always during peak hour, it could happen one, twice or three times in a row. At first I shrugged off the issue, because it never happened to me (because I was outside peak hour, being in a different timezone than most of the users), but at the end it happened once to me and - after much debugging and code inspection - I could link the issue to the expiration job which is incredibly aggressive in the frequency. The job expires all the peers from the database (based on the setting) and do that repeatedly, even for peers that are already expired. In my case this expiration job was racing with other, unrelated, grpc login request. Disabling the expiration job (and running it manually outside business hours) made a big difference, after that I improved the code in my fork but will make a github issue. I think the best would be to have this job scheduled with a low frequency (to batch expiration) by a central schedule and keep it out of the requests.


### Second issue: load increasing exponentially with peers

A worrying issue was the CPU load increasing, which was correlated with a high metric for the login and the UpadeAccountPeers function. 

The UpdateAccountPeer is a function triggered by concurrent grpc requests (peer login IN, peer login Out, expiry, API calls,..) which recalculates the network map of all peers that are connected, so that Netbird will send the updated network map via the grpc channel specific to each user. The NetBird control plane is basically doing that continuously, either because a peer joins the network, or leaves it, .. because all other peers might need the information. 

UpdateAccountPeer is protected against concurrent execution. There is also a throttling mechanism, but it does not prevent executions in rapid succession, when the system is very buzy. I think the buffering mechanism could be improved here.

![netbird performance issue]({{ site.url }}/images/nbperf_sep01.png)

The CPU profile showed that all the CPU was burned by the OS version check and the NB version check, so an immediate relief in order to not stop the migration was to disable those two checks. It made a huge difference.

![netbird CPU profile]({{ site.url }}/images/nbperf_cpuprofile_sep.png)

After having changed that and continuing the migration, we reached 400 concurrent peers and the load started again to become a worry. At that time I kept taking profiles, and all of them were pointing to the UpdateAccountPeers, specifically in the Peer Network Map Calculation, the function to get - for each peer - the list of peers it is connected to. This function is called GetPeerConnectionResources, it was killing us because of our large number of policies.

UpdateAccountPeers -> iterate over all connected peers (10 by 10). For each peer, it calls GetPeerNetworkMap which is basically the info that the control plane sends to each peer in order for netbird on the client peer to configure the wireguard interface and the firewall rules.

In GetPeerNetworkMap the bulk of the work is the GetPeerConnectionResources [GetPeerConnectionResources](
https://github.com/netbirdio/netbird/blob/96f71ff1e15b50eff87a7052432537dad2718b20/management/server/types/account.go#L274) which returns the list of peers and the firewall rules relevant to that peer.

But in our Netbird set-up, we have thousands of peers (our users) that connects to routing peers and only to routing peers (we are basically replacing Fortinet, to give access to our users to our private cloud). So that in this set-up almost all our peers don't need to receive the list of peers to connect (other than for the routing peers and dns, ..) and so this calculation can be completly skipped !!

And secondly, if the GetPeerConnectionResources is so expensive for us, it is because by iterating on all policies, and then on all peers in the groups related to this policy, it is doing over and over the same posture checks.

So looking at what the GetPeerConnectionResources is doing:

- Loop on each policy, this is the huge cost for use because there are 480
- For each policy rule, get the source groups and the posture checks attached to the policy, and then iterate on each peer in the source to execute the posture checks. This returns a boolean indicating if the peer (the one we are computing the network map) is in the source groups or not. If it is not in the source groups and also not in the destination, then we have computed all the posture checks for nothing because the policy is not relevant to this peer. Note that in this evaluation of the posture checks the expired peers (which are not connected) are not excluded. That's why I could see in the logs the same IOS peer repeatedly logging an error about an unsupported OS !! But this IOS peer did not connect to the network since 2 months.

This evaluation of the posture checks, even though the only one left was the process check (which is very efficient), was a huge hotspot for us. Especially the logging inside this hotspot, even when running in Info mode the Debug log were causing huge cpu cost.

The explosion comes from:
- update account peers
- iterate on all connected peers, 10 by 10. Let say 500 connected peers
- each iterate on all policy rules (480)
- each policy rule iterates on all peers related to the policy (even the expired) and - for productio network - execute the posture checks.

To solve this issue with a very low risk change (because functionally nothing is changed), I first evaluate if the rule is relevant to the peer or not. If not I don't need to iterate on the peers inside the rules and to evaluate the posture checks. In the code below, the boolean peerInSources and peerInDestination - which are cheap to compute - can avoid in most of the case the heavy computation with posture checks.

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

I believe this code optimization, which is behing a feature flag, could be incorporated in the upstream project, because it will make sense of all set-ups where the number of policies is big.

Another massive optimization, but this one is probably very specific to the use case of using netbird with peer to routing pers connection only (as opposed to user peers to user peers), is to decide upfront, before generating the ACL peers list and firewall rule, if it is at all needed. Because if the peer is not in any destination and not in any sources, then the acl Peers list and the firewall rules are always empty and we don't even need to call the function.

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

Related to the posture check, my first attempt to solve the issue was to maintain a global cache of peerID+checkID with a small TTL. But then the next cpu profile showed that the bottlenect had become the mutex to read and update the cache. So it showed that more than the posture check it was the insane amount of time it is called that was an issue.




## Conclusion

When hitting performance issue with Netbird the go CPU profiler is very usefull. I did a lot of iteration, each time taking a new profile. What I learned

* You cannot focus on the CPU usage only, because some optimizations caused the CPU usage to increase ! Just because I was removing contention and so the program was doing more work. It is better to focus on the load, the 1min, 5min and 15min. As long as the 5min is below the number of cores the situation is ok.
* During the debugging part, I added log statement. But a log statement in the hot path has a huge impact on CPU. With our setup with 480 rules, I was having rates of log of thousands per seconds. Even when the log level is above the log line, it has a very significant impact on the CPU. So the trick is to guard the logs in the hot path, like this

```
				if log.IsLevelEnabled(log.DebugLevel) {
					log.WithContext(ctx).Debugf("an error occurred check %s: on peer: %s :%s", check.Name(), peer.ID, err.Error())
				}
```
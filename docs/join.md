# Joining the forum & discovering the peers

This is the well-known bootstrap problem in the p2p networks. Currently IPFS doesn't offer any means to build sub-networks (aka overlay networks), so we need to come up with our own way to do so.

## ipfs pubsub

IPFS has the pubsub API, which is in beta. It's hardly usable now, but in theory it will allow a decentralised way to build overlay networks (ignoring the fact that IPFS itself has bootstrap nodes). So here is the idea, which doesn't currently work.

Say there is already a million users in the forum. Now we download the app, which contains the forum id, and want to find a few other users. We can use [ipfs pubsub](https://docs.ipfs.io/reference/api/cli/#ipfs-pubsub) for that. However if we simply use `ipfs pubsub pub $FID "Hello everyone!"` to let online users reply, we may get a million replies, while we need only a few tens. So we use a more sophisticated discovery protocol:

- `ipfs pubsub pub $FID 1` - only those users reply whose [ipfs id](https://docs.ipfs.io/reference/api/cli/#ipfs-id) differs from ours by 1 bit.
- `ipfs pubsub pub $FID 2` - now only those reply who have 2 different bits in their id.
- `ipfs pubsub pub $FID 3`

...and so on. `ipfs id` values are evenly distributed, so if the length of the id is 256 bits and there are 1 million users online, the number of replies to `ipfs pubsub pub $FID 96` will be `10^6 * C(96,256)/2^256` = [16](http://m.wolframalpha.com/input/?i=10%5E6+*+256%21%2F%282%5E256+*+96%21+*+%28256+-+96%29%21%29), meaning that there are 16 users online whose `ipfs id` has 96 different bits.

This is a somewhat heavy procedure and is only necessary when we need to start from nothing. However once we have a list of other forum users, we can use them to discover more users.

## ipfs dht

IPFS exposes the Kademlia DHT API, but in a limited fashion: it doesn't allow to get/set arbitrary keys, only the IPNS ones. It's likely possible to use DHT to build overlay networks.

## The bootstrap nodes

We can use the ideas from torrent magnet links:
- The forum has an immutable id.
- There is a list of nodes who bootstrap the forum when the number of users is low.

When someone creates a forum, it generates a magnet link which includes the hostname/address of the only node that provides the content at the moment:

```
magnet:?xt=urn:dfid:b54a3a64a858&dn=The+Cats+Forum&tr=cats2511.foobar.contoso.com
```

Then this link is shared with others somehow and those others sync the `b54a3a64a858` forum with the `cats2511` node. Then new users go to `cats2511`, ask for the list of peers, go to those peers and so on. Eventually we'll get an overlay p2p network.

What if `cats2511` goes offline? Existing users won't be affected: they keep the list of many peers and `cats2511` is only one of them. However new users won't be able to use the original magnet link, so they'll need to get address of an online user. This is the same problem when a torrent tracker goes offline: users need to find another tracker. The difference is that when a torrent tracker goes offline, it takes offline all its content - the posts, comments and so on, while when `cats2511` goes offline, the same content can be synced from another node. The idea is very similar to [PeerTube](https://github.com/Chocobozzz/PeerTube).



# What is this?
Think about a shared folder on an FTP server where many users can add files. In this folder there is also a special `rules` script that enforces certain rules about what files can be added, who can add them and so on. This shared folder has some disadvantages: there is an admin who is above the rules and all the files are stored in one place. The idea of this project is to use [IPFS](https://ipfs.io) to implement this shared folder pattern without the disadvantages: there won't be an admin and there won't be a single place where the files are stored.

Why not to use IPFS directories? They are immutable. An IPFS directory is just a immutable IPFS file that has links to other IPFS files and directories. An IPFS directory can be thought of as an immutable JSON object where leafs are IPFS files.

What about [ipfs files](https://docs.ipfs.io/reference/api/cli/#ipfs-files)? That's just a more convenient API to create immutable IPFS directories.

# Contents
- [How this is supposed to be used](docs/how-to-use.md)
- [Internal structure](docs/internal-structure.md)
- [The rules script](docs/rules-script.md)
- Typical examples of such shared folders:
  - [Post-moderated forums](docs/examples/post-moderation.md)
  - [Invite-only forums](docs/examples/invite-only.md)

# How this works

Each user runs an IPFS node. When we want to post a new cat, we do the following:

- Run the `rules` script to check that the new cat is ok and meets all the requirements of the forum.
- Use [ipfs add](https://docs.ipfs.io/reference/api/cli/#ipfs-add) to create an IPFS file with the cat's description. IPFS gives us a hash or [CID](https://docs.ipfs.io/guides/concepts/cid) of this file.
- Add the `CID` to the local copy of the shared folder: `/$FID/$CID` where `FID` is the forum id.
- Use [ipfs p2p](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#ipfs-p2p) to send the `CID` of the file to a few other users of the forum.
- Each user who receives this message also runs `rules` and if the `CID` is ok, the `CID` is added to the user's copy of the shared folder and gets re-transimtted to the user's peers. The `rules` script likely needs access to the contents of the file: it gets them with [ipfs get](https://docs.ipfs.io/reference/api/cli/#ipfs-get).

Now what happens when someone wants to spam in the forum. The spammer may skip the `rules` check and just send new files to other users. However other users will run the `rules` check, find out that the files are bad and the spammer will be blocked by those users.

## Joining the forum

See [docs/join.md](docs/join.md).

## Syncing the list of files

We have been offline for a day and now want to see what cats have been added since last time we were online. We pick a random online user in our list of peers and want to sync the list of cats. If the list is small, we could just send the lists of file `CID`s, but this doesn't scale. Here is a more sophisticated, but still fairly simple, sync protocol.

We need to run `sync(x, y)` where `x` and `y` represent two sets of file hashes:

```
x = [ 06b645, 00f4a0, 00e0ad, 141599, 1d8b4e, 1a2287, 101114,         c8d1b0 ]
y = [ 06b645, 00f4a0,         141599, 1d8b4e, 1a2287, 101114, c78f11, c8d1b0 ]
```

We can split the entire set into subsets based on the 1st hash digit:

![](diag/x-set-tree/g.png)

Then we compare the subset hashes and continue this process recursively:

![](diag/sync-seq/g.png)

In this diagram:

1. `X` and `Y` exchange with hashes of the entire sets. They see that the two hashes are different, so some elements need to be synced.
1. `X` splits `x` into subsets based on the 1st hex digit of subset hash and sends the 3 hashes to `Y`:
    - `902787 = H(x:0) = H(06b645,00f4a0,00e0ad)`
    - `882729 = H(x:1) = H(141599,1d8b4e,1a2287,101114)`
    - `877000 = H(x:c) = H(c8d1b0)`
1. `Y` does the same with `y`.
1. `X` and `Y` see that `x:1` and `y:1` have the same hash `882729`, so there is no need to sync them further.
1. `X` needs to continue this process recursively with `x:0` and split it into subsets with hash prefixes `00`, `01`, `02` and so on. However it sees that `x:0` has only 3 items, so it just sends the 3 hashes to `Y`. Same with the `x:c` subset.
1. `Y` does the same with `y:0` and `y:c`.
1. Now `X` knows that it was missing file `c78f11` and `Y` knows that it was missing file `00e0ad`.

Note, that `sync(x, y)` doesn't have to sync the entire sets of files the two nodes have. Instead, `X` may choose to sync only a part of it, e.g. all file added the last week and as long as `Y` can apply the same filter to `y`, they can sync the two subsets:

```
sync(
  x.filter(f => f.time > Date.now() - 10 * day),
  y.filter(f => f.time > Date.now() - 10 * day));
```

# Implementation details

First we need to know where this app needs to run:

- Android and iOS phones.
- Windows and MacOS desktops.
- Websites? Someone who doesn't have an app, could go to one of known gateways, enter `FID` and join. JS can store a very limited amount of data locally: a few MBs at most. P2P connectivity is also not easy in client side JS, even with WebRTC DataChannel.
- Linux servers? This type of forum doesn't need servers that would be always online, however deploying a few servers in key geo locations can help a lot with connectivity and latency. Such servers will be like regular clients, except that they will have much higher storage limits and network bandwidth. It's interesting, that it's possible to arrange a forum where such servers would pay for themselves: every post in the forum would need to have a transaction to a bitcoin wallet associated with the servers and the `rules` script will check validity of the attached transaction id.

Writing code is hard, so the less code the better. We can implement all this logic as a [react native](https://github.com/facebook/react-native) app:

- The IPFS core will go to a RN native module, so `require('ipfs')` will pull
  - [go-ipfs](https://github.com/ipfs/go-ipfs) on desktops and servers.
  - [java-ipfs-api](https://github.com/ipfs/java-ipfs-api) on Android
- The additional logic of discovering forum users and syncing the list of files will go to another RN module.
- The app UI will be written in JS React and present the files in the form of a forum with comments and so on.

![](diag/react-native/g.png)

Possible interface of the `shared-folder` module:

```ts
interface SharedFolder {
  join(id: string): Promise<void>; // finds other peers
  sync(): Promise<void>; // syncs the list of files
  add(path: string, data: string): Promise<void>;
  get(path: string): Promise<string>;
  ls(): Iterable<string>;
}
```

# CLI

We can think of a command line interface built on top of the `shared-folder` module:

```
$ shdir init
$ shdir add RULES /tmp/rules
$ shdir add README.md /tmp/foo.txt
$ shdir publish
776b00
```

Then other peers can join the forum:

```
$ shdir init
$ shdir join 776b00
$ shdir add cats/27718/README.md /tmp/cat.md
$ shdir sync
```

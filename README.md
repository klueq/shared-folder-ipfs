# WTF is this?
Think about a shared folder on an FTP server where many users can add files. In this folder there is also a special `rules` script that enforces certain rules about what files can be added, who can add them and so on. This shared folder has some disadvantages: there is an admin who is above the rules and all the files are stored in one place. The idea of this project is to use [IPFS](https://ipfs.io) to implement this shared folder pattern without the disadvantages: there won't be an admin and there won't be a single place where the files are stored.

Why not to use IPFS directories? They are immutable. An IPFS directory is just a immutable IPFS file that has links to other IPFS files and directories.

What about [ipfs files](https://docs.ipfs.io/reference/api/cli/#ipfs-files)? TODO

# A typical use case

Say we want to start a reddit-like forum about cats:

- We `git clone` this project.
- Give it a unique id that will serve as the id of the shared folder.
- Write the `rules` script that will validate each cat before it's posted.
- Compile everything into an Android or iOS app.
- Publish the app and let users download it.
- Each app instance uses the folder id to find other users of the forum.

The `rules` script can be arbitrarily complex:

- It may just check that a new post contains a description of length 200-500 words.
- It may check that the description has a valid URL to a cat picture.
- It may actually download that URL, check the size of the downloaded picture and its format.
- It may even run an ML model to verify that the picture contains a cat and not something else.
- It may contact a captcha service and sign the post with a RSA key if the user answered the captcha correctly.

# How this works

Each user runs an IPFS node. When we want to post a new cat, we do the following:

- Run the `rules` script to check that the new cat is ok and meets all the requirements of the forum.
- Use [ipfs add](https://docs.ipfs.io/reference/api/cli/#ipfs-add) to create an IPFS file with the cat's description. IPFS gives us a hash or [CID](https://docs.ipfs.io/guides/concepts/cid) of this file.
- Add the `CID` to the local copy of the shared folder: `/$FID/$CID` where `FID` is the forum id.
- Use [ipfs p2p](https://github.com/ipfs/go-ipfs/blob/master/docs/experimental-features.md#ipfs-p2p) to send the `CID` of the file to a few other users of the forum.
- Each user who receives this message also runs `rules` and if the `CID` is ok, the `CID` is added to the user's copy of the shared folder and gets re-transimtted to the user's peers. The `rules` script likely needs access to the contents of the file: it gets them with [ipfs get](https://docs.ipfs.io/reference/api/cli/#ipfs-get).

Now what happens when someone wants to spam in the forum. The spammer may skip the `rules` check and just send new files to other users. However other users will run the `rules` check, find out that the files are bad and the spammer will be blocked by those users.

# Joining the forum

Say there is already a million users in the forum. Now we download the app, which contains the forum id, and want to find a few other users. We can use [ipfs pubsub](https://docs.ipfs.io/reference/api/cli/#ipfs-pubsub) for that. However if we simply use `ipfs pubsub pub $FID "Hello everyone!"` to let online users reply, we may get a million replies, while we need only a few tens. So we use a more sophisticated discovery protocol:

- `ipfs pubsub pub $FID 1` - only those users reply whose [ipfs id](https://docs.ipfs.io/reference/api/cli/#ipfs-id) differs from ours by 1 bit.
- `ipfs pubsub pub $FID 2` - now only those reply who have 2 different bits in their id.
- `ipfs pubsub pub $FID 3`

...and so on. `ipfs id` values are evenly distributed, so if the length of the id is 256 bits and there are 1 million users online, the number of replies to `ipfs pubsub pub $FID 96` will be `10^6 * C(96,256)/2^256` = [16](http://m.wolframalpha.com/input/?i=10%5E6+*+256%21%2F%282%5E256+*+96%21+*+%28256+-+96%29%21%29), meaning that there are 16 users online whose `ipfs id` has 96 different bits.

This is a somewhat heavy procedure and is only necessary when we need to start from nothing. However once we have a list of other forum users, we can use them to discover more users.

# Syncing the state

We have been offline for a day and now want to see what cats have been added since last time we were online. We pick a random online user in our list of peers and want to sync the list of cats. If the list is small, we could just send the lists of file `CID`s, but this doesn't scale. Here is a more sophisticated, but still fairly simple, sync protocol:

- We split our list of cats into two groups: `CID`s that start with `0` and `CID`s that start with `1`. The peer does the same thing.
- We compute hashes of the two groups. So does the peer.
- We exchange with the two hashes. Now we know which of the two groups are different. Maybe both.
- We continue subdividing the groups that differ.
- Once the groups that differ are small enough, we just send the list of `CID`s.

Instead of splitting each group into 2, we can split it into 4 or 8 subgroups. The optimal parameters depend on latency and network speed.

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

![](https://g.gravizo.com/svg?digraph%20G%20{;%20%20node%20[shape=box]%20%20;%20%20ui%20-%3E%20shf%20-%3E%20ipfs%20-%3E%20{go%20java};%20%20shf%20-%3E%20fs;%20%20fs%20[label=%22Native%20File%20System%22];%20%20ipfs%20[label=%22require(%27ipfs%27)%22];%20%20shf%20[label=%22require(%27shared-folder%27)%22];%20%20ui%20[label=%22React%20JS%20UI%22];%20%20subgraph%20cluster_native%20{%20%20%20%20;%20%20%20%20go%20java;%20%20%20%20graph%20[style=dashed%20color=gray];%20%20}%20%20;})

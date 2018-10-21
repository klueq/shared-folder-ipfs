# What is this?
Think about a shared folder on an FTP server where many users can add files. In this folder there is also a special `rules` script that enforces certain rules about what files can be added, who can add them and so on. This shared folder has some disadvantages: there is an admin who is above the rules and all the files are stored in one place. The idea of this project is to use [IPFS](https://ipfs.io) to implement this shared folder pattern without the disadvantages: there won't be an admin and there won't be a single place where the files are stored.

Why not to use IPFS directories? They are immutable. An IPFS directory is just a immutable IPFS file that has links to other IPFS files and directories. An IPFS directory can be thought of as an immutable JSON object where leafs are IPFS files.

What about [ipfs files](https://docs.ipfs.io/reference/api/cli/#ipfs-files)? That's just a more convenient API to create immutable IPFS directories.

# How this is supposed to be used

Say we want to start a reddit-like forum about cats:

- We `git clone` this project.
- Give it a unique id that will serve as the id of the shared folder.
- Write the `rules` script that will validate each cat before it's posted.
- Compile everything into an Android or iOS app.
- Publish the app and let users download it.
- Each app instance uses the folder id to find other users of the forum.

![](diag/forum-subnet/g.png)

In this diagram:

- All the nodes are IPFS nodes.
- Square green nodes participate in the same forum.
- Solid lines are IPFS connections between nodes.
- Dotted lines are additional IPFS connections discovered by the forum nodes.

This means that when `a1` wants to add a new file in the forum, it sends its hash to `c1`, `c3` and `b4` and they later send it to `d2`.

The `rules` script can be arbitrarily complex:

- It may just check that a new post contains a description of length 200-500 words.
- It may check that the description has a valid URL to a cat picture.
- It may actually download that URL, check the size of the downloaded picture and its format.
- It may even run an ML model to verify that the picture contains a cat and not something else.
- It may contact a captcha service and sign the post with a RSA key if the user answered the captcha correctly.

## Internal structure

What a shared folder actually is? It consists of two things:

- A network of nodes who participate in this shared folder. This is a subset of the IPFS nodes.
- Each node has a local copy of the shared folder. This copy is partial.

For simplicity of design (and to make it possible at all), we only allow to add files: the protocol won't allow to remove files or edit them. Thus the complete up to date copy of the shared folder is the union of all the local copies. This complete up to date copy can never be created, though: all the nodes are never online and never stop adding files. Adding a file in this network looks like a wave: there can be multiple waves and the add-only property allows these waves to go thru each other.

The add-only property is a significant restriction, but it doesn't impact forum-like sites much because everything there is usually written once. It would be much better to have a decentralised git repository editable by everyone with correctness enforced by a pre-submit script. We'd essentially need to implement a peer-to-peer `git pull --rebase && git push` where the remote peer isn't trusted. I don't see a way to make such a system work.

Now the internal repsentation of a local copy of the shared folder: it's just a flat list of IPFS files:

```
0073ff # { name:"RULES",            data:"os.exit(0)" }
828990 # { name:"foo/abc",          data:"123" }
8bcc62 # { name:"foo/bar",          data:"Hello world!" }
c7aff2 # { name:"test12",           data:"11" }
883000 # { name:"config/data.json", data:"{}" }
```

The fact that it's a flat list doesn't mean that it can only represent flat shared folders. Each file contains two parts:
- `name` - it's a string that looks like `foo/bar`.
- `data` - this is the actual content, like a markdown file.

For example:

```
$ ipfs ls 8bcc62
bc0087  7 name
771caf 12 data

$ ipfs cat bc0087
foo/bar

$ ipfs cat 771caf
Hello world!
```

In other words, a shared folder can be thought of as a list of JSON objects, where each JSON object has `name` and `data` fields. In IPFS terms every text string is a file that has its own hash (IPFS CID) and a group of such fields is called an IPFS directory which has its own hash.

In practice, this dictionary is stored like a usual dir in a local file system, while the IPFS hashes are used to transmit data between peers:

```
RULES
foo/
  bar
  abc
test12
config/
  data.json
```

## The rules script

Every shared folder has the `RULES` script that verifies every new file before adding it locally. This script is immutable, just like any other IPFS file. Thus if a mistake is made in it, it cannot be fixed later. However this rules script can contain a minimum bootstrap code that goes to the folder called `scripts` for example, looks there for a specific file and runs it. The rules can be as simple as verifying that every new file has a signature, while the public RSA key is hardcoded in the script.

In addition to this, the files have a specific order. This is important because `RULES` may accept files if they are added in one order and reject them if they are added in another order. Simple example: if we add first `admins/mrsmith` and then `users/johndoe`, the `RULES` script approves this, but if we add the user first, the script will complain that there are no admins to approve this user.

> Hard question. Two nodes, A and B, have two slightly different sets of files added in different order. What's the correct way to merge them? This might be related to [OT](https://en.wikipedia.org/wiki/Operational_transformation).

> Why not to use the same format [.git](https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain) uses? This would enable all the tools written for git.

The rules script can be written in any language that can run on all the platforms: Android/iOS phones, Windows/MacOS/Linux desktops and Chrome/Firefox/Safari/Edge browsers. There are two candidates for this:

- Lua. Small VM written in C. Coding-wise it's even better than JS in some aspects. It can be run in the browsers, but that needs a VM written in JS.
- JS. Runs everywhere but needs a JS VM. Not a problem on desktops as they have Node. Not a problem in the browsers, obviously. And shouldn't be a problem in Android/iOS as those have the WebView components that include JS engines. The problem is that different JS engines support different subset of the latest JS spec. The most notable feature is `async/await`.

The rules script needs to have access to the local copy of the shared folder and to the IPFS API to download actual file data and run checks on it. Here is an example of the rules script written in JS:

```js
import ipfs from 'ipfs';
import repo from 'repo';

// Checks that every new file is a markdown file in the docs dir.
export async function verify(filehash) {
  // The file param here is a IPFS hash of a IPFS object with fields
  // path and data. So first we need to resolve this IPFS hash.
  let file = await ipfs.get(filehash);
  let path = await ipfs.get(file.path);

  if (!/^docs\/\w+\.md$/.test(path)) {
    console.log('File path must look like docs/*.md');
    return false;
  }

  if (await repo.exists(path)) {
    console.log('The file already exists.');
    return false;
  }
  
  let data = await ipfs.get(file.data);

  if (data.length < 100) {
    console.log('The file needs to have at least 100 chars.');
    return false;
  }

  if (!/^[\x20-\x7F]+$/.test(data)) {
    console.log('Only ASCII chars are allowed.');
    return false;
  }

  return true;
}
```

The problem here is that if the JS engine doesn't support `async/await` or the `import/export` syntax, converting this script to an equivalent ES5 version is highly non trivial, but doable even at runtime, e.g. by the `babel` module.

## Typical examples of such shared folders

This is a list of typical forum models used on different websites.

### Anybody can post, moderators can ban users and hide posts

The structure of such a forum can look like this:

```
RULES
moderators/
  mrsmith
  johndoe
users/
  1ec45d
  629bfd
  009ff2
  177bbc
banned-users/
  629bfd
hidden-comments/
  882bc5
posts/
  772781/
    README
    comments/
      bbc621
      882bc5
      889900
  20019c/
    README
    comments/
```

The rules ensure the following:

- Only the admin can add files to `moderators` dir. The public RSA key of the admin is in the `RULES` script. Since only the admin can add files there, there is no risk of name conflicts and thus files can have meaningful names. Contents of `moderators/mrsmith` may look like this:
  ```
  USER-ID: 177bbc
  SIGNED-BY: admin b00..17c
  ```
- Anybody can add files to `users` dir. Well, the rules present a captcha or something like that to prevent creating users in batches. Since anybody can add files there, the filename must be a long unique hash, which is ensured by the rules too. Files in the `users` dir contain some user info and their public RSA keys. Contents of `users/177bbc`:
  ```
  DISPLAY-NAME: mrsmith
  PUBLIC-KEY: 63c...887
  ```
- Only moderators can add files to the `banned-users` dir. The rules check that every file there is signed by a public key from the `moderators` dir. This model implies that once a user is banned, it cannot be un-banned. Contents of `banned-users/629bfd`:
  ```
  SIGNED-BY: mrsmith 177bbc 009...725
  ```
- `posts` is editable by those who are in `users` and not in `banned-users`. The dirname should be a hash of the `README` file. As a side effect, posts with the same `README` will be merged. The `README` file should have a signature of the user who added it, as well as the user id:
  ```
  This is my new cat: ![](http://contoso.com/123/cat.jpg)
  
  SIGNED-BY: 177bbc 81b...090
  ```
- The same rule for `comments`.
- `hidden-comments` is editable by `moderators` only. As you see, comments cannot be completely removed or forcibly erased from the local storage of every participant. Instead, the UI for that forum hides the comments, but may present an option to unhide them. The UI may also choose to actually erase the hidden comments from the local storage, but can't force others to do the same.

Now how an attacker may compromise this forum. Since files aren't removable, the only way is to spam. Let's assume that the UI for this forum is written in such a way that it actually deletes files in `hidden-comments` once they become too old, so if the spambot succeeds in adding a file there, it will eventually make the network erase the comment completely. The spambot may disable the `RULES` script locally and may add any files in any order, but it will need to convince others to do the same.

- Adding a file to `hidden-comments` will be rejected because the file needs to be signed by someone from `moderators`.
- Adding someone to `moderators` won't work because the spambot can't fake the admin's signature.

Thus a spambot cannot do much harm in this forum.

### Invite-only model

In this forum there is a set of approved users and a set of candidates who want to be approved:

```
RULES
users/
  241800
  288111
  28cbcc
  98ccc8
candidates/
  213322/
    241800
  828cbc/
```

Here `213322` already got approval from `241800` and now can add himself to `users`.

How can we compromise this system? We need to bring our candidate to the `users` group. Just adding `828cbc` to `users` won't work because we can't fake signatures of real users. However we can add locally a fake user whose signature we know and make it sign our candidate, while the candidate can sign our fake user:

```
users/
  111222
  828cbc
candidates/
  111222/
    828cbc
  828cbc/
    111222
```

Now whoever gets to sync with this copy might be confused because from the `RULES` point of view everything looks correct: all candidates have approvals from those who are approved users. This is where the order of the files becomes important. We can't send all the new files in one batch to a peer in the network. Instead, we have to send the files one by one and the peer will verify them one by one. So we have to choose an order in which we'll be sending the files:

- If we add someone to `users` first, the peer will reject this.
- If we try to approve our candidate `828cbc` with `111222`, the peer will notice that there is no `users/111222` file yet.
- If we try to approve our fake user `111222` with `828cbc`, the peer will notice that `828cbc` cannot approve others.

No matter which order we choose, the peer will reject it.

There is a reasonable concern that if anybody can propose new candidates, that list may quickly become huge and since everyone in the forum needs to store it, this will be a problem. The solution is that the `candidates` dir doesn't need to be kept in this forum: it can be kept elsewhere, maybe as a shared folder too, and `candidates` here can just point to that place. The users of the forum only need to see that everyone in the `users` dir has a valid signature.

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

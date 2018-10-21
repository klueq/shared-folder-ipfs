# Internal structure

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

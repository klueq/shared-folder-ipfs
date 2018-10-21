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
- [Joining the forum and discovering peers](docs/join.md)
- [How this works](docs/how-it-works.md)
- [The sync protocol](docs/sync.md)
- [UI implementation details](docs/ui.md)

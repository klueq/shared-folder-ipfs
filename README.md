# What is this?
Think about a shared folder on an FTP server where many users can add files. In this folder there is also a special `rules` script that enforces certain rules about what files can be added, who can add them and so on. This shared folder has some disadvantages: there is an admin who is above the rules and all the files are stored in one place. The idea of this project is to use [IPFS](https://ipfs.io) to implement this shared folder pattern without the disadvantages: there won't be an admin and there won't be a single place where the files are stored.

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

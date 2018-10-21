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

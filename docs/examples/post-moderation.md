# Post moderated forums

In such forums anybody can post, moderators can ban users and hide posts

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

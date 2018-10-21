# Invite-only forums

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

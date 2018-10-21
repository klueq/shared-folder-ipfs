# The rules script

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

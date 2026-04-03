---
title: Writing a self-update bash script                    
date: 2026-04-03T07:58:00+01:00
tags: 
  - bash
  

author: totomz


---

# Can I update a shell script while it is running?

## TL;DR
It depends on the shell interpreter (bash, zsh, sh) and operating system (linux, bsd, osx).

Generally speaking (Linux/bash) it will **not work**. Bash scripts ARE NOT loaded once in memory, 
but the file is read in blocks. Any change to the script will affect the next blocks, giving unpredictable results.

## "Safe" ways to self-update a script
### Update and terminate
A one-line command that updates the script and exits immediately
```shell
cat "new data 1" "new data 2" > "${script}" && rm -rf "${script}" ; exit 0
```
The `&&` and `;` force the interpreter to read this whole line. The `${script}` contents change, but the script exits immediately so there are no side effects

An improved version is to restart the script once it has been reloaded
```bash
#!/bin/bash
SCRIPT="$0"
LAST_MODIFIED=$(stat -c %Y "$SCRIPT")

while true; do
    CURRENT=$(stat -c %Y "$SCRIPT")
    if [[ "$CURRENT" != "$LAST_MODIFIED" ]]; then
        exec "$SCRIPT" "$@"   # restart itself
    fi
    do_stuff
    sleep 2
done
```

## Force bash to read the whole script
Wrapping the whole script in a `{ ... }` block forces bash to parse everything before executing the first 
command. Any modification to the file mid-run has no effect.
```shell
#!/bin/bash
{
    echo "step 1"
    slow_operation
    echo "step 2"   # bash read this before step 1 even started
}
```

Same effect with `bash <(cat script.sh)`: cat reads the whole file upfront and pipes it to bash, 
so the file on disk is never re-read.

# What is happening
Let's start with a simple script
```shell
$ cat hello.bash
#!/bin/bash

echo "Hello Unix & Linux!"
```

We can execute it with strace
```shell
strace -s 2000 -o strace.log ./hello_ul.bash
```
In the strace.log, after skipping the ~150 syscall to load the interpreter and 350KB of locale stuff, bash reads and executes the script  
```
openat("./hello_ul.bash", O_RDONLY) = 3
ioctl(3, TCGETS, ...) = -1 ENOTTY
lseek(3, 0, SEEK_CUR) = 0
read(3, "#!/bin/bash\necho \"Hello Unix & Linux!\"\n\n", 80) = 41
lseek(3, 0, SEEK_SET)
prlimit64(RLIMIT_NOFILE, ...) = {rlim_cur=1024, rlim_max=1048576}
fcntl(255, F_GETFD) = -1 EBADF
dup2(3, 255) = 255
close(3)
fcntl(255, F_SETFD, FD_CLOEXEC)
read(255, "#!/bin/bash\necho \"Hello Unix & Linux!\"\n\n", 41) = 41
```
Bash opens the script and immediately calls ioctl(TCGETS) on it — this is a sanity check to confirm the fd is not a terminal (it isn't, so it gets ENOTTY, which is the expected result). Then it does an initial read of 80 bytes just to sniff the shebang and decide how to proceed.
After that, bash checks RLIMIT_NOFILE to know the upper bound of available file descriptors, then calls fcntl(255, F_GETFD) — which returns EBADF because fd 255 is not yet open, confirming it's safe to use. It then calls dup2(3, 255) to move the script fd from 3 to 255.
This is deliberate: bash reserves fd 255 (and generally high-numbered fds) for its own internal file handles, keeping the low fds (0, 1, 2 and upward) free for the script's own redirections and pipe plumbing. If bash kept the script open on fd 3, a script doing something like exec 3>somefile would silently clobber it.
Finally it sets FD_CLOEXEC on fd 255 so it gets automatically closed across any exec calls in child processes, and reads the full script content one more time from the new fd before starting execution.

But what happens if the script gets larger?
```shell
$ ( 
    echo '#!/bin/bash'; 
    for i in {1..100000}; do printf "%s\n" "echo \"$i\""; done 
  ) > ascript.bash;
    
$ ls -ahl
-rw-rw-r--  1 totomz totomz 1.3M Apr  3 08:04 ascript.bash
```

If we run it again with `strace`, we will see something like this
```shell
write(1, "751\n", 4)                    = 4
write(1, "752\n", 4)                    = 4
write(1, "753\n", 4)                    = 4
read(255, "\"754\"\necho \"755\"\necho \"756\"\necho \"757\"\necho \"758\"\necho \"759\"\necho \"760\"\necho \"761\"\necho \"762\"\necho \"763\"\necho \"764\"\necho \"765\"\necho \"766\"\necho \"767\"\necho \"7
68\"\necho \"769\"\necho \"770\"\necho \"771\"\necho \"772\"\necho \"773\"\necho \"774\"\necho \"775\"\necho \"776\"\necho \"777\"\necho \"778\"\necho \"779\"\necho \"780\"\necho \"781\"\necho \"782\"\necho \"7
83\"\necho \"784\"\necho \"785\"\necho \"786\"\necho \"787\"\necho \"788\"\necho \"789\"\necho \"790\"\necho \"791\"\necho \"792\"\necho \"793\"\necho \"794\"\necho \"795\"\necho \"796\"\necho \"797\"\necho \"7
98\"\necho \"799\"\necho \"800\"\necho \"801\"\necho \"802\"\necho \"803\"\necho \"804\"\necho \"805\"\necho \"806\"\necho \"807\"\necho \"808\"\necho \"809\"\necho \"810\"\necho \"811\"\necho \"812\"\necho \"8
13\"\necho \"814\"\necho \"815\"\necho \"816\"\necho \"817\"\necho \"818\"\necho \"819\"\necho \"820\"\necho \"821\"\necho \"822\"\necho \"823\"\necho \"824\"\necho \"825\"\necho \"826\"\necho \"827\"\necho \"8
28\"\necho \"829\"\necho \"830\"\necho \"831\"\necho \"832\"\necho \"833\"\necho \"834\"\necho \"835\"\necho \"836\"\necho \"837\"\necho \"838\"\necho \"839\"\necho \"840\"\necho \"841\"\necho \"842\"\necho \"8
43\"\necho \"844\"\necho \"845\"\necho \"846\"\necho \"847\"\necho \"848\"\necho \"849\"\necho \"850\"\necho \"851\"\necho \"852\"\necho \"853\"\necho \"854\"\necho \"855\"\necho \"856\"\necho \"857\"\necho \"8
58\"\necho \"859\"\necho \"860\"\necho \"861\"\necho \"862\"\necho \"863\"\necho \"864\"\necho \"865\"\necho \"866\"\necho \"867\"\necho \"868\"\necho \"869\"\necho \"870\"\necho \"871\"\necho \"872\"\necho \"8
73\"\necho \"874\"\necho \"875\"\necho \"876\"\necho \"877\"\necho \"878\"\necho \"879\"\necho \"880\"\necho \"881\"\necho \"882\"\necho \"883\"\necho \"884\"\necho \"885\"\necho \"886\"\necho \"887\"\necho \"8
88\"\necho \"889\"\necho \"890\"\necho \"891\"\necho \"892\"\necho \"893\"\necho \"894\"\necho \"895\"\necho \"896\"\necho \"897\"\necho \"898\"\necho \"899\"\necho \"900\"\necho \"901\"\necho \"902\"\necho \"9
03\"\necho \"904\"\necho \"905\"\necho \"906\"\necho \"907\"\necho \"908\"\necho \"909\"\necho \"910\"\necho \"911\"\necho \"912\"\necho \"913\"\necho \"914\"\necho \"915\"\necho \"916\"\necho \"917\"\necho \"9
18\"\necho \"919\"\necho \"920\"\necho \"921\"\necho \"922\"\necho \"923\"\necho \"924\"\necho \"925\"\necho \"926\"\necho \"927\"\necho \"928\"\necho \"929\"\necho \"930\"\necho \"931\"\necho \"932\"\necho \"9
33\"\necho \"934\"\necho \"935\"\nech"..., 8192) = 8192
write(1, "754\n", 4)                    = 4
write(1, "755\n", 4)                    = 4
write(1, "756\n", 4)                    = 4
write(1, "757\n", 4)                    = 4
```

You'll notice that the file is being read in at 8KB increments, so Bash and other shells will likely not load a file in its entirety, rather they read them in blocks.
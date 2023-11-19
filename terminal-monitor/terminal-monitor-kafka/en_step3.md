## Test

Let's open a new terminal and test if our application is running.

Firstly, since there is no method to determine the current step from the terminal, we need to specify which step we are in.

```sh
export current_step=step1
```

Execute three random commands that will produce errors. You can press enter after typing random letters.

```sh
cat ~/workspace/tips.txt
```

You will see that a hint is displayed.

Let's do the same for step2.

```sh
export current_step=step2
```

Execute three random commands that will produce errors.

```sh
cat ~/workspace/tips.txt
```

If you see the following output in the last two lines, the process is complete:

```sh
try to use git merge on both branches  |  
research optional flags for git hash object command  |
```
# Chain of Trust
## Releng Tech Talk
### Sep 2017

Note:
- I'm going to talk about the chain of trust.
- Hopefully the talk will take 10-15 minutes, and we'll have time for questions afterwards.

---

## Taskcluster

- massively scalable
- self-serve, powerful
- security challenge

Note:
- We use Taskcluster for our automation.
- Taskcluster is massively scalable... We spin up about 10 million worker instances per year.
- It's self serve, which allows developers to push all their changes without being blocked by releng.
- By its nature, it presents a security challenge for releases.

---

- For taskcluster, there are no official graphs
- Taskcluster scopes allow for arbitrary tasks of the same type

![basic2](img/basic2.png)

Note:
- If you have the scopes to trigger an official task graph, you can also trigger any arbitrary task or graph of the same type.
- The left hand side represents an official graph. Someone pushes to the tree, we create a decision task that creates an official build.
- The middle represents a well-formed graph, where I've modified the decision task, so the build is now modified.
- The right hand side represents an arbitrary build that I've submitted directly.
- Taskcluster has no concept of "official" graphs. There are only well formed tasks and conventions.

---

We want to

- allow this flexibility in general
- restrict release tasks.

Note:
- Taskcluster's flexibility is its strength. Let's only restrict its behavior for release tasks.

---

Sensitive processes:

- secrets must be protected by more than just scopes
- disallow arbitrary actions
- we need to know the request is valid (trace back to tree)

Note:
- Taskcluster secrets is a useful service, but we shouldn't use it for anything we really need to be secure.
- Docker-worker and generic-worker allow for arbitrary scripting in the task definition. This is a security hole for sensitive tasks.
- Tracing the request back to the tree means we can trust the request as much as we trust the tree.

---

Protecting secrets, limiting actions: scriptworker

- under releng control
- static/limited abilities (no bash scripts in task definitions)
- verifies request validity before proceeding (chain of trust)

Note:
- Scriptworker is designed to handle these sensitive tasks.
- It isolates secrets to protected instances, and limits the logic that can be run.

---

Request validity: Chain of Trust

- releng needs to safeguard everything from hg.m.o all the way to the user
- hg.m.o security is a separate problem
- trace the request back to the tree via the chain of trust

Note:
- We assume that hg.m.o is secure; other teams are handling that problem.
- Releng handles the security from hg.m.o through release; to do so, we trace the request back to a trusted tree.

---

Attack vectors

- altered or arbitrary task definition
- malicious docker image
- alter artifacts at rest
- alter pointers to artifacts, like moving softlinks

Note:
- There are two main ways to attack our release processes on taskcluster.
- Arbitrary task definitions, or altered artifacts.

---

Each important level3 worker has an embedded gpg key.

- docker-worker: the key lives outside the container
- generic-worker: the key is not readable by the task user
- scriptworker: the worker is limited and under releng control

Note:
- Each important level 3 workerType has an embedded gpg key.
- This key is the second factor.
- This allows us to create a trail, or chain, to follow back to the tree.

---

Chain of Trust artifact

```python
-----BEGIN PGP SIGNED MESSAGE-----
Hash: SHA256

{
  "artifacts": {
    "path/to/artifact": {
      "sha256": "abcd1234"
    },
    ...
  },
  "chainOfTrustVersion": 1,
  "environment": {
    # worker-impl specific stuff, like ec2 instance id, ip
  },
  "task": {
    # task defn
  },
  "runId": 0,
  "taskId": "...",
  "workerGroup": "...",
  "workerId": "..."
}
-----BEGIN PGP SIGNATURE-----
...
-----END PGP SIGNATURE-----
```

Note:
- The worker creates a chain of trust artifact after running a task.
- The artifact contains information about the task and artifacts, so any modifications are detectable.
- The entire artifact is gpg signed.

---

## Verification

- run on scriptworkers, before they run their task
- verify task definitions against decision task's `task-graph.json`
- verify upstream artifacts match shas
- verify all upstream tasks that modify inputs, until we reach the tree
- restricted scopes can only run on allowlisted trees

Note:
- We run chain of trust verification on the scriptworkers before running the task. If verification fails, we don't run the task.
- We find important tasks upstream. Any task that modifies inputs is important.
- We verify artifacts haven't been modified while at rest.
- We verify restricted scopes only run on restricted trees.

---

Trace back to the tree

![signing](img/signing.png)

---

## Recent changes

---

The decision task is key.

In CoT v2, we will

- simplify the decision task by embedding more logic in the docker image
- verify the decision task came from the tree - including docker shas, workerType, etc (eyaml)
- simplify downstream task verification

---

## Q/A

- [More in-depth slides from the all-hands](https://gitpitch.com/escapewindow/slides-sf-2017/cot)
- [More docs](http://scriptworker.readthedocs.io/en/latest/chain_of_trust.html)

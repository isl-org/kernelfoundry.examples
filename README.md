# KernelFoundry Dev Container Example

A ready-to-run [KernelFoundry](https://github.com/isl-org/kernelfoundry) task: a naive SYCL
matrix multiplication kernel that the algorithm rewrites, benchmarks and profiles on your
Intel GPU. Everything it needs — oneAPI, PyTorch XPU, unitrace, VTune — lives in a container
image, so nothing is installed on your machine.

Clone this repo with one click in VS Code:

<div align="center">

[![Clone to disk](https://img.shields.io/static/v1?label=Dev%20Containers&message=Clone%20the%20example&color=blue&logo=visualstudiocode)](https://vscode.dev/redirect?url=vscode%3A%2F%2Fvscode.git%2Fclone%3Furl%3Dhttps%3A%2F%2Fgithub.com%2Fbenjaminum%2Fkernelfoundry.example)

</div>

Or by hand from the command line:
```bash
git clone https://github.com/isl-org/kernelfoundry.examples
code kernelfoundry.example
```

## Requirements

- A Linux host with an Intel GPU and its driver loaded, so that `/dev/dri` exists.
- Docker, and VS Code with the Dev Containers extension.
- An API key for an LLM, `OPENAI_API_KEY` or `ANTHROPIC_API_KEY`.

Export the key somewhere VS Code itself inherits it, before starting VS Code — setting it in
an already-running terminal is too late, because the value is read by VS Code rather than by
your shell. To use a gateway instead of the vendor endpoint, export `OPENAI_BASE_URL` or
`ANTHROPIC_BASE_URL` the same way.

Alternatively, drop a `.env` file with these variables at the root of this workspace.
`kernelfoundry.algorithm` loads it automatically from the current directory, so it works
without touching VS Code's own environment. Point `KERNELFOUNDRY_ENV_FILE` at a different path
to use one from elsewhere.

## Running the example

From a terminal inside the container:

```bash
python -m kernelfoundry.algorithm run task=./matmul task_origin=custom \
    job_name=matmulg gpu_arch=bmg
```

Set `gpu_arch` to your own architecture; `bmg` is Battlemage, `lnl` is Lunar Lake, and `ptl` is Panther Lake. The same
defaults live in `matmul/config.yaml`. For a quick first run that exercises the whole loop
once, add `max_iters=1 branches_per_iteration=1`.

Results are written to `runs/`. To browse them, start the UI and open the forwarded port
8885:

```bash
python /opt/kernelfoundry/start_gui.py
```

## Driving it from a coding agent

The KernelFoundry MCP server is registered here, so an agent can build, test and optimize the
kernel itself rather than you running the command above. Each editor reads its own file and
they all describe the same server, so use whichever matches your tool and ignore the rest:

```
.vscode/mcp.json            VS Code, under a "servers" key
.cursor/mcp.json            Cursor
.antigravity/mcp_config.json  Antigravity
```

Note that there is no `.mcp.json` at the root, the file most other clients read, because VS Code
mishandles it in a dev container: it starts those servers on the machine running the window
rather than in the container.

Two tools appear once the server is connected. `build_and_test` compiles a task folder, runs
its correctness tests and benchmark, and returns the log with runtime statistics, which is
the fast loop for an agent editing the kernel by hand. `submit_task` runs the full
optimization loop instead and writes the best kernel it finds back between the `EVOLVE`
markers. Try it with a prompt like:

```
Use build_and_test on ./matmul, then make the kernel faster and test again.
```


## If something goes wrong

**No GPU in the container.** The render node on your host is owned by a group whose numeric
id differs per machine, and the container has to carry that id. Run
`stat -c %g /dev/dri/renderD128` on the host, put that number in the `--group-add` line of
`.devcontainer/devcontainer.json`, and rebuild the container. The container's postCreate step
prints the number to use.

**TLS errors when calling the model.** The container trusts your host's CA bundle through a
bind mount, which covers a gateway signed by a private root. If your host keeps its bundle
somewhere other than `/etc/ssl/certs/ca-certificates.crt`, container creation fails with
"bind source path does not exist": set `KF_CA_BUNDLE` on the host, or delete the `mounts`
entry.

**`spawn /usr/local/bin/entrypoint.sh ENOENT`.** The client is starting the server on the
machine running the editor rather than in the container, where that path does not exist. The
window has to be connected to the dev container, since its remote connection is what decides
where its servers run, and the definition has to be the workspace one: a copy in your user or
global MCP configuration belongs to your own machine and always runs there. In VS Code, "MCP:
List Servers" shows which configuration each server came from. An agent running outside the
container needs a `docker exec` wrapper instead of the command in these files.

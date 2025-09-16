
# Namespace and Cgroups Exploration 

---

## Namespace exploration

---

## Explore pid namespace

Take a look at the current processes:
```bash
ps aux
```

Create a new namespace and run bash:
```bash
sudo unshare --fork --pid --mount-proc bash
```

Take a look around:
```bash
ps aux
```

What is PID 1 in the new namespace?

Inside the new namespace, run a long-lived process like `top`. Then in a separate SSH connection to the Docker host, look at the current processes:
```bash
ps aux
```

Find the namespaced processes—what PIDs do they have?

Exit when you're done exploring.

---

## Explore Docker namespaces

Run any container, for example:
```bash
# Sleep for 5 minutes and exit
docker run --detach alpine sleep 300
```

Copy the container ID for your running container, and export it:
```bash
export CID=<container ID>
```

### docker exec
Explore inside the container:
```bash
docker exec -it $CID /bin/sh
# Look around
ps aux
ls /
exit
```

Now use the more general Linux `nsenter` utility to enter the container's namespaces:

```bash
PID=$(docker inspect --format '{{.State.Pid}}' $CID)
sudo nsenter --target $PID --mount --uts --ipc --net --pid /bin/sh
```

Look around, and `exit` when done.

---

## Explore memory limits with cgroups v2

> In cgroups v2, there's a **unified hierarchy** under `/sys/fs/cgroup`.

Install memhog (if not installed):

```bash
sudo apt install -y stress-ng
```

Create a cgroup:

```bash
sudo mkdir /sys/fs/cgroup/playground
```

Set a 10MiB memory limit:
```bash
echo $((10 * 1024 * 1024)) | sudo tee /sys/fs/cgroup/playground/memory.max
```

Allow your user to run inside this cgroup:
```bash
echo $$ | sudo tee /sys/fs/cgroup/playground/cgroup.procs
```

To run inside the cgroup more formally:
```bash
sudo systemd-run --unit=limitedmem --property=MemoryMax=10M --pty bash
```

Then, in that shell, try exceeding the limit:
```bash
stress-ng --vm 1 --vm-bytes 100M --vm-hang 0
```

Exit and clean up:
```bash
sudo rmdir /sys/fs/cgroup/playground
```


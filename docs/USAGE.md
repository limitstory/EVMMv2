# EVMMv2 Usage

This document describes how to execute EVMMv2 after completing the system setup.
EVMMv2 is provided as a reference implementation and is executed directly from source.

---

### CRI-O Socket Permission

EVMMv2 interacts directly with the CRI-API through the CRI-O Unix domain socket
(`/var/run/crio/crio.sock`).  
In the tested environment, the socket permission may be reset during runtime
(e.g., due to CRI-O or systemd behavior), which prevents EVMMv2 from accessing
container runtime information.

To ensure continuous monitoring and control, the socket permission must be
kept writable by the EVMMv2 process. The following command was used in the
experimental setup to maintain the required permissions:

```bash
while true; do
  sudo chmod 777 /var/run/crio/crio.sock
  sleep 100
done
```

---

## Running EVMMv2

EVMMv2 is implemented in Go and runs as a user-space control component.
After cloning the repository and completing the system setup, EVMMv2 can be executed as follows.

### 1. Fetch dependencies

```bash
go mod tidy
```

### 2. Run EVMMv2

```bash
go run main.go
```
This starts the EVMMv2 control component.


Runtime Behavior
Once started, EVMMv2 monitors container memory behavior and applies runtime memory management decisions via the CRI-API.
EVMMv2 dynamically adjusts container memory allocations without restarting containers.
Depending on runtime conditions, EVMMv2 may trigger memory updates or checkpoint-based operations.



Notes

This repository provides a research prototype rather than a production-ready tool.

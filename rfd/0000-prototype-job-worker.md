---
author: Bill Rich (bill@dangeri.ch)
state: draft
---
 
# RFD 0000 - Prototype Job Worker Service 

## Required Approvers

- Engineering: @rosstimothy || @zmb3, Marek Smoliński || @smallinsky, Anton Miniailo || @AntonAM

## What

This document presents the design for a prototype job worker service that
manages and executes arbitrary Linux processes. It details the components,
functionality, and architectural considerations of the service, including the
library, API, and client-side interactions. The service is designed to provide
APIs for starting, stopping, querying the status of jobs, and streaming the
output of running jobs.

## Why

The purpose of creating this prototype is to establish a foundation for a
scalable, secure, and efficient way to handle process management on Linux
systems. This service aims to streamline development operations and ensure
robust process isolation and resource control, enhancing the security and
performance of running jobs. This prototype also serves as a critical step
towards evaluating the practical aspects of process management in controlled
environments, focusing on rapid development and deployment with minimal code
complexity.

## Details

### Components
#### Library
- **Functionality**: The library will include methods to start, stop, query
  status, and get the output of jobs. It will handle streaming output from the
  inception of process execution. 
- **Concurrency**: Support for multiple concurrent clients will be implemented.
  Resource Control: Utilize cgroups for CPU, memory, and disk I/O control per job.
  Resource Isolation: Implement PID, mount, and network namespaces to ensure job isolation.
#### API
- **Technology**: gRPC will be used due to its high performance and suitability
  for streaming data.
- **Methods**:
  - `StartJob`: Starts a new job.
  - `StopJob`: Terminates an existing job.
  - `GetJobStatus`: Returns the current status of a job.
  - `StreamOutput`: Streams the output of a running job.
- **Security**: Utilize mTLS for authentication with a strong set of cipher
  suites and certificate configurations. 
#### Client
- **Interface**: A CLI that can interact with the worker service to start,
  stop, get status, and stream the output of jobs. 
- **Features**: The CLI will communicate with the server using the gRPC API,
  handling all server responses and streaming outputs effectively.

### Detailed Design
#### Streaming Approach
- **Implementation**: Utilize gRPC streaming to send output from the worker to
  the client in real-time. Buffering strategies will be applied to handle
  high-frequency output efficiently.
#### TLS Setup
- **TLS Version**: TLS 1.3 will be used due to its enhanced security features.
- **Cipher Suites**: Use strong cipher suites such as TLS_AES_256_GCM_SHA384 to
  ensure secure communication.
- **Certificate Management**: Use hardcoded certificates for simplicity, with
  TODO items added for future implementation of a more robust certificate
  management system.
#### Process Execution Lifecycle
1. **Start Job**:
    - Jobs are initiated via a `StartJob` RPC call, which includes the command and its arguments.
    - Each job is assigned to a unique cgroup for resource management.
    - Jobs are executed within isolated namespaces to prevent interference and enhance security.
2. **Stream Output**:
    - Output from the process is captured from the start and streamed to the
      client using gRPC's streaming capabilities. 
3. **Stop Job**:
    - Jobs can be terminated using a StopJob RPC call.
    - Processes are immediately killed, ensuring all resources are properly released.
4. **Query Status**:
    - The status of any job can be queried to return whether it is running,
      stopped, or completed. For this initial prototype, the information provided
      will be basic. However, a final product would expand on this to include
      more detailed information, such as the return code, error output, and other
      relevant diagnostics to provide a comprehensive view of the job's execution
      status.

#### Namespace Implementation
Namespaces are a feature of the Linux kernel that provides isolation for
running processes. They make it possible for processes to have their own view
of the system's global resources, such as process IDs, network stacks, mount
points, and interprocess communication resources. We will specifically use the
following namespaces:
  - **PID Namespace**: Ensures process isolation by providing
    processes with separate PID namespaces. This prevents them from seeing or
    interacting with processes in other namespaces.
  - **Network Namespace**: Isolates network interfaces, IP routing tables, port
    numbers, etc., allowing processes to operate on a network as if they were
    the only ones on the system. To stay in scope, the network will remain
    isolated.
  - **Mount Namespace**: Isolates filesystem mount points, allowing processes
    to have their own set of filesystems visible to them.

##### Utilization Strategy
Our service will utilize the syscall package in Go to unshare and manage
namespaces. The following code snippet demonstrates how a new process can be
started in a new set of namespaces:

```go
import "syscall"

func setupNamespaces() error {
    // Unshare creates new namespaces for the calling process
    if err := syscall.Unshare(syscall.CLONE_NEWPID | syscall.CLONE_NEWNET | syscall.CLONE_NEWNS); err != nil {
        return fmt.Errorf("failed to create namespaces: %v", err)
    }
    return nil
}
```

#### Cgroup Implementation
Control groups (cgroups) are another kernel feature that limits, accounts for,
and isolates the resource usage (CPU, memory, disk I/O, etc.) of process
groups. We will implement cgroups to ensure that each job runs within
predefined resource limits, improving system stability and preventing any job
from monopolizing system resources.

##### Configuration Approach
Cgroups will be managed by interacting with the cgroup filesystem typically
mounted under /sys/fs/cgroup. Here is how we plan to create and manage cgroups:

```go
import (
    "io/ioutil"
    "path/filepath"
)

func createCgroup(path, resource, limit string) error {
    cgroupPath := filepath.Join("/sys/fs/cgroup", resource, path)
    if err := os.MkdirAll(cgroupPath, 0755); err != nil {
        return err
    }
    return ioutil.WriteFile(filepath.Join(cgroupPath, "tasks"), []byte(strconv.Itoa(os.Getpid())), 0644)
}
```

### Command Usage

```
Usage: jobworker [command] [options]

A CLI tool to interact with the Job Worker Service, providing secure job management capabilities through mTLS.

Commands:
  start       Start a new job with the specified command and arguments.
  stop        Stop a running job using its unique job ID.
  status      Query the status of a job using its unique job ID.
  stream      Stream the output of a running job using its unique job ID.
  run         Run a local process in specified namespaces and cgroup.

Options:
  --help      Show this help message and exit.
  --cert      Specify the client certificate (and optionally key) file path for mTLS authentication.
              Defaults to ~/.ssl/client.pem if not specified.

Examples:
  jobworker start --cmd "bash myscript.sh"                          Start a job using default certificate and key.
  jobworker stop --job-id 12345 --cert /path/to/custom.pem          Stop a job using a specified PEM file.
  jobworker status --job-id 12345                                   Query the status of a job using the default certificate and key.
  jobworker stream --job-id 12345 --cert /path/to/custom.pem        Stream output of a job using a specified PEM file.
  jobworker run --cmd "ls -l"                                       Run 'ls -l' in new PID, mount, and network namespaces with specified cgroup limits.

Note:
  'run' command requires root privileges to manage namespaces and cgroups.
```

### Security Considerations
- **Authentication**: In this prototype, mTLS will be exclusively used for
  secure communication, where both client and server verify each other using
  pre-generated certificates. To maintain project scope, comprehensive
  certificate and user management will not be included at this stage. However,
  a final product should ideally incorporate these features to enhance security
  and manageability.

- **Authorization**: For this prototype, a simple authorization scheme will be
  implemented, using hardcoded permissions to determine if a client is
  authorized to start, stop, or query specific jobs. This approach is chosen
  due to its simplicity and sufficiency for the limited scope of the prototype.
  For a fully developed system, a more comprehensive permissions configuration
  system would be designed to offer finer control and scalability.

### Development Considerations
- **Code Simplicity**: Aim to write as little code as possible to prevent
  bloating and reduce development time.
- **Configuration**: Use hardcoded values for most configurations to speed up
  the initial setup.
- **TODO Items**:
  - Implement dynamic certificate management.
  - Extend configuration options for production readiness.
  - Enhance error handling and logging mechanisms.
  - Support cgroup limits beyond the hardcoded defaults.
  - Add network configuration and setup for network namespaces.

# teleport-interview
https://github.com/gravitational/careers/blob/main/challenges/systems/challenge-1.md

# Jobs Worker Architecture

# Goal

*   Build a controller and worker architecture that supports job requests managed by the controller as GRPC requests on workers.
*   Any application outside of the GRPC cluster can make HTTP requests to trigger jobs on the cluster and can also create distributed jobs as well.


# General Overview

*   Support single controller instance and supports spawning multiple worker instances.
*   The controller is tasked with managing all worker states. Spawned worker instances are registered at creation/inception and deregistered at time of job completion.
*   The controller and the workers authenticate with each other via mTLS. 
*   The controller and worker instances have GRPC-APIs. Client instances are configured to create new worker instances when jobs are created. 
    *   The controller GRPC-API support:
        *   Register a worker
        *   Deregister a worker
    *   The worker GRPC-API support:
        *   Start a job with job parameters and options as command flags and optional parameters
        *   Stop a job with job parameters and options as command flags and optional parameters
        *   Return the status of a job
        *   Relay a stream output for running jobs by querying via a jobID.
*   The controller also has an HTTP-API (which operates as surrogate agent for to GRPC requests):
    *   Start a job on a specific worker (with worker ID, command and path to the job)
    *   Stop a job on a specific worker (with worker ID and job ID)
    *   Query a job on a specific worker (with worker ID and job ID)
    *   Return the output stream of a job a specific worker (with worker ID and job ID)


# Controller Overview

* [Controller Details](Server/Controller/README) to be placed here. 
* # Controller

Controller exposes an HTTP-server that gives a gateway to the controller-worker-cluster. HTTP requests to the controller api are made to workers on behalf of client cli.

- controller runs an HTTP server as well as a GRPC server.
- controller keeps record of the available workers in the cluster.
- Workers can register and deregister on the controller.


## Data Structures
---

# Workers

Workers exposes a GRPC-server to communicate with the controller. Main job of workers are to run specific jobs requested by the controller and report on those jobs. Jobs can be ruby/python/bash scripts or any executables available on the worker machine.

## Data Structures
---

### Jobs

```golang
// jobsMutex is the lock to access jobs map.
// jobs is the map that holds current/past jobs.
//    - key: job id
//    - value: pointer to the created job object.
var (
	jobsMutex = &sync.Mutex{}
	jobs      = make(map[string]*job)
)

// job holds information about the ongoing or past jobs,
// that were triggered by the controller.
//    - id: UUID assigned by the worker and sent back to the controller.
//    - command: command which the controller run the job with
//    - path: path to the job file/executable sent by the controller.
//    - outFilePath: file path to where the output of the job will be piped.
//    - cmd: pointer to the cmd.Exec command to get job status etc.
//    - done: whether if job is done (default false)
//    - err: error while running the job (default nil)
type job struct {
	id          string
	command     string
	path        string
	outFilePath string
	cmd         *exec.Cmd
	done        bool
	err         error
}
```

## Congroller - Worker(s) Configuration
---

[grpc_server]
addr = "${{grpc_server_addr}}"
use_tls =  ${{use_tls}}
crt_file = ${{crt_file}}
key_file = ${{key_file}}

[controller]
addr = ${{controller_addr}}
```

`grpc_server`:
  - `addr`: Address on which the GRPC server will be run.
  - `use_tls`: Whether the GRPC server should use TLS. If `true`, `crt_file` and `key_file` should be provided.
  - `crt_file`: Path to the certificate file for TLS.
  - `key_file`: Path to the key file for TLS.

`controller`:
  - `address`: Address on which the GRPC server of the controller is run.

## GRPC Server
---

#### Methods:
```
service Worker {
  rpc StartJob(StartJobRequest) returns (StartJobResponse) {}
  rpc StopJob(StopJobRequest) returns (StopJobResponse) {}
  rpc QueryJob(QueryJobRequest) returns (QueryJobResponse) {}
  rpc StreamJob(StreamJobRequest) returns (stream StreamJobResponse) {}
}
```

### Variables:
```
message StartJobRequest {
  string command = 1;
  string path = 2;
}

message StartJobResponse {
  string jobID = 1;
}

message StopJobRequest {
  string jobID = 1;
}

message StopJobResponse {
}

message QueryJobRequest {
  string jobID = 1;
}

message QueryJobResponse {
  bool done = 1;
  bool error = 2;
  string errorText = 3;
}

message StreamJobRequest {
  string path = 1;
}

message StreamJobResponse {
  string output = 1;
}
```

`StartJob`:
- Called by the controller(via a Client command) to start a new job.
- `StartJobRequest` has the `command` to run the job with and the `path` to the script/executable.
- `StartJobResponse` returns a `jobID` created by the worker that later used by the controller to specify the created job. GroupIds are not passed back to the calling client.

`StopJob`:
- Called by the controller to stop a job with the given `jobID`.

`QueryJob`:
- Called by the controller to query a job with the given `jobID`.
- Returns if the job status is set as complete, or if there was an `error` and if there was an error, what the `errorText` output.

`StreamJob`:
- Called by the controller to get streaming output of the job's task logs with a given `jobID`.


### Workers

```golang
var (
	workersMutex = &sync.Mutex{}
	workers      = make(map[string]*worker)
)

// worker holds the information about registered workers
//    - id: uuid assigned when the worker first register.
//    - addr: workers network address, later used to create grpc client to the worker
type worker struct {
	id   string
	addr string
}
```

`grpc_server`:
  - `addr`: Address on which the GRPC server will be run.
  - `use_tls`: Whether the GRPC server should use TLS. If `true`, `crt_file` and `key_file` should be provided.
  - `crt_file`: Path to the certificate file for TLS.
  - `key_file`: Path to the key file for TLS.

`http_server`:
  - `addr`: Address on which the HTTP server will be run on.
  

#### Methods:
```
service controller {
  rpc RegisterWorker(RegisterRequest) returns (RegisterResponse) {}
  rpc DeregisterWorker(DeregisterRequest) returns (DeregisterResponse) {}
}
```

### Variables:
```
message RegisterRequest {
  string address = 1;
}

message RegisterResponse {
  bool success = 1;
  string workerID = 2;
}

message DeregisterRequest {
  string workerID = 1;
}

message DeregisterResponse {
  bool success = 1
}
```

`RegisterWorker`:
- All worker instances register to a controller instance when coming online for first time - service discovery is set with an initial default controller server address. 
- `RegisterRequest`  The address parameter sent to the controller instance by worker instances. It is the registered address of the worker's GRPC Server.
- `worker` instance is created for every worker that registers which contains an assigned `workerID` and the `address` for jobs that require parallel execution each instance would be registered to a new control group. Default resource limitations(CPU,MEM,DISK I/O) to the associated control groups are created when worker instances are spawned which can only be set at inception and not after the job has started or is paused. 

`DeregisterWorker`:
- Path is called by worker instances as part of it's shutdown workflow (such as upon job completion and before going offline.) 
-  Controller recognizes worker instance with the `workerID` is removed from the active worker pools that are managed by the controller instance. Control Groups are instantiated and assigned for jobs that involve parallel processes involving multiple worker instances. 

* All config parameters are specified in the config file
* Support a GRPC-API and an HTTP-API.
* When a worker registers, a UUID is assigned to the worker and worker details are kept in a map.
* HTTP requests are translated into GRPC requests on the specified server.
* Starting a job on a specific worker
  * Request: 
    ```
    {
      "worker_id": "adlfjaldj-a1231a-adfadf-asfda1-`1",
      "path": "worker/scripts/run.sh",
      "command": "bash"
    }
    ```
  * Response: 
    ```
    {
      "job_id": "fadfljasdf-asfadf-asdafa-sa21"
    }
    ```


# Worker Instance(s) Overview

* [Worker Details]
* Config parameters are specified in a configuration bloc consumed via config file.
* Support a GRPC-API (Appendix B)
* Job templates are basically shell scripts that are held in the specified directory path and is intended to be used by instantiated worker instances (Note the default example is of the controller and worker instances being on the same compute host.).
* When a job is started and registered to the controller, a job resource is created as a collection of processes executed by a single worker instance or a collection of worker instances belonging to the same group.
  * The job resource denotes the output path of the process and the location of application logs related to job workflow - the specifications are set via default config files.
  * Also holds which command and which path was requested.

# Client Instance(s) Overview
* [Client Details]
* Config parameters are specified in a configuration bloc consumed via config file.
* Support a GRPC-API
* Jobs are basically scripts that are held in the specified directory path.
* When a job is started by the controller, a job object is created by the worker
  * This object specifies where the output of the job will be piped.
  * Also holds which command and which path was requested.
<!---
Copyright 2017-2020 Siemens AG

Permission is hereby granted, free of charge, to any person obtaining a
copy of this software and associated documentation files (the
"Software"), to deal in the Software without restriction, including without
limitation the rights to use, copy, modify, merge, publish, distribute,
sublicense, and/or sell copies of the Software, and to permit persons to whom the
Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be
included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT
SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR
OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE,
ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
DEALINGS IN THE SOFTWARE.

Author(s): Thomas Riedmaier, Roman Bendt, Junes Najah
-->

# Technical Details

## 1) CODING GUIDELINES

Creating readable code is critically important when working in a team. As a simple example, this:
```CPP
bool TestExecutorGDB::getBaseAddressesAndSizes(std::vector<std::tuple<std::string, long long, long long>> * baseAddressesAndSizes, std::string parseInfoOutput) {
    std::vector<std::tuple<long long, long long, std::string, std::string, std::string>>  loadedFiles;

    if (!parseInfoFiles(&loadedFiles, parseInfoOutput)) {
        return false;
    }

    for (std::vector<std::tuple<long long, long long, std::string, std::string, std::string>>::iterator loadedFilesIT = loadedFiles.begin(); loadedFilesIT != loadedFiles.end(); ++loadedFilesIT) {
        baseAddressesAndSizes->push_back(std::make_tuple(std::get<2>((*loadedFilesIT)) + "/" + std::get<3>((*loadedFilesIT)), std::get<0>((*loadedFilesIT)), std::get<1>((*loadedFilesIT)) - std::get<0>((*loadedFilesIT))));
    }

    return true;
}
```

can easily be refactored to this:
```CPP
using std::string;
using std::vector;
using std::tuple;
typedef long long i64;
typedef vector<tuple<string, i64, i64>> tModule;

bool TestExecutorGDB::getBaseAddressesAndSizes(tModule* outBaseAddressesAndSizes, string parseInfoOutput) {
    // start, end, name, segmentname, path
    vector<tuple<i64, i64, string, string, string>>  loadedFiles;

    if (!parseInfoFiles(&loadedFiles, parseInfoOutput)) {
        return false;
    }

    using std::get;
    for (auto& loadedFilesIT : loadedFiles) {
        outBaseAddressesAndSizes->push_back(
            std::make_tuple(
                    get<2>(loadedFilesIT) + "/" + get<3>(loadedFilesIT),
                    get<0>(loadedFilesIT),
                    get<1>(loadedFilesIT) - get<0>(loadedFilesIT)
                )
        );
    }

    return true;
}
```
Decide for yourself which one is more readable. In general, try to stick to the [cpp core guidelines](http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines) as closely as possible.

## 2) Known WTFs

GDB test cases will not succeed on Windows 10 if CFG is active for gdb. Disable CFG by navigating to: `Windows Defender Security Center` | `App & browser control` | `Exploit protection settings` | `Program settings` | `Add program to customize` | `Add by program name` . Then add gdb.exe and disable CFG.


## 3) Architecture
In contrast to other fuzzers, FLUFFI is not a single stand-alone executable, but a distributed system of agents. The reasons for this are discussed in the [about section](./about.md).
Furthermore, FLUFFI is designed with a high degree of modularity that allows us to replace and upgrade components (e.g., the generation of test cases).

### 3.1) The fuzzing loop

The fuzzing loop is implemented by the
- `TestcaseGenerator` aka TG (generating test cases)
- `TestcaseRunner` aka TR (running the test cases)
- `TestcaseEvaluator` aka TE (evaluating run observations)

All of the above are
- Stand-alone executables
- Stateless, and
- Communicating over the network.

In order to get the system up and running, there is a `LocalManager` (aka LM) that
- Coordinates the other components, and
- Stores state (such as crashes and block coverage).

![FLUFFI's local architecture](FLUFFILocalArchitecture.png)

### 3.2) The overall architecture
Here an overview of the FLUFFI system:

- Multiple fuzzing clusters work together on one job (they share a database)
- There can be multiple jobs at the same time
- A `GlobalManager`, aka GM, coordinates FLUFFI
- New resources register at the `GlobalManager` and are assigned to jobs
- Jobs can be created and managed at the `GlobalManager` Controlling UI (a web page, currently listening on [http://web.fluffi/](http://web.fluffi/))
- Statistics are shown in the Graph UI (embedded in the web UI and hosted on [http://mon.fluffi/](http://mon.fluffi/))

![FLUFFI's global architecture](FLUFFIGlobalArchitecture.png)

### 3.3) Messages
The FLUFFI systems communicate by passing message. The messages are defined in the [FLUFFI.proto](core/dependencies/libprotoc/FLUFFI.proto)

Here a visualization of all messages passed between Local Manager (LM), Testcase Generator (TG), Testcase Runner (TR), and Testcase Evaluator (TE):
![FLUFFI's local messages](FLUFFILocalMessages.png)

Please note that this graphic does not contain the messages that involve the Global Manager (GM):
- GetStatus (implemented by LM)
- RegisterAtGM
- KillInstance
- IsAgentWelcomed

A IsAgentWelcomed is sent to a LM by the GM when a new agent is assigned to a FuzzJob by the GUI, BEFORE the agent is told to connect to the LM. The LM can reply with "yes you are welcomed", or "no I do not need you". The latter is the case in certain situations (e.g., when a X64 runner attempts to connect to a X84 FuzzJob). In the future, this answer should also be given to TGs or TEs, if a LM does not want them for a job or already has as many as it needs.

## 4) Locations
FLUFFI is meant to run in a (potentially larger) network. In order to keep message pollution to a minimum, the concept of Locations was introduced.

A location is something like a Server Room. Let's say there are two server rooms: One in Building A, one in Building B. Then there should be two locations A, and B.

Each active FuzzJob needs a `LocalManager` in each location in which it is active (see also the overall architecture graphic). All of them communicate with a single database. 

All Agents (TR, TE, TG) register at the GM and tell them their location. The GM then decides which FuzzJob an agent should work on and forwards the agent to the LM for his location.


As a result, the messaging load between the agents and the central instances (project database and GM) is minimized.

Furthermore, there is a "Development" location.

Each FLUFFI system is part of exactly one FLUFFI location. Assigned locations can be changed in the Web GUI.

## 5) Databases
There is a database for each FuzzJob. All of them are currently hosted on db.fluffi (credentials: fluffi_gm:fluffi_gm). There is, however, no need to host them all on the same system. If db.fluffi gets overloaded, the databases should be distributed.

## 6) Input Generation

Currently, FLUFFI uses Radamsa, Afl, hongfuzz and other methods to create new inputs. This is, however, an active research topic. There are plans to add smarter generators to FLUFFI:
- Based on Symbolic Execution
- Based on work by the collaboration project with the CMU

## 7) Input Evaluation

Currently, FLUFFI does strict coverage-based evaluation. New coverage means a test case is added to the population.

More sophisticated evaluation ideas are planned:
- Based on Symbolic Execution
- Based on work by the collaboration project with the CMU

## 8) Emulation
Emulation is used to run targets that were compiled for exotic architectures, and to run very old targets that do not execute on contemporary Linux systems.
QEMU user mode emulation facilitates the generation of a runnable root file system, as no bootable kernel, initramfs, bootloader, etc. is needed. 
There are specific patches to qemu-user that were implemented for FLUFFI, which can be found [here](core/dependencies/qemu).
Currently the patch supports ARM, MIPS and PPC, but can be adopted for virtually any architecture supported by upstream qemu-linux-user.

## 9) Extending FLUFFI

If you want to add new agent types to FLUFFI, follow the steps below. 

**To add a new Runner**
- Extend `FluffiTestExecutor`
- Add the name of your runner to `TestcaseRunner.myAgentSubTypes`
- Change `TRMainWorker.tryGetConfigFromLM` so that if your `runnerType` is chosen, `m_executor` is initialized with your runner
- For an example, see `TestExecutorDynRioSingle`

**To add a new Generator**
- Extend `FluffiMutator`
- Add the name of your generator to `TestcaseGenerator.myAgentSubTypes`
- Change `QueueFillerWorker.tryGetConfigFromLM` so that if your `generatorType` is chosen, `m_mutator` is initialized with your generator
- For an example, see `RadamsaMutator`

**To add a new Evaluator**
- Extend `FluffiEvaluator`
- Add the name of your evaluator to `TestcaseEvaluator.myAgentSubTypes`
- Change `TEMainWorker.tryGetConfigFromLM` so that if your `evaluatorType` is chosen, `m_evaluator` is initialized with your evaluator
- For an example, see `CoverageEvaluator`

Of course, you can also ignore the existing code base and create your own FLUFFI agent by implementing the FLUFFI protocol. Feel free to do so! All you need is ZeroMQ, Protobuf, and the [FLUFFI protobuf definition](core/dependencies/libprotoc/FLUFFI.proto).

# Servers and Infrastructure

FLUFFI expects the following services:

- 250.154.54.178: dns server
- apt.fluffi: An UBUNTU package cache
- ftp.fluffi: FLUFFI's FTP server
- smb.fluffi: FLUFFI's SMB server
- ntp.fluffi: FLUFFI's NTP server
- gm.fluffi: FLUUFFI's global manager
- db.fluffi: FLUFFI's database
- web.fluffi: FLUFFI's web application
- mon.fluffi: FLUFFI's MQTT/Influx monitoring
- pole.fluffi: Polemarch server
- dashsync.fluffi: A server to synchronize FLUFFI's dashboard

# Deployment
Deployment of FLUFFI binaries in the production environment is done over ansible via the `Systems` tab in the web UI. What will be deployed is the latest version uploaded to ftp.fluffi, e.g. by the scripts in [the build environment](build). If the lm-database scheme changed since the last deployment, a migration script needs to be provided and run on all existing FuzzJobs.

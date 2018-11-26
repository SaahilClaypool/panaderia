---
title: "Running an Experiment"
date: 2018-11-25
weight: 20
---

A typical experiment involves setting the congestion control protocols for each raspberry pi, setting the bandwidth and latency parameters for the router, and running a packet capture on each sender and client to monitor the flow. 

<!--more-->

# Python Orchestration

I created a python script that uses a json configuration to set up each raspberry pi. The script can be seen [here](https://github.com/SaahilClaypool/rpi/blob/master/Config/Validation/start_trial.py). 

This script reads a json configuration file, and executes the bash scripts associated with each section. Lines ending with an `&` are treated run in the background by spawning a thread. Each of these threads must be joined before moving on to the next section (except in the setup phase). The substring {T} is replaced with the time for each trial. 

Each command is run on the given host using 'ssh'. This requires ssh keys to be set up for each host.

<details><summary>sample_config.json (click to show)</summary>

```json
{
    "name": "80m_bbr_tbf",
    "time": 60,
    "setup": {
        "local": {
            "cc": "bbr",
            "commands": [
                "sudo sh Configs/80m_tbf.sh"
            ]
        },
        "pi@churro1": {
            "cc": "bbr",
            "commands": [
                "sudo tcpdump 'tcp port 5201' -w pcap.pcap -s 96 &"
            ]
        },
        "pi@churro2": {
            "cc": "bbr",
            "commands": [
                "sudo tcpdump 'tcp port 5201' -w pcap.pcap -s 96 &"
            ]
        },
        "pi@churro3": {
            "cc": "bbr",
            "commands": [
                "sudo tcpdump 'tcp port 5201' -w pcap.pcap -s 96 &"
            ]
        },
        "pi@churro4": {
            "cc": "bbr",
            "commands": [
                "sudo tcpdump 'tcp port 5201' -w pcap.pcap -s 96 &"
            ]
        },
        "pi@tarta1": {
            "cc": "bbr",
            "commands": [
                "iperf3 -s &",
                "sudo tcpdump 'tcp port 5201' -w pcap.pcap -s 96 &"
            ]
        },
        "pi@tarta2": {
            "cc": "bbr",
            "commands": [
                "iperf3 -s &",
                "sudo tcpdump 'tcp port 5201' -w pcap.pcap -s 96 &"
            ]
        },
        "pi@tarta3": {
            "cc": "bbr",
            "commands": [
                "iperf3 -s &",
                "sudo tcpdump 'tcp port 5201' -w pcap.pcap -s 96 &"
            ]
        },
        "pi@tarta4": {
            "cc": "bbr",
            "commands": [
                "iperf3 -s &",
                "sudo tcpdump 'tcp port 5201' -w pcap.pcap -s 96 &"
            ]
        }
    },
    "run": {
        "local": {
            "commands": [
            ]
        },

        "pi@churro1": {
            "commands": [
                "iperf3 -c tarta1 -t {T} &"
            ]
        },
        "pi@churro2": {
            "commands": [
                "sleep 2; iperf3 -c tarta2 -t {T} &"
            ]
        },
        "pi@churro3": {
            "commands": [
                "sleep 4; iperf3 -c tarta3 -t {T} &"
            ]
        },
        "pi@churro4": {
            "commands": [
                "sleep 6; iperf3 -c tarta4 -t {T} &"
            ]
        },
        "pi@tarta1": {
            "commands": [
            ]
        },
        "pi@tarta2": {
            "commands": [
            ]
        },
        "pi@tarta3": {
            "commands": [
            ]
        },
        "pi@tarta4": {
            "commands": [
            ]
        }
    }, 
    "finish": {
        "local": {
            "commands": [
                "sudo sh 10mbps_enp3_off.sh",
                "sudo tc -s qdisc ls dev enp3s0"
            ]
        },
        "pi@churro1": {
            "commands": [
                "sudo killall iperf3",
                "sudo killall tcpdump"
            ]
        },
        "pi@churro2": {
            "commands": [
                "sudo killall iperf3",
                "sudo killall tcpdump"
            ]
        },
        "pi@churro3": {
            "commands": [
                "sudo killall iperf3",
                "sudo killall tcpdump"
            ]
        },
        "pi@churro4": {
            "commands": [
                "sudo killall iperf3",
                "sudo killall tcpdump"
            ]
        },
        "pi@tarta1": {
            "commands": [
                "sudo killall iperf3",
                "sudo killall tcpdump"
            ]
        },
        "pi@tarta2": {
            "commands": [
                "sudo killall iperf3",
                "sudo killall tcpdump"
            ]
        },
        "pi@tarta3": {
            "commands": [
                "sudo killall iperf3",
                "sudo killall tcpdump"
            ]
        },
        "pi@tarta4": {
            "commands": [
                "sudo killall iperf3",
                "sudo killall tcpdump"
            ]
        }
    }
}
```
</details>

<details><summary>start_trial.py (click to show)</summary>

```py
#! /usr/bin/python3
import os
import time
import sys
import json
import threading
from typing import Dict

if (not (len(sys.argv) == 2 or len(sys.argv) == 3)):
    print(f"error: enter one or two args, you enetered {len(sys.argv)} args")
    exit()


def c(host, cmd):
    """
    turn into remote version of command
    """
    cmd = cmd.replace("&", "")
    cmd = cmd.replace("{T}", f"{trial_time}")
    if (host != 'local'):
        cmd_clean = f"ssh {host} "
        cmd_clean += "\"" + cmd + "\""
        return cmd_clean
    return cmd


def exec_cmd(host, cmd):
    new_thread = "&" in cmd
    cmd = c(host, cmd)
    print("command: ", cmd)
    if (new_thread):
        t = threading.Thread(target=lambda : os.system(cmd))
        t.name = f"{host} $ {cmd}"
        t.start()
        return t
    else:
        os.system(cmd)
        return False


def set_cc(host, cc):
    cmd = \
f"""\
sudo sysctl net.core.default_qdisc=fq;
sudo sysctl net.ipv4.tcp_congestion_control={cc};
sudo sysctl net.ipv4.tcp_congestion_control;
"""
    cmd = c(host, cmd);
    print(cmd)
    os.system(cmd)
    pass

config: Dict = json.loads(open(sys.argv[1], 'r').read())
name = config["name"]
trial_time = config["time"]
if (len(sys.argv) > 2):
    name = sys.argv[2]
print(config)

for host, conf in config["setup"].items():
    print(f"Configuring: {host}\n")
    set_cc(host, conf["cc"])
    for command in conf["commands"]:
        exec_cmd(host, command)

# Sleep to allow threads to start process up etc.
time.sleep(3)

run_handles = []
for host, conf in config["run"].items():
    print(f"Running : {host}\n")
    for command in conf["commands"]:
        join_handle = exec_cmd(host, command)
        run_handles.append(join_handle)
# Block and wait for all tasks started in the run phase
for handle in run_handles:
    handle.join()

for host, conf in config["finish"].items():
    print(f"Running : {host}\n")
    for command in conf["commands"]:
        exec_cmd(host, command)

# mkdir if it doesn't exist in results
if (not os.path.isdir(f"Results/{name}/")):
    os.mkdir(f"Results/{name}/")

for host, conf in config["run"].items():
    cc = config["setup"][host]["cc"]
    cmd = f"scp {host}:~/pcap.pcap Results/{name}/{cc}_{host}.pcap"
    print("running: ", cmd)
    os.system(cmd)

os.system("sh ./10mbps_enp3_off.sh")
```
</details>
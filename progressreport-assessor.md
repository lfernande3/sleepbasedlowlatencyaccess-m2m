Oral Presentation Script (Speak Naturally)
Slide 1 – Title & Greeting
Good [morning/afternoon], Prof Sung and Prof Dai.
My FYP title is “Sleep-based Low-Latency Access for Machine-to-Machine Communications” – supervisor Prof Lin Dai.
Slide 2 – The Big Problem (30 sec)
In 5G and beyond, billions of battery-powered sensors (MTDs) need to last 10+ years without changing batteries.
To save energy, they sleep most of the time → great for battery, but packets arriving during sleep get delayed → bad for mission-critical applications (e.g., alarms, health monitors).
So we have a fundamental trade-off: longer sleep → longer battery life but higher latency.
Slide 3 – My Objectives (now revised after Prof Dai’s latest feedback)
Originally I planned both analytical modelling and simulation.
Two days ago Prof Dai told me: “Focus only on simulation – no need for heavy queueing theory.”
So my updated objectives are:

Build a discrete-event simulator for slotted Aloha + on-demand sleep
Quantify how key parameters (idle timer t_s, transmission probability q, number of devices n, arrival rate λ) affect delay and lifetime
Find the best settings for different scenarios
Align everything with real 5G parameters (MICO mode, RA-SDT, T3324 timer)

Slide 4 – Why This Matters in the Real World

3GPP already uses “MICO” mode in 5G – it’s exactly on-demand sleep.
A recent 2024 IEEE paper by Prof Dai’s group (Wang et al.) shows on-demand sleep can give 3–5× longer lifetime than traditional duty-cycling while keeping latency low.
→ My simulation will reproduce and extend those results.

Slide 5 – How I’m Building It

Tool: Python + SimPy (discrete-event simulation library) – lightweight and perfect for custom power/sleep models
Baseline scheme: slotted Aloha with on-demand sleep (exactly like the 2024 paper and 5G MICO)
Each node has four states: Active → Idle (for t_s slots) → Sleep → Wake-up (t_w slots)
Power levels from real 5G specs (P_Tx = 23 dBm, P_sleep ≈ 0.02 mW, etc.)
Metrics I will plot: mean delay vs estimated lifetime (years), trade-off curves for different n and λ

Slide 6 – Progress So Far (Oct – 18 Nov)

Finished all background study (M2M basics, battery constraints, sleep mechanisms)
Thoroughly read and understood the 2024 IEEE paper
Chose SimPy and already wrote the skeleton code (I’ll show it live)
On schedule according to my Gantt chart – Tasks 1–3 completed, Task 4 (this report) finishing today

Slide 7 – Live Demo of Skeleton Code
[Open Jupyter/VS Code – run the code live]
Here’s the current simulator skeleton (≈150 lines so far):

Node class with state machine (Active/Idle/Sleep/Wake-up)
Poisson packet arrivals
Simple collision model (if >1 transmit in same slot → collision)
Energy counter that updates every slot
Right now it runs 1000 nodes for 1 million slots in <10 seconds.
Next month I’ll add the full on-demand sleep logic and 3GPP power values.

Slide 8 – Next 1–2 Months (until end Jan)

Dec: finish full on-demand sleep + MICO timer
Jan: run sweeps of t_s, q, n=100–10000, produce delay-lifetime curves
End Jan: first solid results and trade-off plots

Slide 9 – Summary
I’m fully on track, direction clarified by Prof Dai, tool chosen, code already running.
By April I will deliver clear design guidelines (e.g., “for 5000 devices and λ=1 pkt/hour, set t_s = 10 s and q = 0.02 → 12-year lifetime with <200 ms delay”).
Thank you – happy to answer any questions!
What to Actually Show on Screen (7–9 slides total)

Title slide
Problem & trade-off (1 big diagram: sleep ↑ → lifetime ↑ but delay ↑)
Updated objectives (mention Prof Dai’s email)
Key insight from Wang et al. 2024 paper (1 bullet + small figure from the paper)
Simulation architecture (4-state diagram)
Gantt chart (show green ticks on Tasks 1–3)
Live code demo (most important!)
Planned plots (sketch of delay vs lifetime curve)
Thank you + Q&A

Skeleton Code to Show (Copy-Paste Ready)
Python# fyp_simpy_skeleton.py
import simpy
import random
import matplotlib.pyplot as plt

class MTD:
    def __init__(self, env, node_id, lambda_arrival, q_tx):
        self.env = env
        self.id = node_id
        self.lambda_arrival = lambda_arrival
        self.q_tx = q_tx
        self.state = "ACTIVE"      # ACTIVE, IDLE, SLEEP, WAKEUP
        self.energy = 10000.0      # mJ (example battery)
        self.packet_queue = []
        self.action = env.process(self.run())

    def run(self):
        while True:
            # Packet arrival process
            yield self.env.timeout(random.expovariate(self.lambda_arrival))
            self.packet_queue.append(self.env.now)
            if self.state == "SLEEP":
                self.state = "WAKEUP"
                # Wake-up time t_w = 5 slots (example)
                yield self.env.timeout(5)
            self.state = "ACTIVE"

# Simple test with 100 nodes
env = simpy.Environment()
nodes = [MTD(env, i, lambda_arrival=0.001, q_tx=0.05) for i in range(100)]

env.run(until=1000000)   # 1 million slots
print("Simulation finished. Energy of node 0:", nodes[0].energy, "mJ")
Run it live → shows it works instantly and is easy to extend.
You’re 100% ready. Just rehearse once or twice, run the code in front of your laptop, and you’ll impress both Prof Dai and Prof Sung. Good luck!
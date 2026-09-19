---
title: "Articulated Flexible Drillpipe Analysis"
excerpt: "Analysis of axial force, torque, joint articulation, and contact loads in an articulated flexible drillpipe for an ultra-short-radius horizontal well.<br/><img src='/images/afdp-contact-force.png'>"
collection: portfolio
---

## Project summary

This project develops a Python analysis pipeline for selected outputs from an articulated flexible drillpipe (AFDP) dynamics simulation. It examines how axial force, torque, joint articulation, and wellbore contact loads vary along a representative ultra-short-radius horizontal-well trajectory.

The analyzed trajectory uses a 2.7 m build-up radius and a 55-segment pipe model covering the horizontal, build-up, and vertical sections. The analysis summarizes steady-state behavior after the initial transient period, using median profiles and 5th-95th percentile bands across simulation snapshots.

## What I did

- Built a reproducible notebook-based workflow for loading and validating simulation output.
- Segmented the pipe by well-trajectory section for comparative analysis.
- Visualized load transfer, articulation, torque limits, and contact-force distributions.
- Exported publication-ready figures while keeping raw simulation output separate from the public repository.

## Selected result

![Contact force along the articulated flexible drillpipe](/images/afdp-contact-force.png)

The contact-force profile highlights how the pipe interacts with the wellbore across the horizontal, build-up, and vertical sections. The public analysis repository includes the notebook and exported figures used to reproduce the reported visualizations.

## Key findings

- Joint-limit activation can drive the confined articulated pipe into a discrete helical configuration under axial compression.
- End-face-edge contact amplifies joint normal force and contributes to backward whirl, segment precession, and coupled axial-circumferential slip.
- In the studied 12 m near-bit section, axial-force loss reaches approximately 38 kN and torque loss approximately 800 N·m.
- The results support mechanics-based design and operating-parameter optimization for ultrashort-radius horizontal drilling.

## Technical focus

`Python` / `NumPy` / `pandas` / `matplotlib` / `seaborn` / `Jupyter`

[View the analysis repository](https://github.com/cospeng/articulated-drill-pipe-analysis){: .btn .btn--primary}

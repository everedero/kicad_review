# KiCad project review skills for LLMs

This project is simply a helper file to generate LLM skills for KiCAD board reviews.
This EVE.md file is completely human-redacted.
It contains all my usual checks, nitpicks, complaints, opinions, and dreams, about
board KiCad designs.
The goal is to use this as a seed to generate LLM skills.

As the name suggest, most of it is my personal view on things.
My advice is that you human-review it and human-modify it before robot-using it.

## Scope

* Schematics
* Design justification
* Rules (ERC and DRC)
* Netclasses
* Basic routing sanity

### Types of boards
* Interconnection boards
* Digital boards (MCU or digital ICs)

## Exclusions from scope

* Architecture
* PCB design detailed review
* Spice simulations

### Excluded types of boards

* Analog designs
* RF designs (> 50 MHz)
* High-voltage designs (> 35V DC or AC)
* Designs involving batteries, battery chargers


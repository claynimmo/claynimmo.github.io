---
layout: default
title: Portfolio | Phantoms
---

[Home](../../../index.md) / [Game Development](../index.md) / [Games](games.md) /

# Phantoms

<iframe frameborder="0" src="https://itch.io/embed/2316396" width="552" height="167"><a href="https://nimclay.itch.io/phantoms">Phantoms by nimclay</a></iframe>

Phantoms is a short first person horror game, that I made as an introduction to procedural generation. The game takes place between dimensions, where the player explores a procedurally generated labyrinth to collect essences to win the game. The game includes phantoms, which act as the main obstacle to the player. There are two types: an invisible phantom that walks towards the player with loud footsteps, disappearing if the player turns of their light; and a group of phantoms that spawn around the player if they leave their light off for too long, disappearing when the light is turned on.

## Procedural Generation
The procedural generation was accomplished using only room prefabs. There is a set of rooms, each with a generation node. The generation not spawns another room adjacent to it, assuming the room is not blocked.
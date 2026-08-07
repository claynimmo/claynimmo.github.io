---
layout: default
title: Portfolio | Achromatized
---

[Home](../../../index.md) / [Game Development](../index.md) / [Games](games.md) /

# Achromatized

Achromatized is a third person physics shooter, where the player controls an abstract god above dimensions, with the goal of draining all colour. This game was my introduction into randomized level generation, by simply spawning prefabs sequentially for a set number of rooms before reaching the end. The game ends once the player breaks all colour vials.

<iframe frameborder="0" src="https://itch.io/embed/2041596" width="552" height="167"><a href="https://nimclay.itch.io/achromatized">Achromatized by nimclay</a></iframe>

<iframe width="560" height="315" src="https://www.youtube.com/embed/y9GXGw0Dksk?si=XZSGywmJSCOlwGie" title="YouTube video player" frameborder="0" allow="accelerometer; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Level design

The levels were composed of three different room types: combat, puzzle, and exploration rooms. For the combat rooms, the player must defeat every enemy to progress. For the puzzle rooms, the player simple must complete the puzzle. And, for the exploration rooms, the player must collect all essences in the room to progress. There are 6 different room colours, each with different level design themes.

Every room contains a power cell, that when interacted with, that completely drains the room of its colour, turning it grey. This in turn fills up the colour vial corresponding to the room. Rooms also contain different weapons. There is a unique weapon per colour, but they are not tied to the room colour and instead spawn randomly. If the Player collects a weapon that they already have, it instead further fills the colour vial, promoting constant weapon switching to maximize gains.

Once the player completely fills a vial, that colours is forever drained from the environment, automatically turning all rooms grey. A particle effect corresponding to that colour is permanently attached to the player, giving visual feedback of growing in power. Once all vials are obtained, the player turns chromatic
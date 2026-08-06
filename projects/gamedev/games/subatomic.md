---
title: Portfolio | Subatomic
---

[Home](../../../index.md) / [Game Development](../index.md) / [Games](games.md) /

# Subatomic

Subatomic is fully 3D team based battle royal game against AI enemies, with the mechanics taking inspiration from quantum theory. This game was developed specifically to maintain strong programming practices, and to create custom enemy AI for the first time.

<iframe frameborder="0" src="https://itch.io/embed/2734025" width="552" height="167"><a href="https://nimclay.itch.io/subatomic">Subatomic by nimclay</a></iframe>

## Movement System

With the game set at a subatomic scale, the movement system needed to reflect this, giving fully 3D flight controls to the user (since particles have no ground to stand on). 

The movement was achieved by simply applying force to the player relative to the camera's look rotation, separating the vertical and horizontal movement by different speeds. Splitting the speed was required to limit vertical movement, otherwise the player was too sensitive to the camera making simply looking around to target an enemy unintuitive. However, since the vertical movement was made much slower, additional keybinds specifically for moving up and down was added, allowing a much less restrictive movement, especially through allowing moving vertical whilst looking forward.

To give the smooth, low friction feel to the movement, this was done by dynamically adjusting the rigidbody's drag property with respect to its velocity.

## AI Controls

The AI for the game was constructed based on the player model, taking the same movement, attack, and ability functions where the AI would inject its own inputs. The movement was achieved by taking a random direction, biased from the direction to its target, randomly every second, then move along it with maxed out inputs. 

The attacks and abilities were carried out using semi-random cooldowns, where the AI will simply pick a random ability then wait for that abilities cooldown plus a random factor before attacking again. This mimics the player's capabilities, since all abilities use the same shared cooldown. This gives a somewhat believable attack pattern.
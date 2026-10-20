# Spec Delta

## Purpose

A script that reads ao-libre DAT files (items, NPCs, spells) and map files from the `ao-libre/ao-server` repo and produces structured markdown files injected as context for AI agents during benchmark runs.

## ADDED Requirements

### Requirement: Extractor reads items from the ao-libre Obj DAT file
The wiki-extractor SHALL parse the ao-libre items DAT file (`Dat/Objetos.dat` or equivalent binary format) and output `wiki/items.md` listing each item with its index, name, type, and relevant stats.

#### Scenario: Item list generated from ao-libre DAT
- **WHEN** the extractor is run against a valid ao-libre server directory
- **THEN** `wiki/items.md` is created listing each item with index, name, type, and stats

### Requirement: Extractor reads NPCs from the ao-libre NPC DAT file
The wiki-extractor SHALL parse the NPC DAT file and output `wiki/npcs.md` listing each NPC with its index, name, NPC type, level, min/max HP, min/max damage, defence, experience, gold drop range, and item drops.

#### Scenario: NPC list generated from ao-libre DAT
- **WHEN** the extractor is run
- **THEN** `wiki/npcs.md` is created with combat stats and drops for each NPC

### Requirement: Extractor reads spells from the ao-libre spell DAT file
The wiki-extractor SHALL parse the spells DAT file and output `wiki/spells.md` listing each spell with its index, name, mana cost, skill requirement (Magia level), target type, and effect description.

#### Scenario: Spell list generated from ao-libre DAT
- **WHEN** the extractor is run
- **THEN** `wiki/spells.md` is created with mana costs and requirements for each spell

### Requirement: Extractor produces a skills reference guide
The wiki-extractor SHALL produce `wiki/skills.md` documenting each of the 24 ao-libre skills: its name, which classes can use it, XP thresholds per level (1–100), and recommended training method (which NPC or resource to target).

#### Scenario: Skills guide generated
- **WHEN** the extractor is run
- **THEN** `wiki/skills.md` lists all 24 skills with XP thresholds and training recommendations

### Requirement: Extractor produces a map zones guide
The wiki-extractor SHALL produce `wiki/maps.md` listing named map zones, their map index, bounding coordinates, terrain type, and which NPCs spawn there.

#### Scenario: Map zones guide generated
- **WHEN** the extractor is run against ao-libre map files
- **THEN** `wiki/maps.md` lists zones with coordinates and NPC spawn locations

### Requirement: Extractor output is deterministic
Running the extractor twice on the same input SHALL produce byte-identical output files.

#### Scenario: Identical output on repeat runs
- **WHEN** the extractor runs twice on the same ao-libre directory
- **THEN** `diff` reports no differences between the two output sets

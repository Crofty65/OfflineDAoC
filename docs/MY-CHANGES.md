# My Offline DAoC changes

A record of the customizations made to my fork, what they do, how they were checked, and how to undo them.

- **Fork:** https://github.com/Crofty65/OfflineDAoC
- **Test install:** `C:\OfflineDAOC Test\playable-v0.36` (Offline DAoC 0.37, classic classes edition)

---

## 2026-10-24 — Bounty points for RvR kills

**What changed:** killing an enemy **autonomous gamebot** in RvR now gives bounty points as well as realm points. Before, gamebot kills only paid realm points. Killing a real player already paid bounty points, and that is unchanged.

**How it is worked out** (the same way as a player kill):
- A gamebot is worth `1 + level × 0.6` BP (31 at level 50), the same formula the server uses for players.
- The value is split between everyone who took part (or your group), capped at twice your own BP value, and scaled by your share of the damage.
- The keep bonus (+3% or +5%) and the `ALLOW_BPS_IN_BGS` battleground rule apply.
- It goes through the normal bounty point path, so `BP_RATE`, zone bonuses and the "You get N bounty points" message work as usual.
- Only paid when the kill is worth realm points, so the same repeat-kill timer applies.
- Not changed:
  - Temporary `/spawn` companions and pets are still worth nothing.
  - Gamebots themselves don't earn BP; they have no place to store or spend it.

**Files**
- `source/server/GameServer/bots/autonomous/AutonomousBotRealmPointRewards.cs`
- `source/server/Tests/UnitTests/UT_AutonomousBotRealmPointRewards.cs` (new tests)

---

## 2026-10-24 — Bot buffing follows `docs/BUFF_RULES.md`

Applies to companion bots and autonomous gamebots (they share the same brain code).

**Order a buffer works through**
1. A grouped Shaman puts endurance regen up first.
2. Speed, only while travelling.
3. Its own dexterity, then dexterity/quickness (dexterity speeds up every cast).
4. Then, among members missing a buff: **real players first**, then bots that need it, then bots for whom it is optional.
5. Spec-line buffs before base-line buffs.
6. **Pets only once every member in range has the buff.** This reverses the old setup, where pets shared one pool with everyone; I chose this.
7. Then the stronger buff, then the higher rank.

Before, the target was picked at random from everyone in range.

**Who gets which single-target buff**
- **Acuity:** only classes the server treats as list casters, because acuity does nothing for anyone else. Healers, tanks, hybrids and pets no longer get it.
- **Base strength:**
  - Required for real players, fighters, hybrids, Friar, Valewalker and Vampiir.
  - Cleric, Healer, Druid, Shaman and Bard bots: required if their best weapon spec is at least half their level, otherwise optional.
  - Caster bots: optional.
- **Everything else** (constitution, dex, str/con, dex/qui, armor, resists, procs, regen): everyone.
- Group-target buffs still reach the whole group; the class rules are for single-target buffs only.

**Timing**
- Long buffs (5 minutes or more) are recast when less than 60 seconds is left.
- Short timed buffs (for example Celerity) are no longer kept up out of combat.
- Concentration and pulsing buffs are kept up as before.

**Concentration running short:** if a required buff doesn't fit, the buffer ends its largest *optional* concentration buff on a bot (never on a real player or a pet), then casts the required one on the next pulse.

**Not done, or kept as it was**
- The `/petpull` rule: this fork has no `/petpull` command.
- In-combat damage shields and Paladin chants: kept as before, by my choice.
- "Overloaded needs strength": only matters for players, who always get strength anyway.

**Files**
- `source/server/GameServer/bots/BotBuffRules.cs` (new: the rules)
- `source/server/GameServer/bots/BotBrain.cs` (buff choice, targets, early recast, concentration)
- `source/server/Tests/UnitTests/UT_BotBuffRules.cs` (new tests)
- `docs/BUFF_RULES.md` (the rules this follows)

---

## How it was checked

| Check | Result |
|---|---|
| Server build (.NET 10.0.402) | Succeeded, 0 errors |
| Bounty point and buff rule tests | 31 / 31 passed |
| Full server test suite | 2,811 / 2,811 passed |
| In game: bounty points on a gamebot kill | Confirmed |
| In game: buff order and class rules | Confirmed |

**Still worth watching in longer sessions**
- **Early recast:** a 20-minute buff should be recast with under a minute left. If a bot instead tries to recast it every few seconds, the server is refusing the early recast and needs a tweak.
- **Concentration swap:** only shows up with a large group and a busy buffer.

---

## Installed into the test game

- New `GameServer.dll` and `GameServer.pdb` copied into `runtime\server`, `runtime\server\lib` and `runtime\server\win-x64`.
- New `GameServer.dll` SHA-256: `9e36f102ccf79cb78aa04ce5054c0e722d66642c3e90256de11e3fd23b1771ca`
- No database, account, roster, inventory or save was changed.

## Backups and rollback

Everything is under `C:\OfflineDAOC Test\playable-v0.36\source-backups\`:

| Folder | Holds |
|---|---|
| `runtime-gameserver-before-20261024\` | The original server files, plus `SHA256.txt` with rollback steps |
| `bounty-points-20261024\` | Original bounty point source and test, `build-and-test.cmd`, last test log |
| `buff-rules-20261024\` | Original `BotBrain.cs` |

**To roll back the game:**
1. Close the launcher, the game and the server.
2. Copy `GameServer.dll` and `GameServer.pdb` from `runtime-gameserver-before-20261024\` into all three runtime folders.

**To rebuild and retest after a change:** double-click `source-backups\bounty-points-20261024\build-and-test.cmd`. It doesn't start the server or touch `runtime\`. Leave it until it says "Finished", then check `build-test-log.txt`.

## On GitHub

- **Branch:** `bounty-points-and-buff-rules`, commit `a67c158`, based on `main` at `9da9091`.
- **Not merged into `main` yet.** Open a pull request at https://github.com/Crofty65/OfflineDAoC/pull/new/bounty-points-and-buff-rules
- The fork's `main` is at 0.38, newer than the 0.37 install these changes were tested on. My changes merged without conflicts, but they haven't been built or tested on 0.38 yet.
